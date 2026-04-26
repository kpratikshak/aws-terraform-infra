# aws-terraform-infra

To convert this into a professional GitHub repository, we need to structure the code for reusability and provide a `README.md` that guides a user from zero to deployment.

Below is the complete project structure and the source code for the core files.

### Repository Structure
```text
.
├── .gitignore
├── README.md
├── env/
│   └── dev/
│       ├── main.tf            # Main entry point
│       ├── variables.tf       # Input definitions
│       ├── terraform.tfvars   # Actual values
│       ├── providers.tf       # AWS configuration
│       └── outputs.tf         # Key resource IDs
├── modules/                   # Reusable components
│   ├── vpc/
│   ├── ecs/
│   ├── rds/
│   ├── alb/
│   └── security_groups/
```

---

# AWS Microservices Infrastructure with Terraform

This repository contains a modular, industry-standard Terraform configuration for deploying a containerized microservices application on AWS ECS Fargate.

## 🏗️ Architecture
The infrastructure follows a "Security First" approach:
- **VPC** with Public and Private Subnets.
- **Application Load Balancer (ALB)** in public subnets to route traffic.
- **ECS Fargate** services running in private subnets.
- **RDS (PostgreSQL)** isolated in a private data subnet.

## 📸 AWS Console Walkthrough

### 1. The VPC Map
After running `terraform apply`, your VPC should look like this in the AWS Console. Notice the clear separation between public (NAT/ALB) and private (App/DB) tiers.
> 

### 2. ECS Cluster & Services
You will see a single cluster hosting multiple services (Frontend, Node API, Go API).
> **[ ECS -> Clusters -> Your-Cluster-Name -> Services Tab]**

### 3. Target Groups
The ALB routes traffic based on path rules. Check your target groups to ensure health checks are passing (Green).
> **[ EC2 -> Target Groups -> Healthy Hosts View]**

## 🚀 Getting Started

### Prerequisites
- AWS CLI configured with appropriate permissions.
- Terraform (v1.0+) installed.
- Docker (to push images to ECR).

### Deployment
1. **Clone the repo:**
   ```bash
   git clone [https://github.com/partisan/terraform-ecs-microservices.git](https://github.com/YOUR_USERNAME/terraform-ecs-microservices.git)
   cd env/dev
   ```

2. **Initialize and Plan:**
   ```bash
   terraform init
   terraform plan
   ```

3. **Apply:**
   ```bash
   terraform apply -auto-approve
   ```

## 🛠️ Module Breakdown
- **VPC Module:** Sets up 2 Public and 2 Private subnets across 2 AZs.
- **Security Groups:** Implements the principle of least privilege (ALB -> ECS -> RDS).
- **RDS Module:** Provisions a managed Postgres DB with automated backups.
- **ECS Module:** Defines the Fargate cluster and task execution roles.

## 🧹 Cleanup
To avoid unwanted AWS costs:
```bash
terraform destroy
```
```

---

### 2. Root Configuration (`env/dev/main.tf`)
This file orchestrates the modules.

```hcl
provider "aws" {
  region = var.aws_region
}

module "vpc" {
  source               = "../../modules/vpc"
  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.3.0/24", "10.0.4.0/24"]
  availability_zones   = ["us-east-1a", "us-east-1b"]
}

module "security_groups" {
  source = "../../modules/security_groups"
  vpc_id = module.vpc.vpc_id
}

module "rds" {
  source             = "../../modules/rds"
  db_name            = "microservices_db"
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
  sg_id              = module.security_groups.rds_sg_id
}

module "ecs" {
  source             = "../../modules/ecs"
  cluster_name       = "microservices-cluster"
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
}
```

---

### 3. Example Child Module (`modules/vpc/main.tf`)
This shows how the networking is actually built.

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags = { Name = "microservices-vpc" }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "public-subnet-${count.index + 1}" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}
```

---

### 4. Essential `.gitignore`
To prevent leaking sensitive data or large state files.

```text
# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Crash log files
crash.log

# Variables files
*.tfvars
*.tfvars.json

# Ignore sensitive files
.env
```

