# AWS DevOps CI/CD Pipeline using Docker, ECR, ECS Fargate

## Project Overview

This project demonstrates a **real-time AWS DevOps CI/CD architecture** where an application is containerized using Docker, stored securely in Amazon Elastic Container Registry (ECR), and deployed on Amazon Elastic Container Service (ECS) using Fargate.

The entire deployment process is **fully automated** using AWS CI/CD services, ensuring fast, reliable, and scalable application delivery.

---


## Architecture Overview

**CI/CD Flow:**

1. Developer pushes code to GitHub
2. AWS CodePipeline triggers automatically
3. AWS CodeBuild builds Docker image
4. Docker image is pushed to Amazon ECR
5. AWS CodeDeploy deploys image to ECS Fargate
6. Application is exposed via Application Load Balancer (HTTPS)
7. Monitoring and logging via CloudWatch


![AWS DevOps CICD Pipeline using Docker, ECR, ECS Fargate](https://github.com/user-attachments/assets/2e754482-b903-49ee-8b7d-fdc18aac0fac)


---

## ⚙️ CI/CD Pipeline Explanation

### Source Stage (GitHub)
- Developer pushes code to GitHub repository
- CodePipeline detects the change automatically

### Build Stage (CodeBuild)
- CodeBuild reads `buildspec.yml`
- Builds Docker image
- Tags the image with build number
- Pushes image to Amazon ECR

### Deploy Stage (CodeDeploy + ECS)
- CodeDeploy updates ECS task definition
- ECS Fargate pulls the latest image from ECR
- New containers are launched automatically
- Old containers are terminated safely (zero downtime)

---

## Docker Workflow

- Application is packaged using Docker
- Dockerfile defines:
  - Base image
  - Dependencies
  - Application startup command
- Docker image ensures consistency across environments

---

## Security

- HTTPS enabled using **AWS Certificate Manager (ACM)**
- IAM roles with least-privilege access
- Private ECR repository
- ECS tasks run in private subnets

---

## Monitoring & Logging

- Application logs streamed to **CloudWatch Logs**
- Metrics such as CPU & memory usage monitored
- ECS service health checks via ALB

---

## Key Features

✔ Fully automated CI/CD pipeline  
✔ Serverless container deployment (Fargate)  
✔ Scalable and highly available architecture  
✔ Secure HTTPS access  
✔ Zero-downtime deployments  
✔ Real-world DevOps implementation  

---

## Use Cases

- Production-grade CI/CD pipeline
- DevOps interview project
- Containerized application deployment
- AWS DevOps hands-on learning

---

## Author

**Gaju Sawase**  
AWS DevOps Engineer (Aspirant)



## step-by-step 


Launch Amazon Linux 2023 EC2 instance 

yum install docker -y

service docker start

service docker status

docker images 

docker ps -a


===================================================
As we need to have project in Linux machine , we need to clone the project from GitHub repository
=

install git 
--> yum install git -y
--> git clone https://github.com/gaju0203/AWS-DEVOPS-ci-cd-project-using-ECS-Fargate.git
--> cd website


Dockerfile = is used to create images automatically
===================================================

In website directory already Dockerfile exists 

cat dockerfile

FROM nginx:latest

COPY . /usr/share/nginx/html/

EXPOSE 80


docker images --> check if we have any Images

docker build -t website:latest . --> with tag called latest, and image  name website

docker images

docker run -d -p 80:80 website --> [Dont create] this will create container with website image

docker ps -a

copy the instance ip and put in browser to access website

======================
Pushing Image to ECR
======================

Create a Private Respository in ECS - Name: website

Create a Role with AmazonEC2ContainerRegistryFullAccess access and attach to EC2

IN ECR respository, click on view push commands

aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 289857919920.dkr.ecr.ap-south-1.amazonaws.com

docker build -t website .

docker tag website:latest 289857919920.dkr.ecr.ap-south-1.amazonaws.com/website:latest

docker push 289857919920.dkr.ecr.ap-south-1.amazonaws.com/website:latest

See the image in ECR

=======================================

Create a ECS Cluster with Fargate and enable Monitoring(optional)

Cluster name = websitecluster

Fargate: managed by AWS
EC2: AWS will launch EC2 and setup docker inside but we can use it if required

Create Task Definition --> before Run task, we need to create task definition 

Task definition family = website-task-df
 --> Launch type - Fargate
 --> Role - create if required, if app wants to connect to other services
 --> cpu .25vcpu, .5gb
Container 1
 --> Name: website, image: URI of image from ECR - 289857919920.dkr.ecr.ap-south-1.amazonaws.com/website:latest
 --> Enable Logs and rest keep defaults
 --> Create

========== Dont Create TASK NOW =========== If required in real time create task first for internal apps
Create Run Task

 --> Click on CLuster --> TASKS --> Run New Task
 --> Select Launch Type and Fargate
 --> Application Type --> TASK
 --> Family --> select
 --> Desired task --> 2
 --> Create

Click on task --> Public IP , put in browser

Now how will you give this website to customer, you cannot give task IP address to access
For this reason, let us create this as Service

===================================================

Before creating a Service, create a TargetGroup --> IP address and load Balancer , health check /

or , Service will help you to create TG , ELB 

Create Service in ECS
 --> Launch type, 
 --> Application type : Service
 --> Family : nginx-def
 --> Service name = websiteservice
 ->  Desired Task : 4
 --> Networking: remove 1c, select default-sg, publicip on
 --> Expand Load Balancer , ALB, user existing ALB, TG
 --> Expand Service Auto-Scaling, Min = 2, Max = 6, Policy name= sample, metric = CPU, value = 70%
 --> Create


====================================
Now Setup CICD for ECS through GitHub
=========================================


Create a CodeBuild Project

 --> select CodeBuild and create Build Project
 --> projectname: website
 --> Project Type: default
 --> SOurce 1 Primary: SOurce Provider: GitHub 
     Public Repository: 
     Repository URL : https://github.com/ReyazShaik/website
     Leave rest defaults
     Service role: New service role, or select if you have one
     Use BuildSpec: 
     Buildspec name : buildspec.yml  [This file is there in repo, update values in buildspec.yml]
     No artifacts
     unchecklogs
 
 --> Create Build Project

====> Search for ROLE in IAM codebuildrolenew -- Add AmazonEC2ContainerRegistryFullAccess and AmazonEC2ContainerRegistryPowerUser , SecretsManagerReadWrite permissions  or Admin permissions

--> NOw start the Build --> check phase details [This will get error in login, as role doesn't have permission to ecr]

--> And again start build --> now this build will create a new image in ECR --> check in ECR

===============
Now create a CodePipeline --> you cannot create CodeDeploy directly no option, first create pipeline, this will create deploy



Create a new Pipeline 
--> Name: ECS-CICD
--> Queued 
--> New role 
 --> Source - GitHub v2 , Connection = Connect to GitHub , connectioname:aws-codepipeline-connection --> connect to GitHub
     install new app --> select repository --> website --> install --> connect
 --> Repository name - website
 --> default Branch - master - rest keep defaults
     Webhook events --> Event Type --> Push
     FilterType --> Branch -> Branches --> master
 --> Trigger - no filter
 --> Build Provider - other providers -->  CodeBuild

--> Build: CodeBuild, 
    ProjectName = website, 
    Build type: Single
    Skip Test stage
--> Deploy provider : Amazon ECS, REgion, Clustername, ServiceName, Image definitions file: imagedefinitions.json (output file)

NOw Pipeline Starts

Now, see latest CodeBuild, latest ECR for latest image, and elb

Now modify the index.html in GitHub and automatically pipeline runs and deploys 
