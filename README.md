# Helm Chart

Helm chart describes how to run an instance of Ilyde in a running kubernetes cluster. 

## Setup a Kubernetes cluster with kops on aws

### Install kops on your OS 

To install kops on your OS, just follow the this link to the [official documentation](https://kops.sigs.k8s.io/getting_started/install/) of kuberbenetes operation. We'll setup a gosip cluster.

### Setup AWS IAM with required permissions

In order to build clusters within AWS we'll create a dedicated IAM user for kops. This user requires API credentials in order to use kops. Create the user, and credentials, using the AWS console.

The kops user will require the following IAM permissions to function properly:

```
AmazonEC2FullAccess
AmazonRoute53FullAccess  
AmazonS3FullAccess  
IAMFullAccess  
AmazonVPCFullAccess  
```

You can create the kOps IAM user from the command line using the following:
aws iam create-group --group-name kops

```console
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops

aws iam create-user --user-name kops

aws iam add-user-to-group --user-name kops --group-name kops

aws iam create-access-key --user-name kops
```

You should record the SecretAccessKey and AccessKeyID in the returned JSON output, and then use them below:

```console
# configure the aws client to use your new IAM user
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here

# Because "aws configure" doesn't export these vars for kops to use, we export them now
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```

### Setup Cluster State storage

The state storage is where kops stores all cluster configuration. We'll create an S3 bucket with versioning enalbe

```console
# create bucket
aws s3api create-bucket --bucket ilyde-k8s-store --region eu-central-1 --create-bucket-configuration LocationConstraint=eu-central-1
# enable versioning
aws s3api put-bucket-versioning --bucket ilyde-k8s-store --versioning-configuration Status=Enabled
```
Modify region and name with your name and region of your choice.

### Setup a sshpublickey kops will use to encrypt communication from your machine to cluster

you can use `kops create secret`. 

### Create your cluster

Here we'll setup a gossip-based cluster so make sure the name ends with k8s.local, in this case wano.k8s.local

```console
kops create cluster --name wano.k8s.local --state s3://ilyde-k8s-store --node-count 3 --node-size m4.large --zones eu-central-1c 
kops update cluster --state s3://ilyde-k8s-store wano.k8s.local --yes
kops validate cluster --wait 10m
```

### Add Kubernets Dashboard

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
# create a dashboard admin user to be able to access dashboard. Use file in
# kops/addons
kubectl apply -f ./kops/addons/dashboard-adminuser.yaml
# get access token to login into the dashboard
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | sls admin-user | ForEach-Object { $_ -Split '\s+' } | Select -First 1)
```
In another console:


```console
kubectl proxy
```
Your dashboard can be access via http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy

### Install Ilyde

This will install ilyde helm chart on the cluster in common namespace.

```console
helm install -n common ilyde ./ilyde
```

### Update cluster and add additional policies to nodes so they can access autoscaling service in AWS

```console
kops edit cluster --name wano.k8s.cluster --state s3://ilyde-k8s-store
```
Add the following policies to the nodes.

```console
kind: Cluster
...
spec:
  additionalPolicies:
    node: |
      [
        {
          "Effect": "Allow",
          "Action": [
            "autoscaling:DescribeAutoScalingGroups",
            "autoscaling:DescribeAutoScalingInstances",
            "autoscaling:DescribeLaunchConfigurations",
            "autoscaling:SetDesiredCapacity",
            "autoscaling:TerminateInstanceInAutoScalingGroup",
            "autoscaling:DescribeTags"
          ],
          "Resource": "*"
        }
      ]
...
```

### Apply update in cloud


```console
kops update cluster --name wano.k8s.local --state s3://ilyde-k8s-store --yes
kops rolling-update cluster --name wano.k8s.local --state s3://ilyde-k8s-store --yes
```

### Install Autoscaler

```console
 kubectl apply -f ./kops/addons/cluster-autoscaler-autodiscover.yaml
```