# ⚙️ Installation & Configuration Guide  
### Complete DevOps CI/CD & GitOps Pipeline on AWS EKS

This document provides the **detailed setup instructions** for deploying a full end-to-end DevOps pipeline integrating:
**Jenkins, SonarQube, Trivy, DockerHub, ArgoCD, and AWS EKS (Kubernetes)**.

# ⚙️ Installation & Configuration Guide  
### Complete DevOps CI/CD & GitOps Pipeline on AWS EKS

This document provides the **detailed setup instructions** for deploying a full end-to-end DevOps pipeline integrating:
**Jenkins, SonarQube, Trivy, DockerHub, ArgoCD, and AWS EKS (Kubernetes)**.

---

## 🧩 Prerequisites

Before starting, ensure that you have:

- An active **AWS Account**  
- **AWS CLI** installed on your local machine  
- A valid **SSH Key Pair** for connecting to EC2 instances  
- **GitHub Personal Access Token** (for Jenkins and ArgoCD integration)  
- **DockerHub Account & Token** (for pushing images)  
- **Mobaxterm** or a terminal capable of SSH  

---

## 🏗️ 1. Jenkins Master Setup

### AWS Configuration
- Launch an EC2 instance named `JenkinsMaster`.
- Open ports **22 (SSH)** and **8080 (Jenkins Web UI)** in its Security Group.

### Installation Steps
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Java 21
sudo apt install openjdk-21-jre -y

# Install Jenkins
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" | \
sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y

# Enable and start Jenkins service
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins

# Retrieve Jenkins admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

Access Jenkins from your browser at:

http://<JENKINS_PUBLIC_IP>:8080


Login using the retrieved password, then create your admin user.

🧰 2. Jenkins Agent (Docker) Setup
AWS Configuration

Launch another EC2 instance named JenkinsAgent.

Use the same Key Pair.

Installation
sudo apt update && sudo apt upgrade -y
sudo apt install openjdk-21-jre -y
sudo apt install docker.io -y
sudo usermod -aG docker $USER
sudo reboot

SSH Key Authentication between Master and Agent

Purpose: Jenkins Master connects to Jenkins Agent over SSH.

On both machines, edit SSH configuration:

sudo nano /etc/ssh/sshd_config
# Uncomment the following lines:
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2
sudo systemctl restart sshd


On Jenkins Master, generate a key pair:

cd /home/ubuntu
ssh-keygen
cat ~/.ssh/id_ed25519.pub


Copy the public key content into Agent’s ~/.ssh/authorized_keys file:

sudo nano ~/.ssh/authorized_keys
# Paste the master public key at the bottom
sudo systemctl restart sshd


In Jenkins UI:

Go to Manage Jenkins → Nodes → New Node

Create My-Jenkins-Agent (Launch via SSH)

Use Agent’s Private IP

Add credentials: SSH Username with private key using Master’s private key (sudo cat ~/.ssh/id_ed25519)

🔑 3. GitHub Integration

Generate a Personal Access Token on GitHub:
👉 https://github.com/settings/tokens

Scope: repo, workflow, admin:repo_hook

Save this token (it’s shown only once).

In Jenkins → Manage Credentials → Add new “Secret Text” credential.
Example ID: GitHubTokenForJenkins.

🧪 4. SonarQube & PostgreSQL Setup
AWS Configuration

Launch EC2 instance: SonarQube

Open ports: 22 (SSH), 5432 (PostgreSQL), 9000 (SonarQube)

Installation
# Update & install PostgreSQL
sudo apt update && sudo apt upgrade -y
sudo apt install postgresql postgresql-contrib -y
sudo systemctl enable postgresql
sudo passwd postgres
# password: admin

Create SonarQube Database
su - postgres
createuser sonar
psql
ALTER USER sonar WITH ENCRYPTED PASSWORD 'sonar';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
\q
exit

Install Java 17 (Temurin)
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | \
tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] \
https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | \
tee /etc/apt/sources.list.d/adoptium.list
sudo apt update
sudo apt install temurin-17-jdk -y

Configure SonarQube
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.6.zip
sudo apt install unzip -y
sudo unzip sonarqube-10.6.zip -d /opt
sudo mv /opt/sonarqube-10.6 /opt/sonarqube
sudo groupadd sonar
sudo useradd -d /opt/sonarqube -g sonar sonar
sudo chown -R sonar:sonar /opt/sonarqube


Edit configuration file:

sudo nano /opt/sonarqube/conf/sonar.properties


Add:

sonar.jdbc.username=sonar
sonar.jdbc.password=sonar
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube

Create SonarQube Service
sudo nano /etc/systemd/system/sonar.service


Paste:

[Unit]
Description=SonarQube Service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonar
Group=sonar
Restart=always

[Install]
WantedBy=multi-user.target


Start the service:

sudo systemctl enable sonar
sudo systemctl start sonar
sudo systemctl status sonar


Access:

http://<SONARQUBE_PUBLIC_IP>:9000
Username: admin | Password: admin


Create a token named SonarTokenForJenkins under My Account → Security.
In Jenkins, add this token as a Secret Text Credential.

🛡️ 5. Trivy Integration

Trivy scans for vulnerabilities at two levels:

File System Scan (source code)

Image Scan (built Docker image)

Jenkinsfile includes stages:

stage("Trivy File System Scan")
stage("Docker Build & Push")
stage("Trivy Image Scan - JSON + HTML")


This ensures security checks before and after Docker image creation.

☸️ 6. AWS EKS Cluster Setup
Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install

Install kubectl
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.33.4/2025-08-20/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client

Install eksctl
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | \
tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/
eksctl version

Create EKS Cluster
eksctl create cluster --name my-workspace \
--region us-east-1 \
--node-type t3.large \
--nodes 2

🌀 7. ArgoCD Setup (GitOps)
Installation
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd

Install ArgoCD CLI
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
curl -sSL -o /usr/local/bin/argocd \
https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

Expose ArgoCD Server
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc -n argocd


Access the URL shown (LoadBalancer DNS).
Retrieve ArgoCD admin password:

kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
# Decode:
echo <encoded-password> | base64 --decode


Login:

argocd login <ARGOCD_SERVER_URL> --username admin

🧭 8. Jenkins → ArgoCD Trigger (GitOps Automation)

In Jenkins main pipeline:

environment {
    JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
}
stage("Trigger CD Pipeline") {
    steps {
        script {
            sh "curl -v -k --user sadikgok:${JENKINS_API_TOKEN} \
            -X POST -H 'content-type: application/x-www-form-urlencoded' \
            --data 'IMAGE_TAG=${IMAGE_TAG}' \
            'http://<JENKINS_IP>:8080/job/devops-03-pipeline-ArgoCD/buildWithParameters?token=GITOPS_TRIGGER_START'"
        }
    }
}


ArgoCD monitors the GitHub repository (devops-03-pipeline-ArgoCD) and automatically deploys new images to the EKS cluster.

⚖️ 9. Horizontal Pod Autoscaler (HPA)

To enable auto-scaling based on CPU usage:

Example Deployment Resource Limits:
resources:
  limits:
    memory: "256Mi"
    cpu: "500m"

HPA Manifest Example
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: devops-03-pipeline-cd-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: devops-03-pipeline-cd-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50


Apply and verify:

kubectl apply -f hpa.yaml
kubectl get hpa

🧹 10. Cleanup Commands
Delete EKS Cluster
eksctl delete cluster --name my-workspace --region us-east-1

Remove Old Docker Images & Containers
docker rmi $(docker images --format '{{.Repository}}:{{.Tag}}' | grep 'devops-003-pipeline-aws')
docker container rm -f $(docker container ls -aq)
docker volume prune -f

✅ Result

When all steps are completed successfully:

Jenkins builds your application

SonarQube validates code quality

Trivy ensures image security

ArgoCD deploys automatically to EKS

Gmail sends build notifications

🎉 You now have a fully automated CI/CD + GitOps pipeline on AWS!
