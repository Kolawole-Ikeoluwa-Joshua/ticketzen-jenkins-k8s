# Jenkins on Amazon Kubernetes

## Create a cluster

Follow my Introduction to Amazon EKS for beginners guide, to create a cluster <br/>
Repository [here](https://github.com/Kolawole-Ikeoluwa-Joshua/amazon-eks.git)

## Setup our Cloud Storage

```
# deploy EFS storage driver
kubectl apply -k "https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/deploy/kubernetes/overlays/stable"

# get VPC ID
aws eks describe-cluster --name getting-started-eks --query "cluster.resourcesVpcConfig.vpcId" --output text
# Get CIDR range
aws ec2 describe-vpcs --vpc-ids vpc-id --query "Vpcs[].CidrBlock" --output text

# security for our instances to access file storage
aws ec2 create-security-group --description efs-test-sg --group-name efs-sg --vpc-id VPC_ID
aws ec2 authorize-security-group-ingress --group-id sg-xxx  --protocol tcp --port 2049 --cidr VPC_CIDR

# create storage
aws efs create-file-system --creation-token eks-efs

# create mount point
aws efs create-mount-target --file-system-id FileSystemId --subnet-id SubnetID --security-group GroupID

# grab our volume handle to update our PV YAML
aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text
```

More details about EKS storage [here](https://aws.amazon.com/premiumsupport/knowledge-center/eks-persistent-storage/)

### Setup a namespace

```
kubectl create ns jenkins
```

### Setup our storage for Jenkins

```
kubectl get storageclass

# create volume
kubectl apply -f ./amazon-eks/jenkins.pv.yaml
kubectl get pv

# create volume claim
kubectl apply -n jenkins -f ./amazon-eks/jenkins.pvc.yaml
kubectl -n jenkins get pvc
```

### Deploy Jenkins

```
# rbac
kubectl apply -n jenkins -f ./jenkins.rbac.yaml

kubectl apply -n jenkins -f ./jenkins.deployment.yaml

kubectl -n jenkins get pods

```

### Expose a service for agents

```

kubectl apply -n jenkins -f ./jenkins.service.yaml

```

## Jenkins Initial Setup

```
kubectl -n jenkins exec -it <podname> cat /var/jenkins_home/secrets/initialAdminPassword
kubectl port-forward -n jenkins <podname> 8080

# setup user and recommended basic plugins
# let it continue while we move on!

```

## SSH to our node to get Docker user info

```
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa
ssh -i ~/.ssh/id_rsa ec2-user@ec2-13-239-41-67.ap-southeast-2.compute.amazonaws.com
id -u docker
cat /etc/group
# Get user ID for docker
1001
# Get group ID for docker
1001
```

## Docker Jenkins Agent

Docker file is [here](../dockerfiles/dockerfile) <br/>

```
# you can build it

cd ./jenkins/dockerfiles/
docker build . -t <docker-id>/jenkins-slave

```

## Continue Jenkins setup

Install Kubernetes Plugin <br/>
Configure Plugin: Values I used are [here](../readme.md) <br/>

Install Kubernetes Plugin <br/>

## Try a pipeline

```
pipeline {
    agent {
        kubernetes{
            label 'jenkins-slave'
        }

    }
    environment{
        DOCKER_USERNAME = credentials('DOCKER_USERNAME')
        DOCKER_PASSWORD = credentials('DOCKER_PASSWORD')
    }
    stages {
        stage('docker login') {
            steps{
                sh(script: """
                    docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
                """, returnStdout: true)
            }
        }

        stage('git clone') {
            steps{
                sh(script: """
                    git clone https://github.com/Kolawole-Ikeoluwa-Joshua/TicketZen.git
                """, returnStdout: true)
            }
        }

        stage('docker build') {
            steps{
                sh script: '''
                #!/bin/bash
                cd $WORKSPACE/TicketZen/auth
                docker build . --network host -t kolawolejoshua/auth:${BUILD_NUMBER}
                '''
            }
        }

        stage('docker push') {
            steps{
                sh(script: """
                    docker push kolawolejoshua/auth:${BUILD_NUMBER}
                """)
            }
        }

        stage('deploy') {
            steps{
                sh script: '''
                #!/bin/bash
                cd $WORKSPACE/TicketZen/
                #get kubectl for this demo
                curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
                chmod +x ./kubectl
                ./kubectl apply -f ./infra/k8s/auth-mongo-depl.yaml
                cat ./infra/k8s/auth-depl.yaml | sed s/1.0.0/${BUILD_NUMBER}/g | ./kubectl apply -f -
                '''
        }
    }
}
}
```
