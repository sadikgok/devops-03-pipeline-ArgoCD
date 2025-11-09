# 🚀 AWS DevOps CI/CD & GitOps Pipeline

This repository demonstrates a **production-grade CI/CD & GitOps pipeline** deployed on **AWS EKS**, integrating modern DevOps tools to ensure continuous integration, delivery, and deployment with full automation, quality, and security.

## 🧩 Project Overview
This project builds an automated pipeline for a Java (Maven) application that:
- Pulls source code from **GitHub**
- Builds and tests via **Jenkins**
- Performs **static code analysis** using **SonarQube**
- Runs **vulnerability scans** using **Trivy**
- Builds and pushes Docker images to **DockerHub**
- Deploys to **Kubernetes (EKS)** via **Helm** and **ArgoCD (GitOps)**
- Sends build result notifications via **Gmail**

> All infrastructure components (Jenkins, SonarQube, EKS Server) are hosted on **AWS EC2 instances**.  
> The next version will include **Terraform automation** for full infrastructure provisioning.

---

## 🧠 Architecture Overview
The end-to-end DevOps pipeline consists of the following stages:

1. **Code Commit** → Developers push code to GitHub.  
2. **CI Trigger** → Jenkins triggers build using GitHub Webhook/Token.  
3. **Build & Test** → Code is compiled with Maven on Jenkins Agent.  
4. **Code Quality** → SonarQube (PostgreSQL backend) analyzes the code and checks Quality Gates.  
5. **Security Scan** → Trivy scans Docker image for vulnerabilities.  
6. **Docker Build & Push** → Secure image pushed to DockerHub.  
7. **GitOps Deployment** → ArgoCD + Helm automatically deploy the application to AWS EKS Cluster.  
8. **Notification** → Build results sent via Gmail.

---

## 🧰 Tools & Technologies

| Category | Tool / Service |
|-----------|----------------|
| Cloud Infrastructure | AWS EC2, EKS |
| CI/CD | Jenkins (Master-Agent setup) |
| Code Quality | SonarQube + PostgreSQL |
| Security | Trivy |
| Containerization | Docker |
| Deployment | ArgoCD + Helm |
| Infrastructure as Code | Terraform (Planned) |
| Notification | Gmail |
| Monitoring (Optional) | Prometheus + Grafana |

---

## 🧱 System Architecture Diagram

*(You can replace this placeholder with your own diagram image)*  
![Pipeline Architecture](./docs/pipeline-architecture.png)

---

## ⚙️ Prerequisites
Before setup, ensure the following:
- AWS Account with EC2 access  
- AWS CLI installed and configured  
- SSH Key Pair for EC2 connectivity  
- GitHub Personal Access Token  
- DockerHub account (with Access Token)

---

## 🔄 Pipeline Workflow

```text
GitHub → Jenkins → SonarQube → Trivy → DockerHub → ArgoCD → Kubernetes (EKS) → Notification

