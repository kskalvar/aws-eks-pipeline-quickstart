AWS Elastic Kubernetes Service (EKS) CI/CD Pipeline QuickStart  
===============================================

This solution shows you how to create an AWS EKS Cluster CI/CD Pipeline and deploy a simple web application with an external Load Balancer. This readme updates an article "CodeSuite - Continuous Deployment Reference Architecture for Kubernetes" referenced below and provides a more basic step by step process.  

Steps:  
  Create your Amazon EKS Cluster  
  Checkout aws-kube-codesuite from the aws-samples github  
  Deploy the Initial Application  
  Create S3 Deployment Bucket (AWS CloudFormation Templates + Code for Pipeline)  
  Use AWS CloudFormation to Create the CI/CD Pipeline  
  Give Lambda Execution Role Permissions in Amazon EKS Cluster  
  Test CI/CD Pipeline  
  Remove CI/CD Pipeline  
  

## Create your Amazon EKS Cluster
This assumes you already have a EKS Cluster up and running with kubectl configured.  Please see "AWS Elastic Kubernetes Service (EKS) Pipeline QuickStart" link below for setting up a cluster.    
```
https://github.com/kskalvar/aws-eks-pipeline-quickstart  
```

## Checkout aws-eks-pipeline-quickstart from github
You will need to ssh into the AWS EC2 Instance you created above.  This is a step by step process.  

On the instance you have kubectl configured, checkout the codesuite repo from github.  
```
git clone https://github.com/kskalvar/aws-eks-pipeline-quickstart
```

## Deploy the Initial Application
Deploy the application to the EKS Cluster
```
cd aws-eks-pipeline-quickstart
kubectl apply -f ./kube-manifests/deploy-first.yml
```

### Get AWS External Load Balancer Address
Capture EXTERNAL-IP for use below
```
kubectl get svc codesuite-demo -o wide
```
Wait till you see an "EXTERNAL-IP "*.<region>.elb.amazon.com" 

### Test from Browser
Using your client-side browser enter the following URL
```
http://<EXTERNAL-IP>
```

## Create S3 Deployment Bucket
Copy the deployment artifacts from the project deployment directory to an S3 Bucket  

### AWS S3 Console
Create a new bucket with a combination of your "AWS Account Id" and "aws-eks-codesuite".  
Example: "998551034662-aws-eks-codesuite".  This should provide you with a unique bucket name since S3 is a global service  

Click on "Create bucket"
```
Bucket name: <Your AWS Account Id>-aws-eks-codesuite
Region: US East(N.Virginia)
```
Next  
Next  
Next  
Create bucket  

Once your bucket is created upload all the files located in the "aws-eks-pipeline-quickstart/deployment" directory to the bucket.

## Use AWS CloudFormation to Create the CI/CD Pipeline
Create the CI/CD Pipeline using the CloudFormation
```
Note:  There is an issue with the CodeSuite Reference Architecture reference which allows
       it to only be built in the us-west-2 region currently.  The issue has been added to the
       github Issues "Deploy Fails #12" but as of 2019-01-28 it has not been fixed.  I did identify
       a work-around which allows it to deployed in all regions.  That work-around is incorporated in
       this README.md.
```

### AWS CloudFormation Console
Click on "Create Stack"  
Select "Specify an Amazon S3 template URL"  
```
https://s3.amazonaws.com/<Your AWS Account Id>-aws-eks-codesuite/aws-refarch-codesuite-kubernetes.yaml
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


Wait for Status CREATE_COMPLETE before proceeding

## Give Lambda Execution Role in Amazon EKS Cluster
You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.
```
NOTE:  There is a script in /home/ec2-user/aws-eks-pipeline-quickstart/scripts called "configure-aws-auth-pipeline".  
       You may run this script to automate adding a rolearn in .kube/aws-auth-cm.yaml for the pipeline.
       This script uses the naming convention I specified in this HOW-TO.  So if you didn't use the naming convention
       it won't work.  If you do use the script then all you need to do is continue to the "Test CI/CD Pipeline" step.

To Run the Script:

cp ~/aws-eks-pipeline-quickstart/scripts/configure-aws-auth-pipeline .
chmod u+x configure-aws-auth-pipeline
./configure-aws-auth-pipeline
```

### Configure configmap/aws-auth
Add "rolearn" Lambda execution role using kubectl
```
kubectl -n kube-system edit configmap/aws-auth
```
Replace "arn:aws:iam::*:role/eks-codesuite-demo-Pipeline-CodePipelineLambdaRole-*" below with "LambdaRoleArn" from output of CloudFormation script "eks-codesuite-demo-Pipeline-*"  

Note: You need to add a second "rolearn" structure as there will be only one "rolearn" initially.  Be sure to add the second one only,  
      as they appear similar.
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
### Install Credential Helper
git config --global credential.helper '!aws codecommit credential-helper $@'  
git config --global credential.UseHttpPath true


## Test CI/CD Pipeline
You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.  

### AWS CodeCommit Console
Locate "eks-codesuite-demo" under "Repositories"  
Click on "HTTPS" under "Clone URL" 

### Clone Repo and Update Code Base
Copy the sample-app to your new clone CodeCommit Repo
```
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/eks-codesuite-demo
cp aws-eks-pipeline-quickstart/sample-app/* eks-codesuite-demo/
```

### Push Changes to CodeCommit Repo
Use git to push code changes to the repo
```
cd eks-codesuite-demo
git add . && git commit -m "test CodeSuite" && git push origin master
```

### AWS CodePipeline Console
Click on "eks-codesuite-demo-Pipeline-*-Pipeline-*" under Pipelines  
You should be able to watch your codePipeline execute.  Please note you might see a "Failed" Source execution initially. Ignore it.

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
Select "eks-c-repos-*"  
Click on "Delete" Button

### AWS S3 Console
Select "eks-codesuite-demo-pipeline-*-artifactbucket-*"  
Click on "Delete" Button

### AWS CloudFormation Console
Delete "eks-codesuite-demo" Stack  
Wait for "eks-codesuite-demo" to be deleted before proceeding

## References
AWS Elastic Kubernetes Service (EKS) QuickStart  
https://github.com/kskalvar/aws-eks-cluster-quickstart

CodeSuite - Continuous Deployment Reference Architecture for Kubernetes  
https://github.com/aws-samples/aws-kube-codesuite  
