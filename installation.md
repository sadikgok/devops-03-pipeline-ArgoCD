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

## 🗃️ 1. Jenkins Master Setup

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
```

Access Jenkins from your browser at:
```
http://<JENKINS_PUBLIC_IP>:8080
```

Login using the retrieved password, then create your admin user.

---

## 🧰 2. Jenkins Agent (Docker) Setup

### AWS Configuration
- Launch another EC2 instance named `JenkinsAgent`.
- Use the same Key Pair.

### Installation
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openjdk-21-jre -y
sudo apt install docker.io -y
sudo usermod -aG docker $USER
sudo reboot
```

### SSH Key Authentication between Master and Agent

**Purpose:** Jenkins Master connects to Jenkins Agent over SSH.

#### Step 1: Configure SSH on Both Machines
On **both Jenkins Master and Agent**, edit SSH configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Uncomment the following lines:
```
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2
```

Restart SSH service:
```bash
sudo systemctl restart sshd
```

#### Step 2: Generate SSH Key on Jenkins Master
On **Jenkins Master**:

```bash
cd /home/ubuntu
ssh-keygen
# Press Enter for all prompts (default location and no passphrase)

# Display the public key
cat ~/.ssh/id_ed25519.pub
```

Copy the entire output (starting with `ssh-ed25519 ...`).

#### Step 3: Add Master's Public Key to Agent
On **Jenkins Agent**:

```bash
cd /home/ubuntu/.ssh/
sudo nano authorized_keys
```

Paste the Master's public key at the bottom of the file (after the AWS KeyPair line).

Save and restart SSH:
```bash
sudo systemctl restart sshd
```

#### Step 4: Configure Jenkins Node
In Jenkins UI:

1. Go to **Manage Jenkins → Nodes → New Node**
2. Create `My-Jenkins-Agent`
3. Launch method: **Launch agents via SSH**
4. Host: **Agent's Private IP**
5. Credentials: Click **Add**
   - Kind: **SSH Username with private key**
   - Username: `ubuntu`
   - Private Key: **Enter directly**
   
   Get Master's private key:
   ```bash
   # On Jenkins Master
   sudo cat ~/.ssh/id_ed25519
   ```
   
   Copy the entire key including `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`

6. Save and verify the agent connects successfully.

#### Step 5: Disable Built-In Node Executors
Go to **Manage Jenkins → Nodes → Built-In Node → Configure**
- Set **Number of executors** to `0`
- Save

---

## 🔑 3. GitHub Integration

### Generate Personal Access Token
1. Go to: https://github.com/settings/tokens
2. Click **Generate new token (classic)**
3. Scopes: Select `repo`, `workflow`, `admin:repo_hook`
4. Generate and **copy the token** (shown only once!)

### Add Token to Jenkins
1. Go to **Manage Jenkins → Credentials → System → Global credentials**
2. Click **Add Credentials**
3. Kind: **Secret text**
4. Secret: Paste your GitHub token
5. ID: `GitHubTokenForJenkins`
6. Description: `GitHub Token for Jenkins`
7. Click **Create**

---

## 🧪 4. SonarQube & PostgreSQL Setup

### AWS Configuration
- Launch EC2 instance: `SonarQube`
- Open ports: **22 (SSH)**, **5432 (PostgreSQL)**, **9000 (SonarQube)**

### Installation

#### Step 1: Update System and Install PostgreSQL
```bash
# Update & install PostgreSQL
sudo apt update && sudo apt upgrade -y
sudo apt install postgresql postgresql-contrib -y
sudo systemctl enable postgresql
sudo systemctl status postgresql

# Set PostgreSQL password
sudo passwd postgres
# Enter password: admin
```

#### Step 2: Create SonarQube Database
```bash
su - postgres
# Enter password: admin

createuser sonar
psql

# Inside PostgreSQL prompt:
ALTER USER sonar WITH ENCRYPTED PASSWORD 'sonar';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
\q
exit
```

#### Step 3: Install Java 17 (Temurin)
```bash
sudo bash
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | \
tee /etc/apt/keyrings/adoptium.asc

echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] \
https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | \
tee /etc/apt/sources.list.d/adoptium.list

sudo apt update
sudo apt install temurin-17-jdk -y
exit
```

#### Step 4: Configure Linux Kernel Limits
```bash
# Edit limits.conf
sudo nano /etc/security/limits.conf
```

Add at the bottom:
```
sonarqube   -   nofile   65536
sonarqube   -   nproc    4096
```

```bash
# Edit sysctl.conf
sudo nano /etc/sysctl.conf
```

Add at the bottom:
```
vm.max_map_count = 262144
```

Reload system:
```bash
sudo sysctl -p
```

#### Step 5: Download and Configure SonarQube
```bash
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.6.0.92116.zip
sudo apt install unzip -y
sudo unzip sonarqube-10.6.0.92116.zip
sudo mv sonarqube-10.6.0.92116 sonarqube

# Create sonar user and group
sudo groupadd sonar
sudo useradd -d /opt/sonarqube -g sonar sonar
sudo chown -R sonar:sonar /opt/sonarqube
```

Edit configuration:
```bash
sudo nano /opt/sonarqube/conf/sonar.properties
```

Add these lines (uncomment if exists):
```properties
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
```

#### Step 6: Create SonarQube Service
```bash
sudo nano /etc/systemd/system/sonar.service
```

Paste:
```ini
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
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

Start the service:
```bash
sudo systemctl enable sonar
sudo systemctl start sonar
sudo systemctl status sonar

# Monitor logs (wait 2-3 minutes for startup)
sudo tail -f /opt/sonarqube/logs/sonar.log
```

#### Step 7: Access SonarQube
Navigate to:
```
http://<SONARQUBE_PUBLIC_IP>:9000
```

Login:
- **Username:** `admin`
- **Password:** `admin`

Change password immediately after first login.

#### Step 8: Create SonarQube Token for Jenkins
1. Click **Administrator** → **My Account** → **Security**
2. Generate Token:
   - Name: `SonarTokenForJenkins`
   - Type: `User Token`
   - Expires: `No expiration`
3. Copy the token (shown only once!)

#### Step 9: Add SonarQube Token to Jenkins
1. Go to **Manage Jenkins → Credentials → System → Global credentials**
2. Click **Add Credentials**
3. Kind: **Secret text**
4. Secret: Paste SonarQube token
5. ID: `SonarTokenForJenkins`
6. Description: `SonarQube Token for Jenkins`
7. Click **Create**

#### Step 10: Install SonarQube Plugins in Jenkins
1. Go to **Manage Jenkins → Plugins → Available plugins**
2. Search and install:
   - **SonarQube Scanner**
   - **Sonar Quality Gates**
3. Restart Jenkins if required

#### Step 11: Configure SonarQube in Jenkins
1. Go to **Manage Jenkins → System**
2. Scroll to **SonarQube servers**
3. Click **Add SonarQube**
   - Name: `SonarQube-Server`
   - Server URL: `http://<SONARQUBE_PRIVATE_IP>:9000`
   - Server authentication token: Select `SonarTokenForJenkins`
4. Save

#### Step 12: Configure SonarQube Scanner
1. Go to **Manage Jenkins → Tools**
2. Scroll to **SonarQube Scanner installations**
3. Click **Add SonarQube Scanner**
   - Name: `SonarQube-Scanner`
   - Install automatically: ✓ (check)
4. Save

#### Step 13: Create Webhook in SonarQube (For Quality Gate)
1. Login to SonarQube web interface
2. Go to **Administration → Configuration → Webhooks**
3. Click **Create**
   - Name: `Jenkins`
   - URL: `http://<JENKINS_PUBLIC_IP>:8080/sonarqube-webhook/`
4. Click **Create**

---

## 🛡️ 5. Trivy Integration

Trivy scans for vulnerabilities at two levels:

1. **File System Scan** (source code dependencies)
2. **Image Scan** (built Docker image)

### Installation on Jenkins Agent
```bash
# On Jenkins Agent machine
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y

# Verify installation
trivy --version
```

### Jenkinsfile Stages
Your pipeline should include:

```groovy
stage("Trivy File System Scan") {
    steps {
        sh "trivy fs --format table -o trivy-fs-report.html ."
    }
}

stage("Docker Build & Push") {
    steps {
        script {
            withDockerRegistry(credentialsId: 'DockerHubTokenForJenkins') {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
    }
}

stage("Trivy Image Scan") {
    steps {
        sh "trivy image --format json -o trivy-image-report.json ${IMAGE_NAME}:${IMAGE_TAG}"
        sh "trivy image --format table -o trivy-image-report.html ${IMAGE_NAME}:${IMAGE_TAG}"
    }
}
```

---

## 🐳 6. DockerHub Integration

### Create DockerHub Token
1. Login to https://hub.docker.com
2. Go to **Account Settings → Security → Access Tokens**
3. Click **New Access Token**
   - Description: `Jenkins Integration`
   - Access permissions: `Read, Write, Delete`
4. Generate and **copy the token**

### Add DockerHub Credentials to Jenkins
1. Go to **Manage Jenkins → Credentials → System → Global credentials**
2. Click **Add Credentials**
3. Kind: **Username with password**
4. Username: Your DockerHub username
5. Password: Paste the DockerHub token
6. ID: `DockerHubTokenForJenkins`
7. Description: `DockerHub Token for Jenkins`
8. Click **Create**

### Install Docker Plugins in Jenkins
1. Go to **Manage Jenkins → Plugins → Available plugins**
2. Search and install:
   - **Docker**
   - **Docker Pipeline**
3. Restart Jenkins if required

---

## ☸️ 7. AWS EKS Cluster Setup

### AWS Configuration
- Launch EC2 instance: `EKS-Server`
- Instance type: `t3.medium` or larger
- Open port: **22 (SSH)**

### Installation

#### Step 1: Install AWS CLI
```bash
sudo su
cd ~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update

# Verify installation
aws --version
```

#### Step 2: Install kubectl
```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.33.4/2025-08-20/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

#### Step 3: Install eksctl
```bash
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | \
tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/
eksctl version
```

#### Step 4: Configure AWS Credentials
```bash
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Default region: us-east-1
# Default output format: json
```

#### Step 5: Create EKS Cluster
```bash
eksctl create cluster --name my-workspace \
--region us-east-1 \
--node-type t3.large \
--nodes 2
```

**Note:** This process takes 15-20 minutes.

#### Step 6: Verify Cluster
```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

---

## 🌀 8. ArgoCD Setup (GitOps)

### Installation

#### Step 1: Create ArgoCD Namespace
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl get pods -n argocd
```

#### Step 2: Install ArgoCD CLI
```bash
sudo su
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
curl -sSL -o /usr/local/bin/argocd \
https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
exit
```

#### Step 3: Expose ArgoCD Server
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc -n argocd
```

Wait for `EXTERNAL-IP` to appear (AWS Load Balancer DNS).

#### Step 4: Get ArgoCD Admin Password
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml

# Decode password
echo <password-from-yaml> | base64 --decode
```

#### Step 5: Login to ArgoCD
Access ArgoCD UI:
```
http://<ARGOCD_LOADBALANCER_DNS>
```

Login:
- **Username:** `admin`
- **Password:** (decoded password from previous step)

**Important:** Change password immediately:
- Click **User Info** → **Update Password**

#### Step 6: Login via CLI
```bash
argocd login <ARGOCD_LOADBALANCER_DNS> --username admin
# Enter password when prompted

# List clusters
argocd cluster list
```

#### Step 7: Add EKS Cluster to ArgoCD
```bash
# Get cluster context name
kubectl config get-contexts

# Add cluster to ArgoCD
argocd cluster add <CONTEXT_NAME> --name my-workspace-cluster
```

### GitHub Integration for ArgoCD

#### Step 8: Create GitHub Token for ArgoCD
1. Go to: https://github.com/settings/tokens
2. Click **Generate new token (classic)**
3. Scopes: Select `repo` (full control)
4. Generate and **copy the token**

#### Step 9: Add GitHub Repository to ArgoCD
1. Login to ArgoCD UI
2. Go to **Settings → Repositories**
3. Click **Connect Repo**
   - Method: **HTTPS**
   - Type: **git**
   - Project: **default**
   - Repository URL: `https://github.com/<YOUR_USERNAME>/devops-03-pipeline-ArgoCD`
   - Username: Your GitHub username
   - Password: Paste GitHub token
4. Click **Connect**

#### Step 10: Create ArgoCD Application
1. Go to **Applications**
2. Click **+ New App**
   - Application Name: `devops-03-pipeline-cd`
   - Project: `default`
   - Sync Policy: **Automatic**
   - Sync Options: Check `PRUNE RESOURCES` and `SELF HEAL`
   
   **Source:**
   - Repository URL: `https://github.com/<YOUR_USERNAME>/devops-03-pipeline-ArgoCD`
   - Revision: `HEAD`
   - Path: `kubernetes/` (or your manifest directory)
   
   **Destination:**
   - Cluster URL: Select your cluster
   - Namespace: `default`
3. Click **Create**

---

## 🧭 9. Jenkins → ArgoCD Trigger (GitOps Automation)

### Step 1: Create GitHub Repository for GitOps
Create a new repository: `devops-03-pipeline-ArgoCD`
- This repo contains Kubernetes manifests
- ArgoCD watches this repo for changes

### Step 2: Create Jenkins API Token
1. Login to Jenkins
2. Click **admin** (top right) → **Configure**
3. Scroll to **API Token**
4. Click **Add new Token**
   - Name: `JENKINS_API_TOKEN`
5. Click **Generate** and **copy the token**

### Step 3: Add Jenkins API Token as Credential
1. Go to **Manage Jenkins → Credentials → System → Global credentials**
2. Click **Add Credentials**
3. Kind: **Secret text**
4. Secret: Paste Jenkins API token
5. ID: `JENKINS_API_TOKEN`
6. Click **Create**

### Step 4: Create Jenkins Job for ArgoCD
1. Create new Pipeline job: `devops-03-pipeline-ArgoCD`
2. Check **This project is parameterized**
   - Add **String Parameter**
   - Name: `IMAGE_TAG`
   - Default Value: `latest`
3. Check **Trigger builds remotely**
   - Authentication Token: `GITOPS_TRIGGER_START`
4. Pipeline script from SCM:
   - SCM: **Git**
   - Repository URL: `https://github.com/<YOUR_USERNAME>/devops-03-pipeline-ArgoCD`
   - Credentials: Select `GitHubTokenForJenkins`
   - Branch: `*/main`
5. Save

### Step 5: Add Trigger Stage to Main Pipeline
In your main Jenkins pipeline (`devops-03-pipeline-aws-gitops`), add:

```groovy
environment {
    JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
}

stage("Trigger CD Pipeline") {
    steps {
        script {
            sh """
                curl -v -k --user <YOUR_JENKINS_USERNAME>:${JENKINS_API_TOKEN} \
                -X POST \
                -H 'content-type: application/x-www-form-urlencoded' \
                --data 'IMAGE_TAG=${IMAGE_TAG}' \
                'http://<JENKINS_PUBLIC_IP>:8080/job/devops-03-pipeline-ArgoCD/buildWithParameters?token=GITOPS_TRIGGER_START'
            """
        }
    }
}
```

### How It Works
1. Main Jenkins pipeline builds Docker image
2. Pushes image to DockerHub with new tag
3. Triggers ArgoCD pipeline with `IMAGE_TAG`
4. ArgoCD pipeline updates Kubernetes manifest
5. Commits changes to GitHub
6. ArgoCD detects change and syncs to EKS cluster

---

## ⚖️ 10. Horizontal Pod Autoscaler (HPA)

### Why HPA?
HPA automatically scales Pod replicas based on CPU/Memory usage or custom metrics.

### Prerequisites
1. **Metrics Server** must be installed (usually pre-installed on EKS)
2. **Resource limits** must be defined in Deployment

### Step 1: Verify Metrics Server
```bash
kubectl get deployment metrics-server -n kube-system
```

If not installed:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Step 2: Add Resource Limits to Deployment
Edit your `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-03-pipeline-cd-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devops-03-pipeline-cd
  template:
    metadata:
      labels:
        app: devops-03-pipeline-cd
    spec:
      containers:
      - name: devops-03-pipeline-cd
        image: <YOUR_DOCKERHUB_USERNAME>/devops-03-pipeline-aws:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

### Step 3: Create HPA Manifest
Create `hpa.yaml`:

```yaml
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
```

### Step 4: Apply HPA
```bash
kubectl apply -f hpa.yaml
```

### Step 5: Verify HPA
```bash
kubectl get hpa

# Watch HPA in real-time
kubectl get hpa -w
```

Expected output:
```
NAME                         REFERENCE                                  TARGETS   MINPODS   MAXPODS   REPLICAS
devops-03-pipeline-cd-hpa    Deployment/devops-03-pipeline-cd-deployment   0%/50%    2         10        2
```

### Step 6: Test Autoscaling (Optional)
Generate load:
```bash
# Get service URL
kubectl get svc

# Generate traffic (install Apache Bench)
sudo apt-get install apache2-utils -y
ab -n 10000 -c 100 http://<SERVICE_URL>:8080/
```

Watch HPA scale up:
```bash
kubectl get hpa -w
```

---

## 📧 11. Email Notifications (Optional)

### Configure Gmail in Jenkins
1. Go to **Manage Jenkins → System**
2. Scroll to **Extended E-mail Notification**
3. SMTP server: `smtp.gmail.com`
4. SMTP Port: `465`
5. Click **Advanced**
   - Use SSL: ✓
   - Username: Your Gmail
   - Password: App-specific password (not Gmail password)
6. Save

### Add Email Stage to Pipeline
```groovy
post {
    always {
        emailext (
            subject: "Pipeline Status: ${BUILD_STATUS}",
            body: """
                <p>Build Status: ${BUILD_STATUS}</p>
                <p>Build Number: ${BUILD_NUMBER}</p>
                <p>Check console output at ${BUILD_URL}</p>
            """,
            to: 'your-email@gmail.com',
            mimeType: 'text/html'
        )
    }
}
```

---

## 🧹 12. Cleanup Commands

### Delete EKS Cluster
```bash
# Set region
export AWS_DEFAULT_REGION=us-east-1

# Delete cluster (this deletes all nodes and resources)
eksctl delete cluster --name my-workspace --region us-east-1
```

### Clean Docker on Jenkins Agent
```bash
# Remove old images
docker rmi $(docker images --format '{{.Repository}}:{{.Tag}}' | grep 'devops-003-pipeline-aws')

# Remove all containers
docker container rm -f $(docker container ls -aq)

# Remove unused volumes
docker volume prune -f

# Full cleanup
docker system prune -a -f
```

### Stop AWS EC2 Instances
Don't forget to **stop or terminate** EC2 instances to avoid charges:
- JenkinsMaster
- JenkinsAgent
- SonarQube
- EKS-Server

---

## 🔧 13. Troubleshooting

### Jenkins Agent Not Connecting
```bash
# On Agent, check SSH logs
sudo tail -f /var/log/auth.log

# On Master, test SSH connection
ssh -i ~/.ssh/id_ed25519 ubuntu@<AGENT_PRIVATE_IP>
```

### SonarQube Not Starting
```bash
# Check logs
sudo tail -f /opt/sonarqube/logs/sonar.log

# Check Java version (must be 17)
java -version

# Check PostgreSQL
sudo systemctl status postgresql
```

### ArgoCD Application Not Syncing
```bash
# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller

# Force sync from CLI
argocd app sync <APP_NAME>

# Check application status
argocd app get <APP_NAME>
```

### HPA Not Working
```bash
# Check metrics server
kubectl get deployment metrics-server -n kube-system

# Check pod metrics
kubectl top pods

# Describe HPA
kubectl describe hpa <HPA_NAME>
```

### EKS Cluster Creation Failed
```bash
# Check CloudFormation stacks
aws cloudformation describe-stacks --region us-east-1

# Delete failed cluster
eksctl delete cluster --name my-workspace --region us-east-1
```

---

## 📊 14. Monitoring and Verification

### Verify Full Pipeline
1. **Jenkins:** http://<JENKINS_IP>:8080
2. **SonarQube:** http://<SONARQUBE_IP>:9000
3. **ArgoCD:** http://<ARGOCD_LOADBALANCER_DNS>
4. **Application:** http://<K8S_SERVICE_LOADBALANCER>:8080

### Check Pipeline Flow
```bash
# On Jenkins Agent
docker images | grep devops

# On EKS Server
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get hpa

# Check ArgoCD sync status
argocd app list
argocd app get devops-03-pipeline-cd
```

---

## ✅ Final Result

When all steps are completed successfully:

✓ **Jenkins** builds your application  
✓ **SonarQube** validates code quality  
✓ **Trivy** ensures image security  
✓ **Docker** packages and stores images  
✓ **ArgoCD** deploys automatically to EKS  
✓ **HPA** scales based on traffic  
✓ **Gmail** sends build notifications  

---

## 🎯 Next Steps

Consider implementing:
- **Terraform** for infrastructure automation
- **Prometheus & Grafana** for monitoring
- **ELK Stack** for centralized logging
- **Helm Charts** for better Kubernetes management
- **Sealed Secrets** for secure secret management

---

## 📚 References

- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [SonarQube Documentation](https://docs.sonarqube.org/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

🎉 **Congratulations! You now have a fully automated CI/CD + GitOps pipeline on AWS!**