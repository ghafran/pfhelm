# Deploy Application to Rafay
- Modify `values.yaml` change `image.repository` to your docker image
- Modify `templates/deployment.yml` change the ` volumes.ebs-claim` to your EBS volume containing the databases
- Zip helm chart 
```
cd ~/git
zip -r pf.zip ./pfhelm/*
```
- Upload the `pf.zip` file to Rafay to deploy the cluster
- Once deployment is complete, you should be able ssh into the running pod


# Infrastructure Provisioning Guide.
(This first section of the guide assumes some general knowledge of the Rafay UI).

Rafay itself must have permissions to be allowed to communicate with your cloud infrastructure. In this case, we are using AWS.
-	You’ll have to decide how Rafay can communicate on behalf of you to AWS APIs ([docs](https://docs.rafay.co/clusters/eks/credentials/)).
-	You’ll have to decide on what set of AWS API operations Rafay is allowed to perform on behalf of you ([docs](https://docs.rafay.co/clusters/eks/iam_policy/)).

Once you have this figured out you’ll setup something called a Cloud Credentials Provider in the Rafay UI/CLI. Whenever you need to spin up a cluster you’ll just select that cloud credentials provider (by name) from the dropdown.

For Cluster specs, we are using a g4dn.xlarge GPU node, with only 1 node necessary.

# EBS PVC Access
An Elastic Block Storage volume with access to the cluster must be created, I recommend using this guide here, which is also the official AWS guide: 
Requirements: aws cli, eksctl/aws cli
https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi-self-managed-add-on.html
Make sure you’ve set up aws configure so your terminal has access to aws assets.

Your EKS config is inside of ~/.kube/config by default. You can populate that with the config of your choosing, or you can run the follow command to set it using AWS automatically.
```
aws eks --region us-east-1 update-kubeconfig --name <your cluster name>
```

To deploy the Amazon EBS CSI driver to an Amazon EKS cluster
Create an IAM policy that allows the CSI driver's service account to make calls to AWS APIs on your behalf. You can view the policy document on GitHub

Download the IAM policy document from GitHub.
```
curl -o example-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json
```

Create the policy. You can change AmazonEKS_EBS_CSI_Driver_Policy to a different name. If you change it, make sure to change it in later steps.
COMMAND:

```
aws iam create-policy \
    --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
    --policy-document file://example-iam-policy.json
```

1. Create an IAM role and attach the IAM policy to it.
If you’ve already got an ebs-csi-controller-sa, then you have to give it a different name.
```
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster <my-cluster> \
    --attach-policy-arn arn:aws:iam::<accountID>:policy/AmazonEKS_EBS_CSI_Driver_Policy \
    --approve \
    --override-existing-serviceaccounts
```

Retrieve the ARN of the created role and note the returned value (from code below) for use in a later step!

```
aws cloudformation describe-stacks \
    --stack-name eksctl-<my-cluster>-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa \
    --query='Stacks[].Outputs[?OutputKey==`Role1`].OutputValue' \
    --output text
```

The returned value looks like:
```
arn:aws:iam::111122223333:role/eksctl-my-cluster-addon-iamserviceaccount-kube-sy-Role1-1J7XB63IN3L6T
```

Now we can deploy the driver as a helm file:

```
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver

helm repo update

helm upgrade -install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
    --namespace kube-system \
    --set image.repository=<accountID>.dkr.ecr.<region-code>.amazonaws.com/eks/aws-ebs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=ebs-csi-controller-sa
```

Example that we can use for the project with some modifications:
```
git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
```

Travel to the directory with an example storage class:
```
cd aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/
```

So now we need to use a new `claim.yaml`.
At the end, make sure that your claim.yaml is:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 5000Gi
```

And your storageclass.yaml is:
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

Apply the edits and launch the test app:
```
kubectl apply -f specs/
```

IF you make changes to storage or claim to change the sizes, run:
```
kubectl apply -f storageclass.yaml
kubectl apply -f claim.yaml
```

This will apply them. The PVC will need to be deleted (`kubectl delete pvc ebs-claim`) and reloaded though.

Verify the capacity of the PVC
```
kubectl describe pv pvc-37717cd6-d0dc-11e9-b17f-06fad4858a5a
```

You can also check the output of the EBS volume as well.
```
kubectl exec -it app -- cat /data/out.txt
```

Once you’re done with the test app, delete it with:
```
Kubectl delete pod app
```



