# jenkins on AWS EKS Fargate with EFS Persisten Storage

## Referenced from:
How to build container images with Amazon EKS on Fargate 
[Amazon EKS on Fargate](https://aws.amazon.com/blogs/containers/how-to-build-container-images-with-amazon-eks-on-fargate/).

You will need the following to complete the tutorial:

* AWS CLI version 2
* eksctl
* kubectl
* Helm

Let’s start by setting a few environment variables:

    export JOF_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
    export JOF_REGION="ap-southeast-1"
    export JOF_EKS_CLUSTER=jenkins-on-fargate

## Create an EKS cluster
We’ll use eksctl to create an EKS cluster backed by Fargate. This cluster will have no EC2 instances. Create a cluster:

    eksctl create cluster \
      --name $JOF_EKS_CLUSTER \
      --region $JOF_REGION \
      --version 1.18 \
      --fargate

With the `–-fargate` option, `eksctl` creates a pod execution role and Fargate profile and patches the coredns deployment so that it can run on Fargate.

Prepare the environment
The container image that we’ll use to run Jenkins stores data under /var/jenkins_home path of the container. We’ll use Amazon EFS to create a file system that we can mount in the Jenkins pod as a persistent volume. This persistent volume will prevent data loss if the Jenkins pod terminates or restarts.

Download the script to prepare the environment:
