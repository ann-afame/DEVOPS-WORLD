### Project: Deploying Microservices to AWS EKS with CI/CD Using Terraform and AWS Native Tools

#### Project Overview
This project simulates a real-world scenario for a company aiming to set up a CI/CD pipeline for deploying microservices to an Amazon Elastic Kubernetes Service (EKS) cluster using AWS native tools. The tools involved include AWS CodePipeline, AWS CodeBuild, and Terraform for infrastructure as code (IaC).

#### Prerequisites
1. **AWS Account**: Ensure you have an active AWS account.
2. **Terraform Installed**: Install Terraform on your local machine.
3. **AWS CLI Installed**: Install the AWS CLI and configure it with your AWS credentials.
   ```bash
   aws configure
   ```
4. **kubectl Installed**: Install `kubectl` for interacting with your Kubernetes cluster.

### Step 1: Setting Up the Infrastructure with Terraform
1. **Create a Directory for Terraform Files**:
   ```bash
   mkdir company-eks-cicd
   cd company-eks-cicd
   ```

2. **Create `main.tf`**:
   ```hcl
   provider "aws" {
     region = "us-west-2"
   }

   module "vpc" {
     source  = "terraform-aws-modules/vpc/aws"
     version = "3.14.0"

     name = "company-vpc"
     cidr = "10.0.0.0/16"

     azs             = ["us-west-2a", "us-west-2b", "us-west-2c"]
     public_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
     private_subnets = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

     tags = {
       Terraform = "true"
       Environment = "dev"
     }
   }

   module "eks" {
     source          = "terraform-aws-modules/eks/aws"
     cluster_name    = "company-cluster"
     cluster_version = "1.21"
     subnets         = module.vpc.private_subnets
     vpc_id          = module.vpc.vpc_id

     node_groups = {
       eks_nodes = {
         desired_capacity = 3
         max_capacity     = 5
         min_capacity     = 1

         instance_type = "t3.medium"
       }
     }

     tags = {
       Terraform = "true"
       Environment = "dev"
     }
   }
   ```

3. **Create `outputs.tf`**:
   ```hcl
   output "cluster_id" {
     description = "The ID of the EKS cluster."
     value       = module.eks.cluster_id
   }

   output "cluster_endpoint" {
     description = "The endpoint of the EKS cluster."
     value       = module.eks.cluster_endpoint
   }

   output "cluster_security_group_id" {
     description = "The security group IDs of the EKS cluster."
     value       = module.eks.cluster_security_group_id
   }
   ```

4. **Create `variables.tf`**:
   ```hcl
   variable "region" {
     description = "The AWS region to deploy to."
     type        = string
     default     = "us-west-2"
   }
   ```

### Step 2: Initialize and Apply Terraform
1. **Initialize Terraform**:
   ```bash
   terraform init
   ```

2. **Apply Terraform Configuration**:
   ```bash
   terraform apply -auto-approve
   ```

### Step 3: Configure `kubectl` for EKS
1. **Update kubeconfig**:
   ```bash
   aws eks --region us-west-2 update-kubeconfig --name company-cluster
   ```

2. **Verify Nodes**:
   ```bash
   kubectl get nodes
   ```

### Step 4: Set Up AWS CodePipeline for CI/CD
1. **Create an S3 Bucket** for storing artifacts:
   ```bash
   aws s3 mb s3://company-codepipeline-bucket
   ```

2. **Create a CodeCommit Repository**:
   ```bash
   aws codecommit create-repository --repository-name company-repo
   ```

3. **Create a Build Specification File (`buildspec.yml`)**:
   ```yaml
   version: 0.2

   phases:
     install:
       runtime-versions:
         docker: 18
       commands:
         - echo Installing dependencies...
     pre_build:
       commands:
         - echo Logging in to Amazon ECR...
         - $(aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-west-2.amazonaws.com)
         - REPOSITORY_URI=<your-account-id>.dkr.ecr.us-west-2.amazonaws.com/company-repo
         - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
     build:
       commands:
         - echo Build started on `date`
         - echo Building the Docker image...
         - docker build -t $REPOSITORY_URI:latest .
         - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
     post_build:
       commands:
         - echo Build completed on `date`
         - echo Pushing the Docker images...
         - docker push $REPOSITORY_URI:latest
         - docker push $REPOSITORY_URI:$IMAGE_TAG
         - echo Writing image definitions file...
         - printf '[{"name":"company-app","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
   artifacts:
     files: imagedefinitions.json
   ```

4. **Create a CodeBuild Project**:
   ```bash
   aws codebuild create-project --name company-codebuild-project \
     --source type=CODECOMMIT,location=https://git-codecommit.us-west-2.amazonaws.com/v1/repos/company-repo \
     --artifacts type=CODEPIPELINE \
     --environment type=LINUX_CONTAINER,image=aws/codebuild/standard:4.0,computeType=BUILD_GENERAL1_SMALL \
     --service-role arn:aws:iam::[YOUR_ACCOUNT_ID]:role/service-role/codebuild-service-role
   ```

5. **Create a CodePipeline**:
   ```bash
   aws codepipeline create-pipeline --pipeline file://pipeline.json
   ```

   Example `pipeline.json`:
   ```json
   {
     "pipeline": {
       "name": "CompanyPipeline",
       "roleArn": "arn:aws:iam::[YOUR_ACCOUNT_ID]:role/AWSCodePipelineServiceRole",
       "artifactStore": {
         "type": "S3",
         "location": "company-codepipeline-bucket"
       },
       "stages": [
         {
           "name": "Source",
           "actions": [
             {
               "name": "Source",
               "actionTypeId": {
                 "category": "Source",
                 "owner": "AWS",
                 "provider": "CodeCommit",
                 "version": "1"
               },
               "outputArtifacts": [
                 {
                   "name": "source_output"
                 }
               ],
               "configuration": {
                 "RepositoryName": "company-repo",
                 "BranchName": "main"
               }
             }
           ]
         },
         {
           "name": "Build",
           "actions": [
             {
               "name": "Build",
               "actionTypeId": {
                 "category": "Build",
                 "owner": "AWS",
                 "provider": "CodeBuild",
                 "version": "1"
               },
               "inputArtifacts": [
                 {
                   "name": "source_output"
                 }
               ],
               "outputArtifacts": [
                 {
                   "name": "build_output"
                 }
               ],
               "configuration": {
                 "ProjectName": "company-codebuild-project"
               }
             }
           ]
         },
         {
           "name": "Deploy",
           "actions": [
             {
               "name": "Deploy",
               "actionTypeId": {
                 "category": "Deploy",
                 "owner": "AWS",
                 "provider": "ECS",
                 "version": "1"
               },
               "inputArtifacts": [
                 {
                   "name": "build_output"
                 }
               ],
               "configuration": {
                 "ClusterName": "company-cluster",
                 "ServiceName": "company-service",
                 "FileName": "imagedefinitions.json"
               }
             }
           ]
         }
       ]
     }
   }
   ```

### Step 5: Verify the CI/CD Pipeline
1. **Commit Code to CodeCommit**:
   Push your application code to the CodeCommit repository.

2. **Monitor the Pipeline**:
   Check the AWS CodePipeline console to ensure the pipeline runs through all stages: Source, Build, and Deploy.

### Step 6: Create a Kubernetes Deployment
1. **Create `deployment.yaml`**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: company-deployment
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: company-app
     template:
       metadata:
         labels:
           app: company-app
       spec:
         containers:
         - name: company-app
           image: <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/company-repo:latest
           ports:
           - containerPort: 80
   ``
