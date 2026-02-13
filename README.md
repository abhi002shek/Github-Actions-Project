# 🚀 Complete CI/CD Pipeline with GitHub Actions, Docker & AWS EKS

A fully automated CI/CD pipeline that deploys a Java Spring Boot banking application to AWS EKS (Kubernetes) using GitHub Actions, Docker, and Terraform.

## 📋 Table of Contents
- [Architecture Overview](#architecture-overview)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Setup Instructions](#setup-instructions)
- [Pipeline Stages](#pipeline-stages)
- [Deployment Guide](#deployment-guide)
- [Troubleshooting](#troubleshooting)
- [Cost Optimization](#cost-optimization)

---

## 🏗️ Architecture Overview

```
Developer Push → GitHub → GitHub Actions (Self-Hosted Runner)
                              ↓
                    ┌─────────┴─────────┐
                    │   CI/CD Pipeline   │
                    │  1. Compile        │
                    │  2. Security Scan  │
                    │  3. Test           │
                    │  4. Build & QA     │
                    │  5. Docker Build   │
                    │  6. Deploy to EKS  │
                    └─────────┬─────────┘
                              ↓
                    AWS EKS Cluster (3 nodes)
                              ↓
                    Application Running on Kubernetes
```

---

## 🛠️ Tech Stack

### CI/CD & DevOps Tools
- **GitHub Actions** - CI/CD orchestration
- **Self-Hosted Runner** - Custom build environment
- **Docker** - Containerization
- **Docker Hub** - Container registry
- **Terraform** - Infrastructure as Code
- **kubectl** - Kubernetes CLI

### Security & Quality
- **SonarQube** - Code quality analysis
- **Trivy** - Filesystem vulnerability scanning
- **Gitleaks** - Secret detection

### Cloud & Infrastructure
- **AWS EKS** - Managed Kubernetes service
- **AWS VPC** - Network isolation
- **AWS EC2** - Worker nodes
- **AWS IAM** - Access management

### Application Stack
- **Java 17** - Programming language
- **Spring Boot** - Application framework
- **Maven** - Build tool
- **MySQL** - Database

---

## ✅ Prerequisites

### 1. AWS Account
- Active AWS account with billing enabled
- IAM user with admin permissions
- AWS CLI configured

### 2. GitHub Account
- GitHub repository
- Personal Access Token (PAT)

### 3. Docker Hub Account
- Docker Hub username
- Access token

### 4. Local Tools
```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install Terraform
wget https://releases.hashicorp.com/terraform/1.10.3/terraform_1.10.3_linux_amd64.zip
unzip terraform_1.10.3_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

---

## 📁 Project Structure

```
Github-Actions-Project/
├── .github/
│   └── workflows/
│       └── cicd.yaml              # Main CI/CD pipeline
├── terraform-eks/                 # EKS infrastructure code
│   ├── main.tf                    # Main Terraform configuration
│   ├── variables.tf               # Input variables
│   ├── output.tf                  # Output values
│   └── README.md                  # Terraform documentation
├── src/                           # Java application source
├── Dockerfile                     # Container image definition
├── ds.yml                         # Kubernetes deployment manifest
├── pom.xml                        # Maven configuration
└── README.md                      # This file
```

---

## 🚀 Setup Instructions

### Step 1: Fork/Clone Repository

```bash
git clone https://github.com/abhi002shek/Github-Actions-Project.git
cd Github-Actions-Project
```

### Step 2: Configure AWS Credentials

```bash
aws configure
# Enter:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region: ap-south-1
# - Default output format: json
```

### Step 3: Create EKS Cluster with Terraform

```bash
cd terraform-eks

# Initialize Terraform
terraform init

# Review the plan
terraform plan

# Create infrastructure (takes 10-15 minutes)
terraform apply -auto-approve

# Configure kubectl
aws eks update-kubeconfig --name devopsshack-cluster --region ap-south-1

# Verify cluster
kubectl get nodes
```

### Step 4: Set Up Self-Hosted GitHub Runner

**Launch EC2 Instance:**
```bash
# Create EC2 instance (Ubuntu 22.04, t2.medium)
aws ec2 run-instances \
  --image-id ami-0dee22c13ea7a9a67 \
  --instance-type t2.medium \
  --key-name your-key-name \
  --security-group-ids sg-xxxxx \
  --subnet-id subnet-xxxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=github-runner}]'
```

**Install Runner:**
```bash
# SSH into the instance
ssh -i your-key.pem ubuntu@<instance-ip>

# Download and configure runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Configure (get token from GitHub repo Settings → Actions → Runners)
./config.sh --url https://github.com/YOUR_USERNAME/Github-Actions-Project --token YOUR_TOKEN

# Start runner
./run.sh
```

**Install Required Tools on Runner:**
```bash
# Install Docker
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker ubuntu
sudo systemctl restart docker

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS
aws configure
```

### Step 5: Configure GitHub Secrets

Go to: `Settings → Secrets and variables → Actions`

**Add Secrets:**
- `AWS_ACCESS_KEY_ID` - Your AWS access key
- `AWS_SECRET_ACCESS_KEY` - Your AWS secret key
- `DOCKERHUB_TOKEN` - Docker Hub access token
- `SONAR_TOKEN` - SonarQube token

**Add Variables:**
- `DOCKERHUB_USERNAME` - Your Docker Hub username
- `SONAR_HOST_URL` - SonarQube server URL

### Step 6: Deploy Application

```bash
# Push code to trigger pipeline
git add .
git commit -m "Initial deployment"
git push origin main

# Monitor pipeline in GitHub Actions tab
```

---

## 🔄 Pipeline Stages

### 1. **Compile** (Job 1)
```yaml
- Checkout code
- Set up JDK 17
- Compile with Maven
```

### 2. **Security Check** (Job 2)
```yaml
- Install Trivy
- Scan filesystem for vulnerabilities
- Install Gitleaks
- Scan for secrets in code
```

### 3. **Test** (Job 3)
```yaml
- Run unit tests
- Generate test reports
```

### 4. **Build & SonarQube Scan** (Job 4)
```yaml
- Build JAR with Maven
- Upload artifact
- Run SonarQube analysis
- Check quality gate
```

### 5. **Docker Build & Push** (Job 5)
```yaml
- Download JAR artifact
- Build Docker image
- Push to Docker Hub
```

### 6. **Deploy to EKS** (Job 6)
```yaml
- Configure AWS credentials
- Update kubeconfig
- Deploy to Kubernetes
```

---

## 📦 Deployment Guide

### Manual Deployment to EKS

```bash
# 1. Configure kubectl
aws eks update-kubeconfig --name devopsshack-cluster --region ap-south-1

# 2. Verify cluster access
kubectl get nodes

# 3. Deploy application
kubectl apply -f ds.yml

# 4. Check deployment status
kubectl get deployments
kubectl get pods
kubectl get services

# 5. Get application URL
kubectl get svc bankapp-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Access Application

```bash
# Get Load Balancer URL
LOAD_BALANCER=$(kubectl get svc bankapp-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Access application
curl http://$LOAD_BALANCER:8080

# Or open in browser
echo "http://$LOAD_BALANCER:8080"
```

---

## 🐛 Troubleshooting

### Issue 1: Pipeline Fails - "kubectl: command not found"

**Solution:**
```bash
# SSH into runner
ssh -i your-key.pem ubuntu@runner-ip

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

### Issue 2: Docker Permission Denied

**Solution:**
```bash
# Add user to docker group
sudo usermod -aG docker ubuntu

# Restart runner
cd ~/actions-runner
pkill -f Runner.Listener
./run.sh
```

### Issue 3: AWS CLI Installation Fails

**Solution:**
```bash
# Use --update flag
sudo ./aws/install --update
```

### Issue 4: EKS Cluster Not Accessible

**Solution:**
```bash
# Update kubeconfig
aws eks update-kubeconfig --name devopsshack-cluster --region ap-south-1

# Verify
kubectl get nodes
```

### Issue 5: Pods Not Starting

**Solution:**
```bash
# Check pod logs
kubectl logs <pod-name>

# Describe pod for events
kubectl describe pod <pod-name>

# Check if image is pulled
kubectl get pods -o wide
```

---

## 💰 Cost Optimization

### Destroy Resources After Testing

```bash
# 1. Delete Kubernetes resources
kubectl delete -f ds.yml

# 2. Destroy EKS cluster
cd terraform-eks
terraform destroy -auto-approve

# 3. Verify deletion
aws eks list-clusters --region ap-south-1
aws ec2 describe-vpcs --region ap-south-1 --filters 'Name=is-default,Values=false'

# 4. Stop/Terminate EC2 instances
aws ec2 stop-instances --instance-ids i-xxxxx
# or
aws ec2 terminate-instances --instance-ids i-xxxxx
```

### Estimated Costs (ap-south-1 region)

| Resource | Type | Cost/Hour | Cost/Month |
|----------|------|-----------|------------|
| EKS Cluster | Control Plane | $0.10 | ~$73 |
| EC2 Nodes | 3x t2.medium | $0.15 | ~$108 |
| Load Balancer | Classic ELB | $0.025 | ~$18 |
| **Total** | | **~$0.275/hr** | **~$199/month** |

**💡 Tip:** Destroy resources when not in use to avoid charges!

---

## 📊 Monitoring & Logs

### View Pipeline Logs
```bash
# GitHub Actions UI
https://github.com/YOUR_USERNAME/Github-Actions-Project/actions
```

### View Application Logs
```bash
# Get pod name
kubectl get pods

# View logs
kubectl logs <pod-name>

# Follow logs
kubectl logs -f <pod-name>
```

### View Cluster Events
```bash
kubectl get events --sort-by='.lastTimestamp'
```

---

## 🔐 Security Best Practices

1. **Never commit secrets** - Use GitHub Secrets
2. **Use IAM roles** - Instead of access keys when possible
3. **Enable MFA** - On AWS root account
4. **Scan for vulnerabilities** - Trivy & Gitleaks in pipeline
5. **Use private subnets** - For production workloads
6. **Enable encryption** - For EBS volumes and secrets
7. **Regular updates** - Keep dependencies updated

---

## 📚 Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Docker Documentation](https://docs.docker.com/)

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 👤 Author

**Abhishek**
- GitHub: [@abhi002shek](https://github.com/abhi002shek)
- LinkedIn: [Connect with me](https://linkedin.com/in/yourprofile)

---

## 🙏 Acknowledgments

- AWS for EKS service
- GitHub for Actions platform
- HashiCorp for Terraform
- Open source community

---

## 📞 Support

If you have questions or need help:
1. Check [Troubleshooting](#troubleshooting) section
2. Open an [Issue](https://github.com/abhi002shek/Github-Actions-Project/issues)
3. Review [GitHub Discussions](https://github.com/abhi002shek/Github-Actions-Project/discussions)

---

**⭐ If this project helped you, please give it a star!**

---

## 🎯 Project Highlights

- ✅ Complete CI/CD automation
- ✅ Security scanning integrated
- ✅ Infrastructure as Code
- ✅ Container orchestration
- ✅ Production-ready setup
- ✅ Cost-optimized architecture
- ✅ Comprehensive documentation

**Built with ❤️ by Abhishek**
