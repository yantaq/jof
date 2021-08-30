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

    curl -O https://raw.githubusercontent.com/aws-samples/containers-blog-maelstrom/main/EFS-Jenkins/create-env.sh
    chmod +x create-env.sh
     ./create-env.sh

The script will:

* create an EFS file system, EFS mount points, an EFS access point, and a security group
* create an EFS-backed storage class, persistent volume, and persistent volume claim
* deploy the AWS Load Balancer Controller

### **Install Jenkins**

With the load balancer and persistent storage configured, we’re ready to install Jenkins.
Use Helm to install Jenkins in your EKS cluster:

    helm repo add jenkins https://charts.jenkins.io && helm repo update &>/dev/null

    helm install jenkins jenkins/jenkins \
      --set rbac.create=true \
      --set controller.servicePort=80 \
      --set controller.serviceType=ClusterIP \
      --set persistence.existingClaim=jenkins-efs-claim \
      --set controller.resources.requests.cpu=2000m \
      --set controller.resources.requests.memory=4096Mi \
      --set controller.serviceAnnotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb-ip

The Jenkins Helm chart creates a statefulset with 1 replica, and the pod will have 2 vCPUs and 4 GB memory. Jenkins will store its data and configuration at /var/jenkins_home path of the container, which is mapped to the EFS file system we created for Jenkins earlier in this post.

Get the load balancer’s DNS name:

    printf $(kubectl get service jenkins -o \
      jsonpath="{.status.loadBalancer.ingress[].hostname}");echo

Copy the load balancer’s DNS name and paste it in your browser. You should be taken to the Jenkins dashboard. Log in with username admin. Retrieve the admin user’s password from Kubernetes secrets:

    printf $(kubectl get secret jenkins -o \
      jsonpath="{.data.jenkins-admin-password}" \
      | base64 --decode);echo

## Build images

With Jenkins set up, let’s create a pipeline that includes a step to build container images using kaniko.

Create three Amazon Elastic Container Registry (ECR) repositories that will be used to store the container images for the Jenkins agent, kaniko executor, and sample application used in this demo:

    JOF_JENKINS_AGENT_REPOSITORY=$(aws ecr create-repository \
      --repository-name jenkins \
      --region $JOF_REGION \
      --query 'repository.repositoryUri' \
      --output text)

    JOF_KANIKO_REPOSITORY=$(aws ecr create-repository \
      --repository-name kaniko \
      --region $JOF_REGION \
      --query 'repository.repositoryUri' \
      --output text)

    JOF_MYSFITS_REPOSITORY=$(aws ecr create-repository \
      --repository-name mysfits \
      --region $JOF_REGION \
      --query 'repository.repositoryUri' \
      --output text)

Prepare the Jenkins agent container image:

    aws ecr get-login-password \
      --region $JOF_REGION | \
      docker login \
        --username AWS \
        --password-stdin $JOF_JENKINS_AGENT_REPOSITORY

    docker pull jenkins/inbound-agent:4.3-4-alpine
    docker tag docker.io/jenkins/inbound-agent:4.3-4-alpine $JOF_JENKINS_AGENT_REPOSITORY
    docker push $JOF_JENKINS_AGENT_REPOSITORY

Prepare the kaniko container image:

    mkdir kaniko
    cd kaniko

    cat > Dockerfile<<EOF
    FROM gcr.io/kaniko-project/executor:debug
    COPY ./config.json /kaniko/.docker/config.json
    EOF

    cat > config.json<<EOF
    { "credsStore": "ecr-login" }
    EOF

    docker build -t $JOF_KANIKO_REPOSITORY .
    docker push $JOF_KANIKO_REPOSITORY

Create an IAM role for Jenkins service account. The role lets Jenkins agent pods push and pull images to and from ECR:

    eksctl create iamserviceaccount \
        --attach-policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser \
        --cluster $JOF_EKS_CLUSTER \
        --name jenkins-sa-agent \
        --namespace default \
        --override-existing-serviceaccounts \
        --region $JOF_REGION \
        --approve

Create a new job (Pipeline) in the UI:

    cat > kaniko-demo-pipeline.json <<EOF
    pipeline {
        agent {
            kubernetes {
                label 'kaniko'
                yaml """
    apiVersion: v1
    kind: Pod
    metadata:
      name: kaniko
    spec:
      serviceAccountName: jenkins-sa-agent
      containers:
      - name: jnlp
        image: '$(echo $JOF_JENKINS_AGENT_REPOSITORY):latest'
        args: ['\\\$(JENKINS_SECRET)', '\\\$(JENKINS_NAME)']
      - name: kaniko
        image: $(echo $JOF_KANIKO_REPOSITORY):latest
        imagePullPolicy: Always
        command:
        - /busybox/cat
        tty: true
      restartPolicy: Never
    """
            }
        }
        stages {
            stage('Make Image') {
                environment {
                    DOCKERFILE  = "Dockerfile.v3"
                    GITREPO     = "git://github.com/ollypom/mysfits.git"
                    CONTEXT     = "./api"
                    REGISTRY    = '$(echo ${JOF_MYSFITS_REPOSITORY%/*})'
                    IMAGE       = 'mysfits'
                    TAG         = 'latest'
                }
                steps {
                    container(name: 'kaniko', shell: '/busybox/sh') {
                        sh '''#!/busybox/sh
                        /kaniko/executor \\
                        --context=\${GITREPO} \\
                        --context-sub-path=\${CONTEXT} \\
                        --dockerfile=\${DOCKERFILE} \\
                        --destination=\${REGISTRY}/\${IMAGE}:\${TAG}
                        '''
                    }
                }
            }
        }
    }
    EOF

Copy the contents of kaniko-demo-pipeline.json and paste it into the pipeline script section in Jenkins. It should look like this:

Click the Build Now button to trigger a build.


Once you trigger the build you’ll see that Jenkins has a created another pod. The pipeline uses the Kubernetes plugin for Jenkins to run dynamic Jenkins agents in Kubernetes. The kaniko executor container in this pod will clone to code from the sample code repository, build a container image using the Dockerfile in the project, and push the built image to ECR.

    kubectl get pods
    NAME                 READY   STATUS    RESTARTS   AGE
    jenkins-0            2/2     Running   0          4m
    kaniko-wb2pr-ncc61   0/2     Pending   0          2s

You can see the build by selecting the build in Jenkins and going to Console Output.

Once the build completes, return to AWS CLI and verify that the built container image has been pushed to the sample application’s ECR repository:

    aws ecr describe-images \
      --repository-name mysfits \
      --region $JOF_REGION

The output of the command above should show a new image in the ‘mysfits’ repository.

## Cleanup

    helm delete jenkins
    helm delete aws-load-balancer-controller --namespace kube-system
    aws efs delete-access-point --access-point-id $(aws efs describe-access-points --file-system-id $JOF_EFS_FS_ID --region $JOF_REGION --query 'AccessPoints[0].AccessPointId' --output text) --region $JOF_REGION
    for mount_target in $(aws efs describe-mount-targets --file-system-id $JOF_EFS_FS_ID --region $JOF_REGION --query 'MountTargets[].MountTargetId' --output text); do aws efs delete-mount-target --mount-target-id $mount_target --region $JOF_REGION; done
    sleep 5
    aws efs delete-file-system --file-system-id $JOF_EFS_FS_ID --region $JOF_REGION
    aws ec2 delete-security-group --group-id $JOF_EFS_SG_ID --region $JOF_REGION
    eksctl delete cluster $JOF_EKS_CLUSTER --region $JOF_REGION
    aws ecr delete-repository --repository-name jenkins --force --region $JOF_REGION
    aws ecr delete-repository --repository-name mysfits --force --region $JOF_REGION
    aws ecr delete-repository --repository-name kaniko --force --region $JOF_REGION
