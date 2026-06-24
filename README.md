# 🚀 Node.js Todo App — Dockerized Deployment on AWS ECS Fargate

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![Amazon ECR](https://img.shields.io/badge/Amazon%20ECR-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Amazon ECS](https://img.shields.io/badge/Amazon%20ECS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)

A fully containerized **Node.js Todo List application** deployed on **AWS ECS Fargate** using Docker, Amazon ECR, and IAM Roles — built to demonstrate real-world cloud deployment practices without hardcoded credentials.

---

## 📌 Project Overview

This project walks through the complete lifecycle of deploying a Node.js web application to the cloud:

- Writing a production-ready `Dockerfile`
- Building and tagging Docker images on an EC2 instance
- Authenticating to AWS ECR using IAM Roles (no access keys)
- Pushing the image to a private ECR repository
- Running the container as a Fargate task on Amazon ECS
- Monitoring container logs via CloudWatch

---

## 🏗️ Architecture

```
GitHub (Source Code)
        │
        ▼
   EC2 Instance  ◄──── IAM Role (ECR Access)
   (Build Server)
        │
        │  docker build
        ▼
     Docker Image
        │
        │  docker push
        ▼
  Amazon ECR (ap-south-1)
  [Private Container Registry]
        │
        │  pull image
        ▼
  Amazon ECS — Fargate
  [Serverless Container Runtime]
        │
        ▼
   App Live on Port 8000
        │
        ▼
  CloudWatch Logs
  [Container Monitoring]
```
<img width="1356" height="1158" alt="image" src="https://github.com/user-attachments/assets/4c4ef9b9-bb23-41e7-b8e9-5de937bce8fd" />

---

## 🛠️ Tech Stack

| Tool / Service        | Purpose                                      |
|-----------------------|----------------------------------------------|
| Node.js + Express     | Backend web framework                        |
| EJS                   | Server-side HTML templating                  |
| Docker                | Containerization                             |
| AWS EC2               | Build server to create the Docker image      |
| AWS IAM Role          | Secure, keyless authentication to ECR        |
| AWS ECR               | Private Docker image registry                |
| AWS ECS Fargate       | Serverless container deployment              |
| AWS CloudWatch        | Container log monitoring                     |
| GitHub                | Source code management                       |

---

## 📁 Project Structure

```
nodejs-aws-ecs-deployment/
├── app.js                   # Main Express application entry point
├── Dockerfile               # Container build instructions
├── docker-compose.yaml      # Local multi-container setup
├── package.json             # Node.js dependencies
├── package-lock.json        # Locked dependency versions
├── Jenkinsfile              # Jenkins CI/CD pipeline definition
├── azure-pipelines.yml      # Azure DevOps pipeline (alternate CI)
├── sonar-project.properties # SonarQube code quality config
├── test.js                  # Unit tests
├── .gitignore
├── views/                   # EJS HTML templates
│   └── todo.ejs             # Todo list UI
├── DevSecOps/               # Security-integrated pipeline configs
├── k8s/                     # Kubernetes manifests
├── kustomize/               # Kustomize overlays for k8s
└── terraform/               # Infrastructure as Code (IaC)
```

---

## ✅ Prerequisites

Before you begin, make sure you have the following:

- An **AWS Account** with permissions for EC2, ECR, ECS, IAM, and CloudWatch
- **AWS CLI** installed and configured (`aws configure`)
- **Docker** installed on your local machine or EC2 instance
- **Node.js** (v14+) and **npm** installed
- **Git** installed

---

## 🚀 Deployment Guide

### Step 1 — Clone the Repository

```bash
git clone https://github.com/appusagar077/nodejs-aws-ecs-deployment.git
cd nodejs-aws-ecs-deployment
```

### Step 2 — Run Locally (Optional)

```bash
npm install
node app.js
# App runs at http://localhost:8000
```

### Step 3 — Launch EC2 Instance (Build Server)

- Launch an EC2 instance (Amazon Linux 2, t2.micro)
- Attach an **IAM Role** with the `AmazonEC2ContainerRegistryFullAccess` policy
- Open port 8000 in the Security Group for testing

> ⚠️ Using an IAM Role attached to EC2 means **no access keys are hardcoded** — this is the production-safe approach.

### Step 4 — Install Docker on EC2

```bash
sudo yum update -y
sudo yum install docker -y
sudo service docker start
sudo usermod -aG docker ec2-user
# Log out and log back in for group changes to apply
```

### Step 5 — Clone Repo on EC2 and Build Docker Image

```bash
git clone https://github.com/appusagar077/nodejs-aws-ecs-deployment.git
cd nodejs-aws-ecs-deployment

# Build the Docker image
docker build -t nodejs-todo-app .

# Verify the image was created
docker images
```

### Step 6 — Create ECR Repository

```bash
aws ecr create-repository \
  --repository-name nodejs-todo-app \
  --region ap-south-1
```

### Step 7 — Authenticate Docker to ECR

```bash
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  <your-account-id>.dkr.ecr.ap-south-1.amazonaws.com
```

### Step 8 — Tag and Push Image to ECR

```bash
# Tag the image
docker tag nodejs-todo-app:latest \
  <your-account-id>.dkr.ecr.ap-south-1.amazonaws.com/nodejs-todo-app:latest

# Push to ECR
docker push \
  <your-account-id>.dkr.ecr.ap-south-1.amazonaws.com/nodejs-todo-app:latest
```

### Step 9 — Deploy on ECS Fargate

1. Go to **AWS Console → ECS → Create Cluster** → Select **Fargate**
2. Create a **Task Definition**:
   - Launch type: Fargate
   - Container image: your ECR image URI
   - Container port: `8000`
   - Memory: 512 MB, CPU: 0.25 vCPU
3. Create a **Service** inside the cluster using the task definition
4. Under **Networking**, select your VPC and public subnets; enable **Auto-assign public IP**
5. Click **Create Service** — ECS will pull the image from ECR and launch your container

### Step 10 — Verify Deployment

- Go to **ECS → Clusters → Your Cluster → Tasks**
- Click on the running task → copy the **Public IP**
- Open browser: `http://<public-ip>:8000`
- You should see the **Todo List app** live ✅

### Step 11 — Monitor with CloudWatch

- Go to **CloudWatch → Log Groups**
- Find the log group for your ECS task
- View container stdout logs to verify the app started correctly

---

## 🐳 Dockerfile

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 8000

CMD ["node", "app.js"]
```

---

## 🖼️ Screenshots

| Step | Screenshot |
|------|------------|
| ECR Repository with pushed image | *(add screenshot)* |
| ECS Task in RUNNING state | *(add screenshot)* |
| App live in browser via ECS public IP | *(add screenshot)* |
| CloudWatch container logs | *(add screenshot)* |
| IAM Role attached to EC2 | *(add screenshot)* |

---

## 🔑 Key Concepts Demonstrated

**IAM Role over Access Keys** — Instead of storing AWS credentials on the EC2 instance, an IAM Role is attached directly. EC2 automatically gets temporary credentials, which is the secure, production-grade approach.

**Private ECR Registry** — The Docker image is stored in a private AWS-managed registry, not Docker Hub. Only authorized AWS identities can pull from it.

**ECS Fargate (Serverless Containers)** — No EC2 instances to manage for the runtime. AWS handles the underlying infrastructure; you only define what your container needs.

**CloudWatch for Observability** — Container logs are streamed to CloudWatch automatically, giving you a single place to debug and monitor your running application.

---

## 🧹 Cleanup (Avoid AWS Charges)

```bash
# Delete ECS Service and Cluster from AWS Console
# Then delete the ECR repository
aws ecr delete-repository \
  --repository-name nodejs-todo-app \
  --region ap-south-1 \
  --force

# Terminate EC2 instance from AWS Console
```

> ⚠️ Always clean up resources after testing to avoid unexpected AWS charges.

---

## 🙋 Author

**Sagar SM**
Cloud & DevOps Engineer | AWS | Kubernetes | Terraform | Jenkins

- GitHub: [@appusagar077](https://github.com/appusagar077)
- LinkedIn: [linkedin.com/in/sagar-sm](https://linkedin.com/in/sagar-sm)

