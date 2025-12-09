# README — Full flow: Infra → Network → Security → DB → Deploy Web & App

This README shows a complete, opinionated, repeatable flow to go from **creating the VPC (with private web/app/db subnets across 2 AZs)** to **deploying the web and app tiers** and **creating / initializing the DB**.
I provide two deployment options for the compute layer — **recommended: Containers on ECS Fargate** and **alternative: EC2 Auto Scaling Groups (ASG) with user-data** — plus CI/CD examples (GitHub Actions), Terraform snippets for infra, and CLI commands you can run now.

> Assumptions / prerequisites
>
> * AWS CLI configured with appropriate credentials and `us-east-1` default region.
> * `terraform` (if using Terraform), `docker`, `aws` CLI, and GitHub account for Actions.
> * You have permissions to create VPCs, EC2, RDS, ECR, ECS, ALB, IAM, and Route tables.
> * Replace placeholder IDs and names (`vpc-xxxx`, `subnet-xxxx`, `ACCOUNT_ID`, `REGION`, `YOUR_DOMAIN`, etc.) with real values.

---

# 1. Architecture summary

* VPC: `10.0.0.0/16`
* Public subnets: 2 (one per AZ) — used for NAT Gateways and ALB (ALB can be public)
* Private subnets: 3 per AZ (web, app, db) → **total 6 private subnets** (web/app/db × 2 AZs)
* NAT Gateway: 1 per AZ (for high availability)
* ALB (public) → routes to web tasks/instances in web-private subnets
* Internal ALB (optional) → routes from web → app
* DB: Amazon RDS (MySQL/Postgres) in DB private subnets, Multi-AZ
* Deployment: ECS Fargate (recommended) or EC2 ASG (alternative)
* CI: GitHub Actions builds image → push to ECR → update ECS service (or update ASG/LaunchTemplate)

---

# 2. Subnet & VPC CIDR plan (recommended)

| Name          | CIDR         | AZ         |
| ------------- | ------------ | ---------- |
| public-1      | 10.0.0.0/24  | us-east-1a |
| public-2      | 10.0.1.0/24  | us-east-1b |
| web-private-1 | 10.0.10.0/24 | us-east-1a |
| web-private-2 | 10.0.11.0/24 | us-east-1b |
| app-private-1 | 10.0.20.0/24 | us-east-1a |
| app-private-2 | 10.0.21.0/24 | us-east-1b |
| db-private-1  | 10.0.30.0/24 | us-east-1a |
| db-private-2  | 10.0.31.0/24 | us-east-1b |

---

# 3. Create infra — two ways

## Option A — Terraform (recommended for infra-as-code)

> Minimal Terraform module structure (high-level). Save as `main.tf`, `variables.tf` and `outputs.tf`. This is a skeleton — adapt for your naming/requirements.

**main.tf (skeleton)**

```hcl
provider "aws" {
  region = var.region
}

resource "aws_vpc" "this" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "three-tier-vpc" }
}

# public subnets
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.this.id
  cidr_block              = element(["10.0.0.0/24","10.0.1.0/24"], count.index)
  availability_zone       = element([var.az1, var.az2], count.index)
  map_public_ip_on_launch = true
  tags = { Name = "public-${count.index+1}" }
}

# private web subnets (2)
resource "aws_subnet" "web_private" {
  count = 2
  vpc_id = aws_vpc.this.id
  cidr_block = element(["10.0.10.0/24","10.0.11.0/24"], count.index)
  availability_zone = element([var.az1, var.az2], count.index)
  tags = { Name = "web-private-${count.index+1}" }
}

# private app subnets (2)
# private db subnets (2)
# Create IGW, NAT Gateways, Route Tables (one private RT per AZ pointing to NAT), security groups, etc.
```

**variables.tf**

```hcl
variable "region" { default = "us-east-1" }
variable "az1" { default = "us-east-1a" }
variable "az2" { default = "us-east-1b" }
```

**Usage**

```bash
terraform init
terraform plan -out plan.tf
terraform apply plan.tf
```

> I can expand to full Terraform module (with NAT, RTs, route associations, ALB, security groups, RDS) if you want — say “Terraform full”.

---

## Option B — AWS CLI (quick manual commands)

Below are the essential commands (run sequentially). Replace placeholders.

**Create VPC**

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value=three-tier-vpc
```

**Create public subnets**

```bash
PUB1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.0.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
PUB2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1b --query 'Subnet.SubnetId' --output text)
```

**Create private subnets (web/app/db × 2 AZs)**

```bash
WEB1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.10.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
WEB2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 --availability-zone us-east-1b --query 'Subnet.SubnetId' --output text)
APP1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.20.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
APP2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.21.0/24 --availability-zone us-east-1b --query 'Subnet.SubnetId' --output text)
DB1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.30.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
DB2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.31.0/24 --availability-zone us-east-1b --query 'Subnet.SubnetId' --output text)
```

**Create Internet Gateway and attach**

```bash
IGW=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW
```

**Create and allocate Elastic IPs for NATs and create NATs (1 per AZ)**

```bash
EIP1=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
EIP2=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

NAT1=$(aws ec2 create-nat-gateway --subnet-id $PUB1 --allocation-id $EIP1 --query 'NatGateway.NatGatewayId' --output text)
NAT2=$(aws ec2 create-nat-gateway --subnet-id $PUB2 --allocation-id $EIP2 --query 'NatGateway.NatGatewayId' --output text)
```

**Create route tables and associate** — public RT -> IGW; private RTs -> NATs (per-AZ). (Omitted for brevity; ask if needed.)

---

# 4. Security: Security Groups & IAM

**Frontend SG** (allow HTTP/HTTPS from anywhere)

```bash
FRONT_SG=$(aws ec2 create-security-group --group-name FrontendSG --description "Frontend SG" --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $FRONT_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $FRONT_SG --protocol tcp --port 443 --cidr 0.0.0.0/0
```

**Backend SG** (allow traffic from FrontendSG only on app port)

```bash
BACK_SG=$(aws ec2 create-security-group --group-name BackendSG --description "App Backend SG" --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $BACK_SG --protocol tcp --port 8080 --source-group $FRONT_SG
```

**DB SG** (allow only from BackendSG to DB port)

```bash
DB_SG=$(aws ec2 create-security-group --group-name DbSG --description "DB SG" --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 3306 --source-group $BACK_SG
```

**IAM role for tasks/instances**

* For ECS tasks, create an IAM role with `AmazonECSTaskExecutionRolePolicy` and an inline policy to access S3, Secrets Manager, or parameter store (if needed).
* For EC2 instances, create an instance profile with policies to fetch artifacts from S3/SSM.

---

# 5. Database (RDS) creation & initialization

**Create RDS (MySQL example)**

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name three-tier-db-subnet-group \
  --db-subnet-group-description "DB subnets" \
  --subnet-ids $DB1 $DB2

aws rds create-db-instance \
  --db-instance-identifier prod-mysql \
  --allocated-storage 20 \
  --db-instance-class db.t3.medium \
  --engine mysql \
  --master-username admin \
  --master-user-password 'ChangeThisPassword!' \
  --multi-az \
  --vpc-security-group-ids $DB_SG \
  --db-subnet-group-name three-tier-db-subnet-group \
  --backup-retention-period 7
```

**Initialize DB (from your CI or local laptop)**

```bash
mysql -h <RDS_ENDPOINT> -u admin -p
# then run:
CREATE DATABASE javaapp;
USE javaapp;
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(50) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  email VARCHAR(100) NOT NULL UNIQUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_username ON users(username);
CREATE INDEX idx_email ON users(email);
```

For migrations use: Flyway, Liquibase, or a migration script in CI.

---

# 6. Deploy compute — two options

## Option A — Recommended: ECS Fargate (containers)

**Why**: no EC2 management, autoscaling, easy rolling updates, integrates with ALB.

### Steps

1. **Create ECR repo**

```bash
aws ecr create-repository --repository-name your-web-app --region us-east-1
aws ecr create-repository --repository-name your-app-service --region us-east-1
```

2. **Build & push image (locally or CI)**

```bash
# build
docker build -t your-web-app:latest ./web
docker tag your-web-app:latest $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/your-web-app:latest

# authenticate and push
aws ecr get-login-password | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
docker push $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/your-web-app:latest
```

3. **Create ECS Cluster and Task Definitions**
   Use `aws ecs register-task-definition` or Terraform/CloudFormation. Task definition must use subnets (private) and security groups; add `awsvpc` networking mode. Use Fargate launch type.

4. **Create Application Load Balancer (public)**

* ALB target group for web service
* Listener on 80/443 → target group with web tasks
* Optional internal ALB for app tasks (private)

5. **Create ECS Service**

* Service pointing to task definition with desired count and autoscaling policies. Service will register tasks with target group; ALB health checks ensure rolling updates.

6. **CI/CD (GitHub Actions example)**
   See section "CI/CD example — ECS".

---

## Option B — Alternative: EC2 ASG + Launch Template + ALB

**Why**: Useful if you need full-control OS-level operations or legacy apps.

### Steps (summary)

1. Create Launch Template with user-data that installs the app (pulls artifact from S3 or runs Docker).
2. Create Auto Scaling Group across `web-private-1` & `web-private-2` with target group to the ALB.
3. Create internal ALB and backend ASG for app tier.
4. RDS in DB private subnets.

**Example user-data for web EC2 (pull from S3)**
`web_userdata.sh`

```bash
#!/bin/bash
yum update -y
yum install -y nginx awscli
aws s3 cp s3://my-app-bucket/web/frontend.tar.gz /tmp/frontend.tar.gz
cd /usr/share/nginx/html && tar xzf /tmp/frontend.tar.gz
systemctl enable nginx && systemctl start nginx
```

**Launch instance**

```bash
aws ec2 run-instances \
  --image-id ami-xxxx \
  --instance-type t3.micro \
  --subnet-id $WEB1 \
  --security-group-ids $FRONT_SG \
  --user-data file://web_userdata.sh
```

**Rolling updates (best practice)**

* Bake AMI with code (Packer) and update ASG LaunchTemplate version.
* Or use SSM Run Command to pull a new artifact on all instances.

---

# 7. CI/CD Examples

## GitHub Actions — build/push & update ECS service (simplified)

Create `.github/workflows/deploy-ecs.yml`:

```yaml
name: CI/CD → ECR → ECS

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-region: us-east-1
        role-to-assume: arn:aws:iam::${{ secrets.ACCOUNT_ID }}:role/GitHubActionsDeployRole
        role-duration-seconds: 900
        role-session-name: gha-deploy

    - name: Log in to ECR
      run: aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com

    - name: Build, tag, push
      run: |
        docker build -t web-app:latest ./web
        docker tag web-app:latest ${{ secrets.ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/web-app:latest
        docker push ${{ secrets.ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/web-app:latest

    - name: Update ECS service
      run: |
        aws ecs update-service --cluster your-cluster --service web-service --force-new-deployment
```

> This forces ECS to pull the new image (if tag `latest` is updated). Better: use task definition revisions with immutable tags.

## GitHub Actions — push artifact to S3 & trigger ASG update (alternative)

* Build artifact -> upload to S3
* Update Launch Template version or use SSM to run `aws s3 cp` on instances
* Or use CodeDeploy for in-place deployments

---

# 8. Bootstrapping & secrets

* Store DB credentials & app secrets in **AWS Secrets Manager** or **SSM Parameter Store (secure)**. Grant ECS task role or EC2 instance role permission to read secrets.
* Example IAM policy for secrets access (attach to task role):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["secretsmanager:GetSecretValue", "ssm:GetParameter"],
    "Resource": ["arn:aws:secretsmanager:us-east-1:ACCOUNT_ID:secret:myapp/*","arn:aws:ssm:us-east-1:ACCOUNT_ID:parameter/myapp/*"]
  }]
}
```

---

# 9. DB migrations

* Use a DB migration tool (Flyway / Liquibase / Alembic / Django migrations). Run migrations from CI after deployment or as part of a release pipeline.
* Example (CI step):

```bash
mysql -h $RDS_ENDPOINT -u $DB_USER -p$DB_PASS < db/migrations/001_create_tables.sql
```

---

# 10. Monitoring & backups

* **RDS**: automated backups & snapshots configured in RDS creation.
* **App & Web**: CloudWatch logs (ECS LogDriver or EC2 CloudWatch Agent).
* **Alarms**: CPU, memory, unhealthy targets in ALB, high DB connections.
* **Logging**: centralize app logs to CloudWatch Logs + set retention.

---

# 11. Security & best practices checklist

* No DB exposed to internet; DB SG allows only app SG.
* ALB is public; app tasks/instances in private subnets.
* Use HTTPS (TLS) on ALB; manage certs via AWS Certificate Manager (ACM).
* Use IAM roles for EC2/ECS — never embed AWS credentials.
* Secrets in Secrets Manager. Rotate DB passwords.
* Use least-privilege IAM policies.
* Use multi-AZ NAT and RDS Multi-AZ for HA.

---

# 12. Quick troubleshooting tips

* Instances not starting: check subnet route tables and NAT.
* Tasks in PENDING: check ENI availability and subnets.
* ALB shows `draining`/`unhealthy`: verify target group health checks and security groups.
* ECS failing to pull image: check ECR permissions, repo policy, and image tag.

---

# 13. Example quick checklist to run now

1. Create VPC + subnets + IGW + NATs + route tables (Terraform or CLI).
2. Create security groups and IAM roles.
3. Create RDS in DB subnet group and wait for `available`.
4. Create ECR repos. Build and push Docker images.
5. Create ECS cluster, task definitions, ALB, and services (or create Launch Template & ASG).
6. Configure CI/CD to push images and run `aws ecs update-service` (or update ASG/LaunchTemplate).
7. Run DB migrations.
8. Validate: open ALB DNS, hit web endpoints, check logs and metrics.

---

# 14. Want me to generate now?

I can produce:

* ✅ Full **Terraform** module that creates VPC, subnets, IGW, NAT, route tables, ALB, ECS cluster (Fargate), ECR repos, and RDS (complete).
* ✅ A runnable **GitHub Actions** workflow for ECR → ECS deployment (more advanced: update task def with new image digest).
* ✅ A **Launch Template + ASG AWS CLI script** + `user-data` for EC2 option.
* ✅ Full `task-definition.json` sample for ECS Fargate.
* ✅ Example `Dockerfile` and sample app minimal code.

Which one should I generate for you now? (I can create the Terraform full module next if you want — say **“Terraform full”**.)
