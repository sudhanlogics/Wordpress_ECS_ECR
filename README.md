# 🏗️ WordPress on AWS ECS — Production Deployment Guide

> **Stack:** WordPress + PHP 8.1-FPM + Nginx · MySQL 8.0 (ECS sidecar) · Amazon EFS · ECS Fargate · ECR · ALB + ACM (HTTPS) · GitHub Actions CI/CD
> **Region:** `ap-south-2` (Hyderabad)

---

## 🧭 Table of Contents

1. [Architecture Overview](#1--architecture-overview)
2. [Prerequisites](#2--prerequisites)
3. [Create a Custom VPC](#3--create-a-custom-vpc)
4. [Create Subnets](#4--create-subnets)
5. [Create Internet Gateway and Route Tables](#5--create-internet-gateway-and-route-tables)
6. [Create NAT Gateway](#6--create-nat-gateway)
7. [Create Security Groups](#7--create-security-groups)
8. [Launch a Bastion Host (Ubuntu EC2)](#8--launch-a-bastion-host-ubuntu-ec2)
9. [ECR — Container Registry](#9--ecr--container-registry)
10. [EFS — Persistent Storage for WordPress & MySQL](#10--efs--persistent-storage-for-wordpress--mysql)
11. [Secrets Manager](#11--secrets-manager)
12. [IAM Roles & Policies](#12--iam-roles--policies)
13. [ECS Cluster & Task Definition](#13--ecs-cluster--task-definition)
14. [ALB + HTTPS with ACM](#14--alb--https-with-acm)
15. [ECS Service](#15--ecs-service)
16. [GitHub Actions CI/CD](#16--github-actions-cicd)
17. [GitHub Secrets Reference](#17--github-secrets-reference)
18. [Post-Deployment](#18--post-deployment)
19. [Useful Commands](#19--useful-commands)
20. [Troubleshooting](#20--troubleshooting)
21. [Cleanup](#21--cleanup)

---

## 1. 🧩 Architecture Overview

```
GitHub Push (main)
       │
       ▼
GitHub Actions
  ├─ docker build
  ├─ docker push ──► ECR (ap-south-2)
  └─ ecs update-service
            │
            ▼
      Internet (HTTPS :443)
            │
            ▼
      ALB ◄── ACM Certificate
      (HTTP :80 → HTTPS :443 redirect)
            │
            ▼
   ECS Fargate Task (awsvpc) [Private Subnet]
   ┌──────────────────────────────────────────┐
   │                                          │
   │  ┌───────────────────────┐               │
   │  │   wordpress container │               │
   │  │   Nginx + PHP 8.1-FPM │ :80           │
   │  │   connects via        │               │
   │  │   127.0.0.1:3306      │               │
   │  └──────────┬────────────┘               │
   │             │ dependsOn: HEALTHY          │
   │  ┌──────────▼────────────┐               │
   │  │   mysql container     │ :3306          │
   │  │   mysql:8.0           │               │
   │  └──────────┬────────────┘               │
   │             │                            │
   │    ┌────────┴────────┐                   │
   │    ▼                 ▼                   │
   │  EFS /wp-content  EFS /mysql-data        │
   │                                          │
   └──────────────────────────────────────────┘

   Public Subnet: ALB + Bastion Host
   Private Subnet: ECS Tasks + EFS Mount Targets
```

**Why MySQL as a sidecar?**
Both containers share the same `awsvpc` network namespace inside the ECS task — WordPress reaches MySQL at `127.0.0.1:3306` with zero latency and no extra AWS service cost. MySQL data is persisted on **EFS** so it survives task restarts, redeployments, and image rebuilds.

> ⚠️ **Scaling note:** Keep `desired-count = 1`. Running two tasks would give each its own isolated MySQL instance, splitting the database. If you ever need horizontal scaling, migrating to RDS is straightforward since your data is already on EFS.

**Why EFS?**
EFS replaces both Docker named volumes from `docker-compose.yml` (`wp_data`, `db_data`). Two isolated access points are used — one for WordPress content, one for MySQL data files — each with correct Linux ownership.

---

## 2. 🧰 Prerequisites

- **AWS CLI v2** installed and configured
- **Docker** installed locally (for the initial image push)
- A registered **domain name** with DNS access
- AWS account in `ap-south-2`

```bash
# Set region and verify identity
REGION="ap-south-2"
aws configure set region $REGION
aws sts get-caller-identity
```

---

## 3. 🪐 Create a Custom VPC

```bash
REGION="ap-south-2"
VPC_CIDR="10.0.0.0/16"
VPC_NAME="wordpress-vpc"

VPC_ID=$(aws ec2 create-vpc \
  --cidr-block $VPC_CIDR \
  --region $REGION \
  --query 'Vpc.VpcId' \
  --output text)

aws ec2 create-tags \
  --resources $VPC_ID \
  --tags Key=Name,Value=$VPC_NAME

aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support

echo "✅ VPC Created: $VPC_ID"
```

---

## 4. 🌐 Create Subnets

```bash
# Public Subnets (ALB + Bastion Host)
PUB_SUBNET1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-2a \
  --query 'Subnet.SubnetId' \
  --output text)
aws ec2 create-tags --resources $PUB_SUBNET1 --tags Key=Name,Value=wp-public-a

PUB_SUBNET2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-south-2b \
  --query 'Subnet.SubnetId' \
  --output text)
aws ec2 create-tags --resources $PUB_SUBNET2 --tags Key=Name,Value=wp-public-b

# Private Subnets (ECS Tasks + EFS)
PRI_SUBNET1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.3.0/24 \
  --availability-zone ap-south-2a \
  --query 'Subnet.SubnetId' \
  --output text)
aws ec2 create-tags --resources $PRI_SUBNET1 --tags Key=Name,Value=wp-private-a

PRI_SUBNET2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.4.0/24 \
  --availability-zone ap-south-2b \
  --query 'Subnet.SubnetId' \
  --output text)
aws ec2 create-tags --resources $PRI_SUBNET2 --tags Key=Name,Value=wp-private-b

echo "✅ Public Subnets:  $PUB_SUBNET1, $PUB_SUBNET2"
echo "✅ Private Subnets: $PRI_SUBNET1, $PRI_SUBNET2"
```

---

## 5. 🌍 Create Internet Gateway and Route Tables

```bash
# Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 create-tags --resources $IGW_ID --tags Key=Name,Value=${VPC_NAME}-igw

# Public Route Table — routes internet traffic via IGW
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)
aws ec2 create-tags --resources $PUB_RT_ID --tags Key=Name,Value=wp-public-rt

aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

aws ec2 associate-route-table --route-table-id $PUB_RT_ID --subnet-id $PUB_SUBNET1
aws ec2 associate-route-table --route-table-id $PUB_RT_ID --subnet-id $PUB_SUBNET2

echo "✅ Internet Gateway: $IGW_ID"
echo "✅ Public Route Table: $PUB_RT_ID"
```

---

## 6. 🔀 Create NAT Gateway

ECS tasks run in private subnets and need outbound internet access to pull Docker images from ECR and Docker Hub (`mysql:8.0`).

```bash
# Allocate Elastic IP for NAT
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

# Create NAT Gateway in the first public subnet
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUBNET1 \
  --allocation-id $EIP_ALLOC \
  --query 'NatGateway.NatGatewayId' \
  --output text)
aws ec2 create-tags --resources $NAT_GW_ID --tags Key=Name,Value=wp-nat-gw

echo "⏳ Waiting for NAT Gateway to become available..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_ID
echo "✅ NAT Gateway: $NAT_GW_ID"

# Private Route Table — routes outbound traffic via NAT
PRI_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)
aws ec2 create-tags --resources $PRI_RT_ID --tags Key=Name,Value=wp-private-rt

aws ec2 create-route \
  --route-table-id $PRI_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_ID

aws ec2 associate-route-table --route-table-id $PRI_RT_ID --subnet-id $PRI_SUBNET1
aws ec2 associate-route-table --route-table-id $PRI_RT_ID --subnet-id $PRI_SUBNET2

echo "✅ Private Route Table: $PRI_RT_ID"
```

---

## 7. 🔒 Create Security Groups

```bash
# ALB Security Group — internet-facing HTTP/HTTPS
ALB_SG=$(aws ec2 create-security-group \
  --group-name wp-alb-sg \
  --description "WordPress ALB - HTTP and HTTPS from internet" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG \
  --ip-permissions \
  '[{"IpProtocol":"tcp","FromPort":80,"ToPort":80,"IpRanges":[{"CidrIp":"0.0.0.0/0"}]},
    {"IpProtocol":"tcp","FromPort":443,"ToPort":443,"IpRanges":[{"CidrIp":"0.0.0.0/0"}]}]'
aws ec2 create-tags --resources $ALB_SG --tags Key=Name,Value=wp-alb-sg

# ECS Task Security Group — port 80 from ALB only
ECS_SG=$(aws ec2 create-security-group \
  --group-name wp-ecs-sg \
  --description "WordPress ECS Tasks - HTTP from ALB only" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG \
  --protocol tcp --port 80 \
  --source-group $ALB_SG
aws ec2 create-tags --resources $ECS_SG --tags Key=Name,Value=wp-ecs-sg

# EFS Security Group — NFS from ECS tasks only
EFS_SG=$(aws ec2 create-security-group \
  --group-name wp-efs-sg \
  --description "WordPress EFS - NFS from ECS tasks only" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)
aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG \
  --protocol tcp --port 2049 \
  --source-group $ECS_SG
aws ec2 create-tags --resources $EFS_SG --tags Key=Name,Value=wp-efs-sg

# Bastion Host Security Group — SSH from your IP
BASTION_SG=$(aws ec2 create-security-group \
  --group-name wp-bastion-sg \
  --description "Bastion Host - SSH access" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)
aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG \
  --protocol tcp --port 22 \
  --cidr 0.0.0.0/0        # ⚠️ Restrict to your IP in production: e.g. 203.0.113.0/32
aws ec2 create-tags --resources $BASTION_SG --tags Key=Name,Value=wp-bastion-sg

echo "✅ ALB SG:     $ALB_SG"
echo "✅ ECS SG:     $ECS_SG"
echo "✅ EFS SG:     $EFS_SG"
echo "✅ Bastion SG: $BASTION_SG"
```

> MySQL port 3306 is **not** in any security group. It is only reachable within the ECS task via `127.0.0.1` — never exposed to the network.

---

## 8. 💻 Launch a Bastion Host (Ubuntu EC2)

The bastion host lives in the public subnet and provides SSH access into private resources for debugging.

```bash
KEY_NAME="wordpress-bastion-key"

# Create Key Pair and save locally
aws ec2 create-key-pair \
  --key-name $KEY_NAME \
  --query 'KeyMaterial' \
  --output text > ${KEY_NAME}.pem
chmod 400 ${KEY_NAME}.pem

# Get latest Ubuntu 24.04 LTS AMI (Canonical owner ID)
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
  --query 'Images | sort_by(@, &CreationDate)[-1].ImageId' \
  --output text \
  --region $REGION)

# Launch Bastion Host in public subnet
BASTION_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name $KEY_NAME \
  --security-group-ids $BASTION_SG \
  --subnet-id $PUB_SUBNET1 \
  --associate-public-ip-address \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=WordPress-Bastion}]" \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "⏳ Waiting for Bastion Host..."
aws ec2 wait instance-running --instance-ids $BASTION_ID

BASTION_IP=$(aws ec2 describe-instances \
  --instance-ids $BASTION_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "✅ Bastion Host: $BASTION_ID"
echo "✅ SSH: ssh -i ${KEY_NAME}.pem ubuntu@${BASTION_IP}"
```

---

## 9. 📦 ECR — Container Registry

The WordPress image (Nginx + PHP-FPM) is built from the repo's `Dockerfile` and pushed to ECR. The MySQL container pulls `mysql:8.0` directly from Docker Hub — no ECR push needed for it.

### 9.1 Create Repository

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_URI="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/docker-wordpress"

aws ecr create-repository \
  --repository-name docker-wordpress \
  --region $REGION \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

echo "✅ ECR URI: $ECR_URI"
```

### 9.2 Build & Push Initial Image

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin \
  "${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"

# Clone repo and build
git clone https://github.com/sudhanlogics/docker-wordpress.git
cd docker-wordpress

docker build -t docker-wordpress .
docker tag docker-wordpress:latest ${ECR_URI}:latest
docker push ${ECR_URI}:latest

echo "✅ Image pushed: ${ECR_URI}:latest"
```

### 9.3 Lifecycle Policy (keep last 10 images)

```bash
aws ecr put-lifecycle-policy \
  --repository-name docker-wordpress \
  --region $REGION \
  --lifecycle-policy-text '{
    "rules": [{
      "rulePriority": 1,
      "description": "Keep last 10 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    }]
  }'
```

---

## 10. 🗂️ EFS — Persistent Storage for WordPress & MySQL

EFS provides two isolated access points that replace the `wp_data` and `db_data` Docker named volumes from `docker-compose.yml`.

| Access Point | Mounted At (in container) | UID/GID | Purpose |
|---|---|---|---|
| `EFS_AP_WP` | `/var/www/html/wp-content` | 33 (www-data) | Themes, plugins, uploads |
| `EFS_AP_DB` | `/var/lib/mysql` | 999 (mysql) | MySQL database files |

### 10.1 Create EFS Filesystem

```bash
EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=wordpress-efs \
  --region $REGION \
  --query 'FileSystemId' \
  --output text)

echo "✅ EFS: $EFS_ID"
```

### 10.2 Create Mount Targets (one per private subnet)

```bash
aws efs create-mount-target \
  --file-system-id $EFS_ID \
  --subnet-id $PRI_SUBNET1 \
  --security-groups $EFS_SG \
  --region $REGION

aws efs create-mount-target \
  --file-system-id $EFS_ID \
  --subnet-id $PRI_SUBNET2 \
  --security-groups $EFS_SG \
  --region $REGION

echo "⏳ Waiting for EFS mount targets..."
sleep 30
echo "✅ EFS mount targets ready"
```

### 10.3 Create Access Points

```bash
# WordPress wp-content (www-data = UID/GID 33)
EFS_AP_WP=$(aws efs create-access-point \
  --file-system-id $EFS_ID \
  --posix-user Uid=33,Gid=33 \
  --root-directory "Path=/wp-content,CreationInfo={OwnerUid=33,OwnerGid=33,Permissions=755}" \
  --tags Key=Name,Value=wp-content-ap \
  --region $REGION \
  --query 'AccessPointId' \
  --output text)

# MySQL data (mysql = UID/GID 999)
EFS_AP_DB=$(aws efs create-access-point \
  --file-system-id $EFS_ID \
  --posix-user Uid=999,Gid=999 \
  --root-directory "Path=/mysql-data,CreationInfo={OwnerUid=999,OwnerGid=999,Permissions=700}" \
  --tags Key=Name,Value=mysql-data-ap \
  --region $REGION \
  --query 'AccessPointId' \
  --output text)

echo "✅ WordPress EFS Access Point: $EFS_AP_WP"
echo "✅ MySQL EFS Access Point:     $EFS_AP_DB"
```

---

## 11. 🔑 Secrets Manager

All credentials are stored in Secrets Manager and injected into containers as environment variables at task startup. Because WordPress and MySQL share the same task network namespace, `MYSQL_HOST` is `127.0.0.1`.

```bash
aws secretsmanager create-secret \
  --name wordpress/config \
  --region $REGION \
  --secret-string "{
    \"MYSQL_HOST\": \"127.0.0.1\",
    \"MYSQL_DATABASE\": \"wordpress\",
    \"MYSQL_USER\": \"dbadmin\",
    \"MYSQL_PASSWORD\": \"YourStrongPassword123!\",
    \"MYSQL_ROOT_PASSWORD\": \"YourStrongRootPassword123!\",
    \"WP_TABLE_PREFIX\": \"wp_\",
    \"WP_DEBUG\": \"false\",
    \"DOMAIN\": \"your-domain.com\"
  }"

SECRET_ARN=$(aws secretsmanager describe-secret \
  --secret-id wordpress/config \
  --region $REGION \
  --query 'ARN' \
  --output text)

echo "✅ Secret ARN: $SECRET_ARN"
```

To update a value later:

```bash
aws secretsmanager update-secret \
  --secret-id wordpress/config \
  --region $REGION \
  --secret-string '{"WP_DEBUG":"true", ...}'
```

---

## 12. 🛡️ IAM Roles & Policies

### 12.1 ECS Task Execution Role

Allows ECS to pull images from ECR and read secrets at container startup.

```bash
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "ecs-tasks.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

aws iam put-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-name SecretsManagerAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue", "kms:Decrypt"],
      "Resource": "*"
    }]
  }'

echo "✅ ecsTaskExecutionRole ready"
```

### 12.2 ECS Task Role

Assumed by the running containers — grants EFS access and SSM for ECS Exec (exec into container).

```bash
aws iam create-role \
  --role-name ecsWordpressTaskRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "ecs-tasks.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam put-role-policy \
  --role-name ecsWordpressTaskRole \
  --policy-name EFSAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:ClientMount",
        "elasticfilesystem:ClientWrite",
        "elasticfilesystem:DescribeMountTargets",
        "elasticfilesystem:DescribeFileSystems"
      ],
      "Resource": "*"
    }]
  }'

# SSM required for ECS Exec (debugging inside containers)
aws iam attach-role-policy \
  --role-name ecsWordpressTaskRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

echo "✅ ecsWordpressTaskRole ready"
```

### 12.3 GitHub Actions Deployment Role (OIDC — no long-lived keys)

```bash
# Register GitHub OIDC provider — only once per AWS account
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# ⚠️ Replace YOUR_GITHUB_ORG with your GitHub username or org
aws iam create-role \
  --role-name GitHubActionsWordPressRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::'"${ACCOUNT_ID}"':oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_ORG/docker-wordpress:ref:refs/heads/main"
        }
      }
    }]
  }'

aws iam put-role-policy \
  --role-name GitHubActionsWordPressRole \
  --policy-name DeployPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload",
          "ecr:PutImage"
        ],
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": [
          "ecs:DescribeServices",
          "ecs:UpdateService",
          "ecs:RegisterTaskDefinition",
          "ecs:DescribeTaskDefinition"
        ],
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": "iam:PassRole",
        "Resource": [
          "arn:aws:iam::*:role/ecsTaskExecutionRole",
          "arn:aws:iam::*:role/ecsWordpressTaskRole"
        ]
      }
    ]
  }'

echo "✅ GitHubActionsWordPressRole ready"
```

---

## 13. ⚙️ ECS Cluster & Task Definition

### 13.1 Create ECS Cluster

```bash
aws ecs create-cluster \
  --cluster-name wordpress-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1 \
  --region $REGION

# CloudWatch Log Group
aws logs create-log-group --log-group-name /ecs/wordpress --region $REGION
aws logs put-retention-policy \
  --log-group-name /ecs/wordpress \
  --retention-in-days 30 \
  --region $REGION

echo "✅ ECS Cluster: wordpress-cluster"
```

### 13.2 Task Definition

Save this as `.aws/task-definition.json` in your repository root.

Replace these four placeholders before committing:

| Placeholder | Replace with |
|---|---|
| `ACCOUNT_ID` | Your 12-digit AWS account ID |
| `EFS_ID` | From Step 10.1 |
| `EFS_AP_WP` | From Step 10.3 (WordPress access point) |
| `EFS_AP_DB` | From Step 10.3 (MySQL access point) |

```json
{
  "family": "wordpress-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsWordpressTaskRole",
  "volumes": [
    {
      "name": "efs-wp-content",
      "efsVolumeConfiguration": {
        "fileSystemId": "EFS_ID",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "EFS_AP_WP",
          "iam": "ENABLED"
        }
      }
    },
    {
      "name": "efs-mysql-data",
      "efsVolumeConfiguration": {
        "fileSystemId": "EFS_ID",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "EFS_AP_DB",
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "mysql",
      "image": "mysql:8.0",
      "essential": true,
      "portMappings": [],
      "mountPoints": [
        {
          "sourceVolume": "efs-mysql-data",
          "containerPath": "/var/lib/mysql",
          "readOnly": false
        }
      ],
      "secrets": [
        { "name": "MYSQL_ROOT_PASSWORD", "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:MYSQL_ROOT_PASSWORD::" },
        { "name": "MYSQL_DATABASE",      "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:MYSQL_DATABASE::" },
        { "name": "MYSQL_USER",          "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:MYSQL_USER::" },
        { "name": "MYSQL_PASSWORD",      "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:MYSQL_PASSWORD::" }
      ],
      "healthCheck": {
        "command": ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1"],
        "interval": 10,
        "timeout": 5,
        "retries": 5,
        "startPeriod": 30
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/wordpress",
          "awslogs-region": "ap-south-2",
          "awslogs-stream-prefix": "mysql"
        }
      }
    },
    {
      "name": "wordpress",
      "image": "ACCOUNT_ID.dkr.ecr.ap-south-2.amazonaws.com/docker-wordpress:IMAGE_TAG",
      "essential": true,
      "portMappings": [
        { "containerPort": 80, "protocol": "tcp" }
      ],
      "mountPoints": [
        {
          "sourceVolume": "efs-wp-content",
          "containerPath": "/var/www/html/wp-content",
          "readOnly": false
        }
      ],
      "dependsOn": [
        { "containerName": "mysql", "condition": "HEALTHY" }
      ],
      "secrets": [
        { "name": "MYSQL_HOST",      "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:MYSQL_HOST::" },
        { "name": "MYSQL_DATABASE",  "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:MYSQL_DATABASE::" },
        { "name": "MYSQL_USER",      "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:MYSQL_USER::" },
        { "name": "MYSQL_PASSWORD",  "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:MYSQL_PASSWORD::" },
        { "name": "DOMAIN",          "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:DOMAIN::" },
        { "name": "WP_DEBUG",        "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:WP_DEBUG::" },
        { "name": "WP_TABLE_PREFIX", "valueFrom": "arn:aws:secretsmanager:ap-south-2:ACCOUNT_ID:secret:wordpress/config:WP_TABLE_PREFIX::" }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost/ || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 120
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/wordpress",
          "awslogs-region": "ap-south-2",
          "awslogs-stream-prefix": "wordpress"
        }
      }
    }
  ]
}
```

Register the task definition:
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

sed -e "s/ACCOUNT_ID/$ACCOUNT_ID/g" \
    -e "s|EFS_ID|$EFS_ID|g" \
    -e "s|EFS_AP_WP|$EFS_AP_WP|g" \
    -e "s|EFS_AP_DB|$EFS_AP_DB|g" \
    -e "s|IMAGE_TAG|latest|g" \
    .aws/task-definition.json > .aws/task-definition-rendered.json

aws ecs register-task-definition \
  --cli-input-json file://.aws/task-definition-rendered.json \
  --region $REGION

echo "✅ Task definition registered: wordpress-task"
```

---

## 14. 🔐 ALB + HTTPS with ACM

### 14.1 Request ACM Certificate

```bash
CERT_ARN=$(aws acm request-certificate \
  --domain-name "your-domain.com" \
  --subject-alternative-names "www.your-domain.com" \
  --validation-method DNS \
  --region $REGION \
  --query 'CertificateArn' \
  --output text)

echo "✅ Certificate ARN: $CERT_ARN"
```

Get the DNS validation CNAME records to add to your DNS provider:

```bash
aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --region $REGION \
  --query 'Certificate.DomainValidationOptions[].{Name:ResourceRecord.Name,Value:ResourceRecord.Value}'
```

Add the CNAME records to your DNS, then wait for validation:

```bash
aws acm wait certificate-validated \
  --certificate-arn $CERT_ARN \
  --region $REGION

echo "✅ Certificate ISSUED"
```

### 14.2 Create ALB

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name wordpress-alb \
  --subnets $PUB_SUBNET1 $PUB_SUBNET2 \
  --security-groups $ALB_SG \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --region $REGION \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --region $REGION \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

echo "✅ ALB ARN: $ALB_ARN"
echo "✅ ALB DNS: $ALB_DNS"
```

### 14.3 Create Target Group

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name wordpress-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --matcher HttpCode=200-399 \
  --region $REGION \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

echo "✅ Target Group: $TG_ARN"
```

### 14.4 Create Listeners

```bash
# HTTP → HTTPS permanent redirect
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions '[{
    "Type": "redirect",
    "RedirectConfig": {
      "Protocol": "HTTPS",
      "Port": "443",
      "StatusCode": "HTTP_301"
    }
  }]' \
  --region $REGION

# HTTPS → forward to ECS Target Group
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTPS --port 443 \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --certificates CertificateArn=$CERT_ARN \
  --default-actions '[{
    "Type": "forward",
    "TargetGroupArn": "'"$TG_ARN"'"
  }]' \
  --region $REGION

echo "✅ ALB Listeners created (HTTP redirect + HTTPS forward)"
```

### 14.5 Point DNS to ALB

Add these records in your DNS provider:

| Type | Name | Value |
|------|------|-------|
| `CNAME` | `@` | `$ALB_DNS` |
| `CNAME` | `www` | `$ALB_DNS` |

> If using **Route 53**, use an `A` record with **Alias → Application Load Balancer** for the apex domain instead of a CNAME.

---

## 15. 🚀 ECS Service

```bash
aws ecs create-service \
  --cluster wordpress-cluster \
  --service-name wordpress-service \
  --task-definition wordpress-task \
  --desired-count 1 \
  --launch-type FARGATE \
  --platform-version LATEST \
  --network-configuration "awsvpcConfiguration={
    subnets=[${PRI_SUBNET1},${PRI_SUBNET2}],
    securityGroups=[${ECS_SG}],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=${TG_ARN},containerName=wordpress,containerPort=80" \
  --health-check-grace-period-seconds 180 \
  --deployment-configuration "minimumHealthyPercent=0,maximumPercent=100" \
  --enable-execute-command \
  --region $REGION

echo "✅ ECS Service created: wordpress-service"
```

> `minimumHealthyPercent=0, maximumPercent=100` is required for single-instance deployments. It stops the old task before starting the new one, ensuring only one task holds the MySQL EFS mount at a time.

Verify the service is running:

```bash
aws ecs describe-services \
  --cluster wordpress-cluster \
  --services wordpress-service \
  --region $REGION \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount,Event:events[0].message}'
```

---

## 16. 🤖 GitHub Actions CI/CD

### 16.1 Workflow File

Create `.github/workflows/deploy.yml` in your repository:

```yaml
name: Build & Deploy to ECS

on:
  push:
    branches:
      - main

env:
  AWS_REGION: ap-south-2
  ECR_REPOSITORY: docker-wordpress
  ECS_CLUSTER: wordpress-cluster
  ECS_SERVICE: wordpress-service
  TASK_DEFINITION: .aws/task-definition.json
  CONTAINER_NAME: wordpress

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    name: Build, Push & Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsWordPressRole
          aws-region: ${{ env.AWS_REGION }}

      # Alternative: static credentials (use if not using OIDC)
      # - name: Configure AWS credentials
      #   uses: aws-actions/configure-aws-credentials@v4
      #   with:
      #     aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      #     aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      #     aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push WordPress image to ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker tag  $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
                      $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Render updated task definition (wordpress container only)
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ env.TASK_DEFINITION }}
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      - name: Register & deploy task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true

      - name: Deployment summary
        run: |
          echo "✅ Deployed: ${{ steps.build-image.outputs.image }}"
          echo "🔗 https://${{ secrets.DOMAIN }}"
```

> `amazon-ecs-render-task-definition` only replaces the `wordpress` container image. The `mysql` container always keeps `mysql:8.0` — untouched on every deploy.

---

## 17. 🔐 GitHub Secrets Reference

Go to **Settings → Secrets and variables → Actions** in your repository and add:

| Secret | Value | Required |
|--------|-------|----------|
| `AWS_ACCOUNT_ID` | `123456789012` | ✅ Always |
| `DOMAIN` | `your-domain.com` | ✅ Always |
| `AWS_ACCESS_KEY_ID` | `AKIA...` | Only if **not** using OIDC |
| `AWS_SECRET_ACCESS_KEY` | `...` | Only if **not** using OIDC |

---

## 18. ✅ Post-Deployment

### Complete WordPress Setup

1. Visit `https://your-domain.com` — the WordPress installer will appear
2. Fill in site title, admin username, password, and email
3. Log in at `https://your-domain.com/wp-admin`

### Update WordPress URLs

In `wp-admin → Settings → General`, set both fields to `https://your-domain.com`.

Or via WP-CLI using ECS Exec from the Bastion Host or directly:

```bash
TASK_ID=$(aws ecs list-tasks \
  --cluster wordpress-cluster \
  --service-name wordpress-service \
  --region $REGION \
  --query 'taskArns[0]' \
  --output text)

aws ecs execute-command \
  --cluster wordpress-cluster \
  --task $TASK_ID \
  --container wordpress \
  --interactive \
  --command "/bin/bash"

# Inside the container:
# wp option update siteurl 'https://your-domain.com' --allow-root
# wp option update home 'https://your-domain.com' --allow-root
```

### Verify MySQL Persistence After Redeployment

```bash
aws ecs execute-command \
  --cluster wordpress-cluster \
  --task $TASK_ID \
  --container mysql \
  --interactive \
  --command "mysql -u root -p -e 'SHOW TABLES IN wordpress;'"
```

Your WordPress tables should be present after every redeployment since data lives on EFS, not inside the container.

---

## 19. 🛠️ Useful Commands

```bash
# Tail all container logs live
aws logs tail /ecs/wordpress --follow --region $REGION

# Tail only MySQL logs
aws logs tail /ecs/wordpress --follow \
  --log-stream-name-prefix mysql --region $REGION

# Tail only WordPress logs
aws logs tail /ecs/wordpress --follow \
  --log-stream-name-prefix wordpress --region $REGION

# List running tasks
aws ecs list-tasks \
  --cluster wordpress-cluster \
  --service-name wordpress-service \
  --region $REGION

# Force a new deployment (pull latest image)
aws ecs update-service \
  --cluster wordpress-cluster \
  --service wordpress-service \
  --force-new-deployment \
  --region $REGION

# Check EFS mount targets
aws efs describe-mount-targets \
  --file-system-id $EFS_ID \
  --region $REGION

# List EFS access points
aws efs describe-access-points \
  --file-system-id $EFS_ID \
  --region $REGION \
  --query 'AccessPoints[].{Id:AccessPointId,Path:RootDirectory.Path}'

# SSH into Bastion Host
ssh -i wordpress-bastion-key.pem ubuntu@${BASTION_IP}

# Verify resources
aws ec2 describe-vpcs --vpc-ids $VPC_ID
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID"
aws ec2 describe-instances --instance-ids $BASTION_ID
```

---

## 20. 🔍 Troubleshooting

### WordPress starts before MySQL is ready

The `dependsOn` with `condition: HEALTHY` ensures WordPress only starts after MySQL passes its health check. If you still see early connection errors on first boot, increase `startPeriod` in the MySQL `healthCheck` from `30` to `60`.

### How to check why a task stopped

```bash
aws ecs describe-tasks \
  --cluster wordpress-cluster \
  --tasks $(aws ecs list-tasks \
    --cluster wordpress-cluster \
    --desired-status STOPPED \
    --region $REGION \
    --query 'taskArns[0]' \
    --output text) \
  --region $REGION \
  --query 'tasks[0].{StopCode:stopCode,Reason:stoppedReason,Containers:containers[].{Name:name,ExitCode:exitCode,Reason:reason}}'
```

### EFS mount fails

- Confirm mount targets are in the **same private subnets** as ECS tasks (`$PRI_SUBNET1`, `$PRI_SUBNET2`)
- Confirm EFS security group allows TCP 2049 from the ECS security group
- Confirm task role has `elasticfilesystem:ClientMount` and `elasticfilesystem:ClientWrite`
- Confirm `transitEncryption: ENABLED` and `iam: ENABLED` match in task definition volumes

### MySQL fails — "data directory not empty"

Happens when a crashed MySQL init left partial files on EFS. Reset cleanly:

```bash
# ⚠️ This deletes all MySQL data
aws efs delete-access-point --access-point-id $EFS_AP_DB --region $REGION

# Recreate with a fresh path
EFS_AP_DB=$(aws efs create-access-point \
  --file-system-id $EFS_ID \
  --posix-user Uid=999,Gid=999 \
  --root-directory "Path=/mysql-data-v2,CreationInfo={OwnerUid=999,OwnerGid=999,Permissions=700}" \
  --tags Key=Name,Value=mysql-data-ap \
  --region $REGION \
  --query 'AccessPointId' --output text)

echo "✅ New MySQL Access Point: $EFS_AP_DB"
```

Update the task definition with the new `EFS_AP_DB` value and redeploy.

### ALB target group shows unhealthy

- MySQL + WordPress first boot takes 40–60 seconds — increase `--health-check-grace-period-seconds` to `240`
- Confirm health check path `/` returns HTTP 200–399
- Check both container logs for startup errors: `aws logs tail /ecs/wordpress --follow --region $REGION`

### HTTPS not working

- Confirm certificate: `aws acm describe-certificate --certificate-arn $CERT_ARN --region $REGION --query 'Certificate.Status'`
- Confirm HTTPS listener references the correct `CertificateArn`
- Confirm DNS CNAME/Alias points to the ALB DNS name, not a task IP

### Can't SSH to Bastion

- Confirm Bastion is in a **public subnet** with `--associate-public-ip-address`
- Confirm the Bastion security group allows TCP 22 from your IP
- Use `chmod 400 wordpress-bastion-key.pem` before SSH

---

## 21. 🧹 Cleanup

Run in reverse order to avoid dependency errors:

```bash
# ECS
aws ecs update-service --cluster wordpress-cluster --service wordpress-service --desired-count 0 --region $REGION
aws ecs delete-service --cluster wordpress-cluster --service wordpress-service --region $REGION
aws ecs delete-cluster --cluster wordpress-cluster --region $REGION

# ALB
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN --region $REGION
aws elbv2 delete-target-group --target-group-arn $TG_ARN --region $REGION

# ACM
aws acm delete-certificate --certificate-arn $CERT_ARN --region $REGION

# EFS
aws efs delete-access-point --access-point-id $EFS_AP_WP --region $REGION
aws efs delete-access-point --access-point-id $EFS_AP_DB --region $REGION
aws efs delete-mount-target --mount-target-id <mt-id-a> --region $REGION
aws efs delete-mount-target --mount-target-id <mt-id-b> --region $REGION
aws efs delete-file-system --file-system-id $EFS_ID --region $REGION

# ECR
aws ecr delete-repository --repository-name docker-wordpress --force --region $REGION

# EC2 Bastion
aws ec2 terminate-instances --instance-ids $BASTION_ID
aws ec2 delete-key-pair --key-name $KEY_NAME

# Networking (wait ~2 min after terminating instances)
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW_ID
aws ec2 release-address --allocation-id $EIP_ALLOC
aws ec2 delete-security-group --group-id $ALB_SG
aws ec2 delete-security-group --group-id $ECS_SG
aws ec2 delete-security-group --group-id $EFS_SG
aws ec2 delete-security-group --group-id $BASTION_SG
aws ec2 detach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-route-table --route-table-id $PUB_RT_ID
aws ec2 delete-route-table --route-table-id $PRI_RT_ID
aws ec2 delete-subnet --subnet-id $PUB_SUBNET1
aws ec2 delete-subnet --subnet-id $PUB_SUBNET2
aws ec2 delete-subnet --subnet-id $PRI_SUBNET1
aws ec2 delete-subnet --subnet-id $PRI_SUBNET2
aws ec2 delete-vpc --vpc-id $VPC_ID

echo "🧹 All resources deleted successfully."
```

---

## ✅ Security Checklist

- [ ] `.env` is in `.gitignore` — never committed to Git
- [ ] All credentials in Secrets Manager — not in task definition `environment` block
- [ ] ECS tasks in **private subnets** with `assignPublicIp=DISABLED`
- [ ] MySQL has **no `portMappings`** — port 3306 not reachable outside the task
- [ ] ALB SG allows only 80/443 from `0.0.0.0/0`
- [ ] ECS SG allows only port 80 from the ALB SG
- [ ] EFS SG allows only port 2049 from the ECS SG
- [ ] Bastion SG SSH restricted to your IP (not `0.0.0.0/0`) in production
- [ ] ECR scan-on-push enabled
- [ ] CloudWatch log retention set (30 days)
- [ ] ACM certificate covers both `domain.com` and `www.domain.com`
- [ ] GitHub Actions uses OIDC — no long-lived AWS keys stored in GitHub Secrets
- [ ] `--enable-execute-command` used only for emergency debugging
