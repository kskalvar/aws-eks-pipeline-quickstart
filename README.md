AWS Elastic Kubernetes Service (EKS) CI/CD Pipeline QuickStart  
===============================================

For anyone who's looked at AWS CodeStar you’ve probably come away a little underwhelmed.  Not that it doesn’t do what it's supposed to do it's just too simple.  Pipelines I would be interested in would be a little more sophisticated and integrate with a maybe a Kubernetes Cluster.  So when I ran into a AWS Code Pipeline Reference Architecture that did exactly that I definitely thought it was worth a look.

This solution shows you how to create an AWS EKS Cluster CI/CD Pipeline and deploy a simple web application with an external Load Balancer. This readme updates an article "CodeSuite - Continuous Deployment Reference Architecture for Kubernetes" referenced below and provides a more basic step by step process.  

Steps:  
  Create your Amazon EKS Cluster  
  Checkout aws-kube-codesuite from the aws-samples github  
  Deploy the Initial Application  
  Use AWS CloudFormation to Create the CI/CD Pipeline  
  Give Lambda Execution Role Permissions in Amazon EKS Cluster  
  Test CI/CD Pipeline  
  Remove CI/CD Pipeline  


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

## Deploy the Initial Application
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

### Test from Browser
Using your client-side browser enter the following URL
```
http://<EXTERNAL-IP>
```
## Use AWS CloudFormation to Create the CI/CD Pipeline

Please note, there is an issue with the CodeSuite Reference Architecture which allows it to only be built in the us-west-2 region currently.  The issue has been added to the github Issues "Deploy Fails #12" but as of 2019-01-28 it has not been fixed.  I did identify a work-around which allows it to work in the us-east-1 region.  That work-around is incorporated in this README.md.

### AWS CloudFormation Console
Click on "Create Stack"  
Select "Specify an Amazon S3 template URL"  
```
https://s3.amazonaws.com/998551034662-aws-eks-codesuite/aws-refarch-codesuite-kubernetes.yaml
```
Click on "Next"  
```
Stack name: eks-codesuite-demo
ClusterName: eks-cluster
```
Click on "Next"  
Click on "Next"  
Select Check Box "I acknowledge that AWS CloudFormation might create IAM resources with custom names"  
Select Check Box "I acknowledge that AWS CloudFormation might require the following capability: CAPABILITY_AUTO_EXPAND"  
Click on "Create"


## Give Lambda Execution Role Permissions in Amazon EKS Cluster
You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.


### Configure configmap/aws-auth

Add "rolearn" Lambda execution role using kubectl
```
kubectl -n kube-system edit configmap/aws-auth
```
Replace "arn:aws:iam::*:role/eks-codesuite-demo-Pipeline-CodePipelineLambdaRole-*" below with "LambdaRoleArn" from output of CloudFormation script "eks-codesuite-demo-Pipeline-*"  

Note: You need to add the second "rolearn" structure as there will be only one "rolearn" initially
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
You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.  

### Install Credential Helper
git config --global credential.helper '!aws codecommit credential-helper $@'  
git config --global credential.UseHttpPath true

### AWS CodeCommit Console
Locate "eks-codesuite-demo" under "Repositories"  
Click on "HTTPS" under "Clone URL" 

### Clone Repo and Update Code Base
Copy the sample-app to your new clone CodeCommit Repo
```
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/eks-codesuite-demo
cp aws-kube-codesuite/sample-app/* eks-codesuite-demo/
```
### Modify Dockerfile AWS_DEFAULT_REGION
Dockerfile still references "us-west-2" so change to "us-east-1"
```
cd eks-codesuite-demo
```
edit Dockerfile
```
ENV AWS_DEFAULT_REGION us-east-1
```
### Push Changes to CodeCommit Repo
Use git to push code changes to the repo
```
git add . && git commit -m "test CodeSuite" && git push origin master
```

### AWS CodePipeline Console
Click on ```eks-codesuite-demo-Pipeline-*-Pipeline-*``` under Pipelines  
You should be able to watch your codePipeline execute

### Test Deployment from Browser
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

### AWS ECR Console
Select ```eks-c-repos-*```  
Click on "Delete" Button

### AWS S3 Console
Select ```eks-codesuite-demo-pipeline-*-artifactbucket-*```  
Click on "Delete" Button

### AWS CloudFormation
Delete "eks-codesuite-demo" Stack  
Wait for "eks-codesuite-demo" to be deleted before proceeding


## References
AWS Elastic Kubernetes Service (EKS) QuickStart  
https://github.com/kskalvar/aws-eks-cluster-quickstart

CodeSuite - Continuous Deployment Reference Architecture for Kubernetes  
https://github.com/aws-samples/aws-kube-codesuite  
