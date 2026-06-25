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

- Writing a production-ready `Dockerfile` with **multi-stage builds**
- **Reducing Docker image size** by separating build and production stages
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
        │  docker build (multi-stage)
        ▼
     Docker Image
     (optimized — prod deps only)
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
| Docker (Multi-stage)  | Containerization with image size optimization|
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
├── Dockerfile               # Multi-stage container build instructions
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

# Verify the image was created and check its size
docker images
```

### Step 6 — Multi-Stage Docker Build & Image Size Optimization

One of the key optimizations in this project is using a **multi-stage Dockerfile** to significantly reduce the final image size.

**How it works:**

- **Stage 1 (Builder)** — Uses the full Node.js alpine image, installs all dependencies (including devDependencies), and runs tests. This stage is only used during the build process.
- **Stage 2 (Production)** — Starts from a fresh alpine image and runs `npm install --only=production`, skipping all dev dependencies like test frameworks and linters. Only production-ready files are copied from Stage 1.

**Result:** The final image contains zero dev tooling — just what's needed to run the app. This translates to a noticeably smaller image, faster ECR push times, faster ECS pull times, and a reduced attack surface in production.

```dockerfile
# ── Stage 1: Builder ──────────────────────────────────────────
# Full node image used only for installing deps and running tests
FROM node:12.2.0-alpine AS builder

# Working directory inside the builder stage
WORKDIR /node

# Copy package files first (layer caching trick — npm install
# only re-runs when package.json changes, not on every code change)
COPY package*.json ./

# Install all dependencies including devDependencies (needed for tests)
RUN npm install

# Copy rest of the source code
COPY . .

# Run tests in builder stage — if tests fail, build stops here
RUN npm run test


# ── Stage 2: Production ───────────────────────────────────────
# Fresh alpine image — nothing from builder stage carries over
# except what we explicitly COPY
FROM node:12.2.0-alpine

# Working directory in final image
WORKDIR /node

# Copy only package files
COPY package*.json ./

# Install ONLY production dependencies (no devDependencies)
# This is the key size reduction — no test libs, no build tools
RUN npm install --only=production

# Copy app source from builder stage (not from your local machine)
COPY --from=builder /node/app.js .
COPY --from=builder /node/views ./views

# Expose app port
EXPOSE 8000

# Start the app
CMD ["node", "app.js"]
```

### Step 7 — Create ECR Repository

```bash
aws ecr create-repository \
  --repository-name nodejs-todo-app \
  --region ap-south-1
```

### Step 8 — Authenticate Docker to ECR

```bash
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  <your-account-id>.dkr.ecr.ap-south-1.amazonaws.com
```

### Step 9 — Tag and Push Image to ECR

```bash
# Tag the image
docker tag nodejs-todo-app:latest \
  <your-account-id>.dkr.ecr.ap-south-1.amazonaws.com/nodejs-todo-app:latest

# Push to ECR
docker push \
  <your-account-id>.dkr.ecr.ap-south-1.amazonaws.com/nodejs-todo-app:latest
```

### Step 10 — Deploy on ECS Fargate

1. Go to **AWS Console → ECS → Create Cluster** → Select **Fargate**
2. Create a **Task Definition**:
   - Launch type: Fargate
   - Container image: your ECR image URI
   - Container port: `8000`
   - Memory: 512 MB, CPU: 0.25 vCPU
3. Create a **Service** inside the cluster using the task definition
4. Under **Networking**, select your VPC and public subnets; enable **Auto-assign public IP**
5. Click **Create Service** — ECS will pull the image from ECR and launch your container

### Step 11 — Verify Deployment

- Go to **ECS → Clusters → Your Cluster → Tasks**
- Click on the running task → copy the **Public IP**
- Open browser: `http://<public-ip>:8000`
- You should see the **Todo List app** live ✅

### Step 12 — Monitor with CloudWatch

- Go to **CloudWatch → Log Groups**
- Find the log group for your ECS task
- View container stdout logs to verify the app started correctly

---

## 🖼️ Screenshots

| Step | Screenshot |
|------|------------|
| Docker image size before vs after multi-stage build | *<img width="1117" height="234" alt="Screenshot (56)" src="https://github.com/user-attachments/assets/96c3addd-96e6-4a7d-8b5e-a54415046705" />
* |
| ECR Repository with pushed image | *<img width="1920" height="902" alt="Screenshot (57)" src="https://github.com/user-attachments/assets/d77277ff-a4fd-4f84-b839-bbb3bfdf4e07" />
* |
| ECS Task in RUNNING state | *<img width="1920" height="858" alt="Screenshot (58)" src="https://github.com/user-attachments/assets/188647eb-c8b3-4a22-b411-a8be3f15cd79" />
* |
| App live in browser via ECS public IP | *<img width="1920" height="1080" alt="Screenshot (59)" src="https://github.com/user-attachments/assets/c88cc1e1-46fd-4ebf-b584-242c9b0d851f" />
* |
| CloudWatch container logs | *<img width="1920" height="1080" alt="Screenshot (62)" src="https://github.com/user-attachments/assets/7a3096fb-344b-4fcf-8f67-b424ff22f143" />
* |
| IAM Role attached to EC2 | *<img width="1920" height="1080" alt="Screenshot (61)" src="https://github.com/user-attachments/assets/3112724c-7599-4a4f-bbc2-f6406942321b" />
* |

---

## 🔑 Key Concepts Demonstrated

**Multi-Stage Docker Builds** — The Dockerfile uses two stages: a builder stage that installs all dependencies and runs tests, and a production stage that starts fresh with only production dependencies. This keeps the final image lean, secure, and fast to pull.

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

- GitHub: [@sagar.smanjunath](https://github.com/sagar.smanjunath)
- LinkedIn: [linkedin.com/in/sagar-sm](https://linkedin.com/in/sagar-sm)

