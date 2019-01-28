AWS Elastic Kubernetes Service (EKS) CI/CD Pipeline QuickStart  
===============================================

This solution shows you how to create an AWS EKS Cluster CI/CD Pipeline and deploy a simple web application with an external Load Balancer. This readme updates an article "CodeSuite - Continuous Deployment Reference Architecture for Kubernetes" referenced below and provides a more basic step by step process.  

Steps:  
  Create your Amazon EKS Service Role  
  Create your Amazon EKS Cluster VPC  
  Remove Your AWS EKS Cluster  


## Create your Amazon EKS Cluster
This assumes you already have a EKS Cluster up and running with kubectl configured.  Please see "AWS Elastic Kubernetes Service (EKS) Pipeline QuickStart" link below for setting up a cluster.    
```
https://github.com/kskalvar/aws-eks-pipeline-quickstart  
```

## Checkout aws-kube-codesuite from the aws-samples github

You will need to ssh into the AWS EC2 Instance you created above.  This is a step by step process.  

On the instance you have kubectl configured, checkout the codesuite repo from github.  
```
git clone https://github.com/aws-samples/aws-kube-codesuite
```

## Deploy the Application
Deploy the nginx default container application to the EKS Cluster
```
cd aws-kube-codesuite
kubectl apply -f ./kube-manifests/deploy-first.yml
```

### Get AWS External Load Balancer Address
Capture EXTERNAL-IP for use below
```
kubectl get svc codesuite-demo -o wide
```
Wait till you see "EXTERNAL-IP ```*.<region>.elb.amazon.com```" 

### Test from browser
Using your client-side browser enter the following URL
```
http://<EXTERNAL-IP>
```
## Use AWS CloudFormation to Create the CI/CD Pipeline

Please note, there is an issue with the CodeSuite Reference Architecture which allows it to only be built in the us-west-2 region currently.  The issue has been added to the github "Deploy Fails #12" but as of 2019-01-28 it has not been fixed.  I did identify a work-around which allows it to work in the us-east-1 region.

### AWS CloudFormation Console
Click on "Create Stack"  
Select "Specify an Amazon S3 template URL"  
```
https://s3.amazonaws.com/998551034662-aws-eks-codesuite/aws-refarch-codesuite-kubernetes.yaml
```
Click on "Next"  
```
Stack name: eks-codesuite-demo
The name of your EKS cluster: eks-cluster
```
Click on "Next"  
Click on "Next"  
Select Check Box "I acknowledge that AWS CloudFormation might create IAM resources with custom names"  
Select Check Box "I acknowledge that AWS CloudFormation might require the following capability: CAPABILITY_AUTO_EXPAND"
Click on "Create"


## Give the Lambda execution role permissions in Amazon EKS cluster
You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.


### Configure configmap/aws-auth

Add "rolearn" Lambda execution role
```
kubectl -n kube-system edit configmap/aws-auth
```
Replace "arn:aws:iam::*:role/eks-codesuite-demo-Pipeline-CodePipelineLambdaRole-*" below with "LambdaRoleArn" from output of CloudFormation script "eks-codesuite-demo-Pipeline-*"
```
apiVersion: v1
data:
  mapRoles: |
    - rolearn: arn:aws:iam::*:role/eks-nodegroup-NodeInstanceRole-*
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::*:role/eks-codesuite-demo-Pipeline-CodePipelineLambdaRole-*
      username: admin
      groups:
        - system:masters
```

## Test CI/CD Pipeline

### Install Credential Helper
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true

### AWS CodeCommit Console
Locate "eks-codesuite-demo" under "Repositories"
Click on "HTTPS" under "Clone URL" 

### Clone Repo and Update Code Base
```
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/eks-codesuite-demo
cp aws-kube-codesuite/sample-app/* eks-codesuite-demo/
```
### Modify Dockerfile AWS_DEFAULT_REGION 
```
cd eks-codesuite-demo
```
edit Dockerfile
```
ENV AWS_DEFAULT_REGION us-east-1
```
### Push Changes to CodeCommit Repo
```
git add . && git commit -m "test CodeSuite" && git push origin master
```

### AWS CodePipeline Console
Click on "eks-codesuite-demo-Pipeline-*-Pipeline-*" under Pipelines  
You should be able to watch your codePipeline execute

### Test Deployment from browser
Using your client-side browser enter the following URL
```
http://<EXTERNAL-IP>
```

### Delete Deployment, Service
Use kubectl to delete application
```
kubectl delete deployment,service codesuite-demo
```

## Remove CI/CD Pipeline
Before proceeding be sure you delete deployment,service codesuite-demo as instructed above.  Failure to do so will cause cloudformation
script to fail.

S3  codesuite-demo-pipeline-bvchgrte7e-artifactbucket-1ddhbbqms304g

### AWS ECR Console
Select "eks-c-repos-*"
Click on "Delete" Button

### AWS S3 Console
Select "eks-codesuite-demo-pipeline-*-artifactbucket-*"
Click on "Delete" Button

### AWS CloudFormation
Delete "eks-codesuite-demo" Stack  
Wait for "eks-codesuite-demo" to be deleted before proceeding


## References
CodeSuite - Continuous Deployment Reference Architecture for Kubernetes  
https://github.com/aws-samples/aws-kube-codesuite  
