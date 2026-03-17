# CI/CD Pipeline — Java Spring Boot to AWS EKS

![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?logo=github-actions&logoColor=white)
![AWS EKS](https://img.shields.io/badge/AWS-EKS-FF9900?logo=amazon-aws&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?logo=terraform&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containerization-2496ED?logo=docker&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-Quality_Gate-4E9BCD?logo=sonarqube&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-Security_Scan-1904DA)

---

## Overview

A fully automated, security-hardened CI/CD pipeline that takes a Java Spring Boot banking application from commit to production on AWS EKS — with zero manual steps.

Pipeline enforces quality and security gates at every stage: static code analysis via SonarQube, filesystem vulnerability scanning via Trivy, and secret detection via Gitleaks — all before a single line ships to Kubernetes.

**Key engineering decisions:**
- Self-hosted GitHub Actions runner on EC2 — full control over build environment and no shared runner limitations
- Immutable Docker image tags (SHA-based) — no `latest` tag, every build is traceable
- Security gates that block the pipeline on critical findings — security is not optional
- EKS infrastructure provisioned entirely via Terraform — zero ClickOps

---

## Architecture

```
Developer Push
      │
      ▼
┌─────────────────┐
│   GitHub         │  Webhook trigger on push to main
└────────┬────────┘
         │
┌────────▼────────────────────────────────┐
│   GitHub Actions (Self-Hosted EC2 Runner) │
│                                          │
│  ┌──────────┐  Job 1: Compile            │
│  │  Maven   │  Java 17, Spring Boot      │
│  └──────────┘                            │
│  ┌──────────┐  Job 2: Security Check     │
│  │  Trivy   │  Filesystem CVE scan        │
│  │ Gitleaks │  Secret detection           │
│  └──────────┘                            │
│  ┌──────────┐  Job 3: Test               │
│  │  Maven   │  Unit + integration tests  │
│  └──────────┘                            │
│  ┌──────────┐  Job 4: Build + QA         │
│  │SonarQube │  Static analysis           │
│  │          │  Quality gate enforcement  │
│  └──────────┘                            │
│  ┌──────────┐  Job 5: Docker Build       │
│  │  Docker  │  Multi-stage image build   │
│  │  Hub/ECR │  Push with SHA tag         │
│  └──────────┘                            │
│  ┌──────────┐  Job 6: Deploy             │
│  │  Helm /  │  Rolling deploy to EKS     │
│  │ kubectl  │  Health verification       │
│  └──────────┘                            │
└─────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│         AWS EKS Cluster                │
│  ┌──────────────────────────────────┐  │
│  │  Spring Boot Banking App         │  │
│  │  3 nodes · ALB · Auto Scaling    │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
```

---

## Tech Stack

| Category | Tools |
|----------|-------|
| **CI/CD** | GitHub Actions, Self-Hosted Runner |
| **Security** | Trivy (CVE scanning), Gitleaks (secret detection), SonarQube (SAST) |
| **Build** | Java 17, Spring Boot, Maven |
| **Containers** | Docker (multi-stage builds), Docker Hub |
| **Orchestration** | Kubernetes / AWS EKS, Helm, kubectl |
| **IaC** | Terraform (VPC, EKS, EC2, IAM) |
| **Cloud** | AWS — EKS, EC2, VPC, IAM, ALB, EBS |
| **Database** | MySQL with persistent volumes |

---

## Pipeline Stages — Detail

### Job 1: Compile
- Checkout source code
- Set up JDK 17
- Compile with Maven — fails fast on compilation errors before any compute is wasted

### Job 2: Security Check
- **Trivy** filesystem scan — detects CVEs in dependencies and OS packages
- **Gitleaks** — scans entire commit history for accidentally committed secrets
- Pipeline blocks on critical/high findings — security is a hard gate, not advisory

### Job 3: Test
- Maven unit and integration test suite
- Test reports generated and uploaded as artifacts

### Job 4: Build + SonarQube Analysis
- Full Maven build producing deployable JAR
- SonarQube static code analysis — code smells, bugs, security hotspots
- Quality gate enforcement — pipeline fails if gate is not passed
- JAR artifact uploaded for next stage

### Job 5: Docker Build + Push
- Download JAR artifact from previous stage
- Multi-stage Docker build — minimal final image
- Push to registry with SHA-based immutable tag — no `latest` drift
- Image tag propagated to deployment stage

### Job 6: Deploy to EKS
- Configure AWS credentials via GitHub Secrets
- Update kubeconfig for target cluster
- Rolling deployment to Kubernetes — `maxUnavailable: 0`, `maxSurge: 1`
- Health check verification post-deploy

---

## Repository Structure

```
Github-Actions-Project/
├── .github/
│   └── workflows/
│       └── cicd.yaml              # Full CI/CD pipeline definition
├── terraform-eks/                 # EKS infrastructure as code
│   ├── main.tf                    # VPC, EKS cluster, node groups
│   ├── variables.tf               # Input variables
│   ├── output.tf                  # Cluster endpoint, kubeconfig outputs
│   └── README.md                  # Terraform usage documentation
├── src/                           # Java Spring Boot application source
├── Dockerfile                     # Multi-stage container image definition
├── ds.yml                         # Kubernetes deployment + service manifest
├── pom.xml                        # Maven build configuration
└── sonar-project.properties       # SonarQube project configuration
```

---

## Infrastructure — Terraform

EKS cluster and all supporting AWS infrastructure provisioned via Terraform.

**What's provisioned:**
- Custom VPC with public/private subnets
- EKS cluster with managed node group (3x EC2 nodes)
- IAM roles and policies scoped to least-privilege
- Application Load Balancer for external traffic
- Security groups with controlled ingress/egress

```bash
cd terraform-eks

terraform init
terraform plan
terraform apply -auto-approve

# Configure kubectl
aws eks update-kubeconfig --name <cluster-name> --region ap-south-1

# Verify
kubectl get nodes
```

---

## Setup

### Prerequisites
- AWS account with IAM credentials configured
- GitHub repository with Actions enabled
- Docker Hub account (or ECR)
- SonarQube instance (self-hosted or SonarCloud)

### GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS IAM access key |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM secret key |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `SONAR_TOKEN` | SonarQube authentication token |

### GitHub Variables Required

| Variable | Description |
|----------|-------------|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `SONAR_HOST_URL` | SonarQube server URL |

### Self-Hosted Runner Setup

```bash
# Launch EC2 instance (Ubuntu 22.04, t2.medium)
# SSH into instance, then:

# Install Docker
sudo apt update && sudo apt install -y docker.io
sudo usermod -aG docker ubuntu
sudo systemctl restart docker

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Register runner (get token from repo Settings → Actions → Runners)
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz
./config.sh --url https://github.com/<your-username>/Github-Actions-Project --token <YOUR_TOKEN>
./run.sh
```

---

## Deploy

### Trigger Pipeline
```bash
git add .
git commit -m "feat: your change description"
git push origin main
# Pipeline triggers automatically via webhook
```

### Manual Deployment
```bash
aws eks update-kubeconfig --name <cluster-name> --region ap-south-1
kubectl apply -f ds.yml
kubectl get deployments
kubectl get pods
kubectl get svc bankapp-service
```

### Get Application URL
```bash
kubectl get svc bankapp-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

---

## Troubleshooting

**kubectl not found on runner**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client
```

**Docker permission denied on runner**
```bash
sudo usermod -aG docker ubuntu
# Restart runner process
```

**Pods not starting**
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get events --sort-by='.lastTimestamp'
```

**EKS cluster not accessible**
```bash
aws eks update-kubeconfig --name <cluster-name> --region ap-south-1
kubectl get nodes
```

**SonarQube quality gate failing**
```bash
# Check analysis results in SonarQube UI
# Review sonar-project.properties for correct project key
```

---

## Cost Awareness

| Resource | Type | Approx. Cost |
|----------|------|-------------|
| EKS Control Plane | Managed | ~$73/month |
| EC2 Worker Nodes | 3x t2.medium | ~$108/month |
| ALB | Load Balancer | ~$18/month |
| EC2 Runner | t2.medium | ~$30/month |

**Destroy when not in use:**
```bash
kubectl delete -f ds.yml
cd terraform-eks && terraform destroy -auto-approve
aws ec2 terminate-instances --instance-ids <runner-instance-id>
```

---

## Security Practices

- All secrets stored in GitHub Secrets — never in code or env files
- IAM credentials scoped to minimum required permissions
- Trivy blocks pipeline on critical CVEs before image is pushed
- Gitleaks scans entire commit history on every run
- SonarQube quality gate enforced — code doesn't ship with critical bugs
- Private subnets for worker nodes — only ALB is internet-facing
- EBS volumes encrypted at rest

---

## Author

**Abhishek Singh** — DevOps Engineer · SRE · Platform Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat&logo=linkedin&logoColor=white)](https://linkedin.com/in/abhishek-singh-2b96961a0)
[![Portfolio](https://img.shields.io/badge/Portfolio-000000?style=flat&logo=vercel&logoColor=white)](https://portfolio-abhi002sheks-projects.vercel.app)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)](https://github.com/abhi002shek)

---

*📍 Hyderabad, India · Open to DevOps / SRE / Cloud / Platform Engineer roles*
