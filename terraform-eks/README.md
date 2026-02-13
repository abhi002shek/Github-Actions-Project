# EKS Terraform Infrastructure

This Terraform configuration creates a complete AWS EKS (Elastic Kubernetes Service) cluster with all necessary networking components.

## Architecture

- **VPC**: 10.0.0.0/16 CIDR block
- **Subnets**: 2 public subnets across different availability zones (ap-south-1a, ap-south-1b)
- **EKS Cluster**: Managed Kubernetes control plane
- **Node Group**: 3 t2.medium worker nodes
- **Security Groups**: Configured for cluster and node communication
- **IAM Roles**: Proper permissions for EKS cluster and worker nodes

## Prerequisites

- AWS CLI configured with credentials
- Terraform installed (v1.0+)
- SSH key pair created in AWS (default: jenkins)

## Usage

```bash
# Initialize Terraform
terraform init

# Review the plan
terraform plan

# Apply the configuration
terraform apply

# Destroy resources when done
terraform destroy
```

## Variables

- `ssh_key_name`: SSH key pair name for EC2 instances (default: jenkins)

## Outputs

- `cluster_id`: EKS cluster ID
- `node_group_id`: Node group ID
- `vpc_id`: VPC ID
- `subnet_ids`: List of subnet IDs

## Author

Abhishek - DevOps Engineer
