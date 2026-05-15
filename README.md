# 🏗️ WordPress on AWS ECS — Production Deployment Guide

> **Stack:** WordPress + PHP 8.1-FPM + Nginx · MySQL 8.0 (ECS sidecar) · Amazon EFS · ECS Fargate · ECR · ALB + ACM (HTTPS)
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
16. [Post-Deployment](#16--post-deployment)
17. [Useful Commands](#17--useful-commands)
18. [Troubleshooting](#18--troubleshooting)
19. [Cleanup](#19--cleanup)

---

## 1. 🧩 Architecture Overview

```
Manual Deploy (see Step 9)
            │
            ▼
      Internet (HTTPS :443)
            │
            ▼
      ALB (public subnet) ◄── ACM Certificate
      HTTP :80 → HTTPS :443 redirect
      X-Forwarded-Proto: https ──────────────────┐
            │                                    │
            ▼                                    ▼
   ECS Fargate Task (PRIVATE subnet, no public IP)
   ┌──────────────────────────────────────────────┐
   │                                              │
   │  ┌───────────────────────────────────┐       │
   │  │   wordpress container       :80   │       │
   │  │   Nginx + PHP 8.1-FPM             │       │
   │  │   wp-config: HTTPS forced via     │       │
   │  │   HTTP_X_FORWARDED_PROTO header   │       │
   │  │   connects to MySQL: 127.0.0.1    │       │
   │  └──────────────────┬────────────────┘       │
   │                     │ dependsOn: HEALTHY      │
   │  ┌──────────────────▼────────────────┐       │
   │  │   mysql container          :3306  │       │
   │  │   mysql:8.0                       │       │
   │  └──────────┬──────────────┬─────────┘       │
   │             ▼              ▼                  │
   │      EFS /wp-content  EFS /mysql-data         │
   │                                              │
   └──────────────────────────────────────────────┘
            │
            ▼ (outbound only, via NAT Gateway)
      Docker Hub (mysql:8.0)
      ECR (wordpress image)
      Secrets Manager

   Public Subnets:  ALB + NAT Gateway + Bastion Host
   Private Subnets: ECS Tasks + EFS Mount Targets
```

### Why Two Bugs Occur Together

| Problem | Root Cause | Fix |
|---|---|---|
| ECS task fails in private subnet | `assignPublicIp=DISABLED` but no NAT Gateway route — Docker Hub / ECR unreachable | NAT Gateway in public subnet + private route table pointing `0.0.0.0/0` → NAT |
| WordPress generates `http://` links | ALB terminates HTTPS and forwards plain HTTP to ECS; WordPress sees HTTP and sets all URLs accordingly | Tell WordPress to trust `X-Forwarded-Proto` header from the ALB (via `wp-config.php`) |

---

## 2. 🧰 Prerequisites

- **AWS CLI v2** installed and configured
- **Docker** installed locally (for the initial image push)
- A registered **domain name** with DNS access
- AWS account in `ap-south-2`

```bash
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
# Public Subnets — ALB, NAT Gateway, Bastion Host
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

# Private Subnets — ECS Tasks + EFS (no direct internet, NAT only)
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

## 5. 🌍 Create Internet Gateway and Public Route Table

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 create-tags --resources $IGW_ID --tags Key=Name,Value=${VPC_NAME}-igw

# Public Route Table — IGW for ALB and Bastion
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

echo "✅ Internet Gateway:   $IGW_ID"
echo "✅ Public Route Table: $PUB_RT_ID"
```

---

## 6. 🔀 Create NAT Gateway + Private Route Table

> ⚠️ **This is the fix for private-subnet ECS task failures.**
>
> ECS tasks with `assignPublicIp=DISABLED` have no direct internet route. Without a NAT Gateway they cannot reach:
> - **Docker Hub** — to pull `mysql:8.0`
> - **ECR** — to pull the WordPress image
> - **Secrets Manager** — to fetch credentials at startup
>
> The NAT Gateway sits in a **public subnet** and gives private subnet resources outbound-only internet access. Inbound connections from the internet are still blocked.

```bash
# Elastic IP for the NAT Gateway
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

# NAT Gateway must be in a PUBLIC subnet
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUBNET1 \
  --allocation-id $EIP_ALLOC \
  --query 'NatGateway.NatGatewayId' \
  --output text)
aws ec2 create-tags --resources $NAT_GW_ID --tags Key=Name,Value=wp-nat-gw

echo "⏳ Waiting for NAT Gateway to become available (~60s)..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_ID
echo "✅ NAT Gateway: $NAT_GW_ID"

# Private Route Table — all outbound goes via NAT (not IGW)
PRI_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)
aws ec2 create-tags --resources $PRI_RT_ID --tags Key=Name,Value=wp-private-rt

aws ec2 create-route \
  --route-table-id $PRI_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_ID        # ← NAT, not IGW

aws ec2 associate-route-table --route-table-id $PRI_RT_ID --subnet-id $PRI_SUBNET1
aws ec2 associate-route-table --route-table-id $PRI_RT_ID --subnet-id $PRI_SUBNET2

echo "✅ Private Route Table: $PRI_RT_ID (routes via NAT)"
```

**Verify the route is correct:**
```bash
aws ec2 describe-route-tables \
  --route-table-ids $PRI_RT_ID \
  --query 'RouteTables[0].Routes'
# Should show: DestinationCidrBlock=0.0.0.0/0, NatGatewayId=nat-xxx
# NOT GatewayId=igw-xxx
```

---

## 7. 🔒 Create Security Groups

```bash
# ALB — accepts HTTP/HTTPS from anywhere
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

# ECS Task — accepts port 80 from ALB only (private subnet, no public IP)
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

# EFS — accepts NFS from ECS tasks only
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

# Bastion — SSH from your IP only
BASTION_SG=$(aws ec2 create-security-group \
  --group-name wp-bastion-sg \
  --description "Bastion Host - SSH access" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)
aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG \
  --protocol tcp --port 22 \
  --cidr 0.0.0.0/0   # ⚠️ Replace with your IP: e.g. 203.0.113.10/32
aws ec2 create-tags --resources $BASTION_SG --tags Key=Name,Value=wp-bastion-sg

echo "✅ ALB SG:     $ALB_SG"
echo "✅ ECS SG:     $ECS_SG"
echo "✅ EFS SG:     $EFS_SG"
echo "✅ Bastion SG: $BASTION_SG"
```

> MySQL port 3306 has **no security group rule**. It is only reachable at `127.0.0.1` inside the ECS task — never on the network.

---

## 8. 💻 Launch a Bastion Host (Ubuntu EC2)

```bash
KEY_NAME="wordpress-bastion-key"

aws ec2 create-key-pair \
  --key-name $KEY_NAME \
  --query 'KeyMaterial' \
  --output text > ${KEY_NAME}.pem
chmod 400 ${KEY_NAME}.pem

# Latest Ubuntu 24.04 LTS AMI
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
  --query 'Images | sort_by(@, &CreationDate)[-1].ImageId' \
  --output text \
  --region $REGION)

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

### 9.2 Fix WordPress `wp-config.php` Before Building

> ⚠️ **This is the fix for WordPress generating `http://` links behind the ALB.**
>
> The ALB terminates HTTPS and forwards plain HTTP to the ECS container. WordPress sees the request as HTTP and generates all internal links as `http://`. The fix is adding these lines to `wp-config.php` **before** `require_once ABSPATH . 'wp-settings.php'`:
>
> ```php
> // Trust the X-Forwarded-Proto header sent by the ALB
> if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
>     $_SERVER['HTTPS'] = 'on';
> }
> define('FORCE_SSL_ADMIN', true);
> ```
>
> This tells WordPress: "if the ALB says the original request was HTTPS, treat this as an HTTPS connection." WordPress then generates `https://` for all URLs, CSS, JS, and admin redirects.

Edit your repo's `wp-config.php` (or wherever it is generated in the Dockerfile) and add the block above. Then build and push:

```bash
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin \
  "${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"

git clone https://github.com/sudhanlogics/docker-wordpress.git
cd docker-wordpress

# --- ADD THE FIX to wp-config.php here before building ---

docker build -t docker-wordpress .
docker tag docker-wordpress:latest ${ECR_URI}:latest
docker push ${ECR_URI}:latest

echo "✅ Image pushed: ${ECR_URI}:latest"
```

### 9.3 Lifecycle Policy

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

### 10.2 Create Mount Targets in Private Subnets

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

echo "⏳ Waiting 30s for EFS mount targets..."
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

echo "✅ WordPress EFS AP: $EFS_AP_WP"
echo "✅ MySQL EFS AP:     $EFS_AP_DB"
```

---

## 11. 🔑 Secrets Manager

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

---

## 12. 🛡️ IAM Roles & Policies

### 12.1 ECS Task Execution Role

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

aws iam attach-role-policy \
  --role-name ecsWordpressTaskRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

echo "✅ ecsWordpressTaskRole ready"
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

aws logs create-log-group --log-group-name /ecs/wordpress --region $REGION
aws logs put-retention-policy \
  --log-group-name /ecs/wordpress \
  --retention-in-days 30 \
  --region $REGION

echo "✅ ECS Cluster: wordpress-cluster"
```

### 13.2 Task Definition Template

Save as `.aws/task-definition.json`. Variables like `ACCOUNT_ID`, `EFS_ID`, `EFS_AP_WP`, `EFS_AP_DB`, `IMAGE_TAG` are substituted at registration time by the `sed` command in Step 13.3 — **do not hard-code them**.

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

### 13.3 Render & Register Task Definition

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Substitute all placeholders — no manual editing needed
sed -e "s/ACCOUNT_ID/$ACCOUNT_ID/g" \
    -e "s/EFS_ID/$EFS_ID/g" \
    -e "s/EFS_AP_WP/$EFS_AP_WP/g" \
    -e "s/EFS_AP_DB/$EFS_AP_DB/g" \
    -e "s/IMAGE_TAG/latest/g" \
    .aws/task-definition.json > /tmp/task-definition-rendered.json

aws ecs register-task-definition \
  --cli-input-json file:///tmp/task-definition-rendered.json \
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

Get DNS validation records:

```bash
aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --region $REGION \
  --query 'Certificate.DomainValidationOptions[].{Name:ResourceRecord.Name,Value:ResourceRecord.Value}'
```

Add the CNAMEs to your DNS provider, then wait:

```bash
aws acm wait certificate-validated --certificate-arn $CERT_ARN --region $REGION
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

# HTTPS → forward to ECS (ALB sends X-Forwarded-Proto: https to the container)
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

echo "✅ ALB Listeners created"
```

### 14.5 Point DNS to ALB

| Type | Name | Value |
|------|------|-------|
| `CNAME` | `@` | `$ALB_DNS` |
| `CNAME` | `www` | `$ALB_DNS` |

> Route 53 users: use an `A` record with **Alias → Application Load Balancer** for the apex domain.

---

## 15. 🚀 ECS Service

> ⚠️ **Private subnet is correct here.** Tasks have `assignPublicIp=DISABLED` and use the NAT Gateway (Step 6) for outbound access. The ALB routes inbound traffic from its public subnets to the private task IPs.

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

echo "✅ ECS Service: wordpress-service (private subnet, NAT outbound)"
```

Verify:

```bash
aws ecs describe-services \
  --cluster wordpress-cluster \
  --services wordpress-service \
  --region $REGION \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount,Event:events[0].message}'
```

---


## 16. ✅ Post-Deployment

### Complete WordPress Setup

1. Visit `https://your-domain.com` — the WordPress installer will appear
2. Fill in site title, admin username, password, and email
3. Log in at `https://your-domain.com/wp-admin`

### Verify HTTPS Links Are Correct

After setup, check `wp-admin → Settings → General`. Both **WordPress Address** and **Site Address** must show `https://your-domain.com`. If they show `http://`, the `wp-config.php` fix in Step 9.2 was not included in the build — rebuild and redeploy.

### ECS Exec into Containers

```bash
TASK_ID=$(aws ecs list-tasks \
  --cluster wordpress-cluster \
  --service-name wordpress-service \
  --region $REGION \
  --query 'taskArns[0]' \
  --output text)

# Exec into WordPress container
aws ecs execute-command \
  --cluster wordpress-cluster \
  --task $TASK_ID \
  --container wordpress \
  --interactive \
  --command "/bin/bash"

# Exec into MySQL container
aws ecs execute-command \
  --cluster wordpress-cluster \
  --task $TASK_ID \
  --container mysql \
  --interactive \
  --command "mysql -u root -p -e 'SHOW TABLES IN wordpress;'"
```

---

## 17. 🛠️ Useful Commands

```bash
# Tail logs (all containers)
aws logs tail /ecs/wordpress --follow --region $REGION

# Tail MySQL only
aws logs tail /ecs/wordpress --follow \
  --log-stream-name-prefix mysql --region $REGION

# Tail WordPress only
aws logs tail /ecs/wordpress --follow \
  --log-stream-name-prefix wordpress --region $REGION

# Force new deployment
aws ecs update-service \
  --cluster wordpress-cluster \
  --service wordpress-service \
  --force-new-deployment \
  --region $REGION

# Verify private subnet routing
aws ec2 describe-route-tables \
  --route-table-ids $PRI_RT_ID \
  --query 'RouteTables[0].Routes'

# Check EFS mount targets
aws efs describe-mount-targets \
  --file-system-id $EFS_ID --region $REGION

# SSH to Bastion
ssh -i wordpress-bastion-key.pem ubuntu@${BASTION_IP}
```

---

## 18. 🔍 Troubleshooting

### Private subnet — task fails to start / cannot pull image

**Symptom:** Task stops immediately; stopped reason mentions `CannotPullContainerError` or `ResourceInitializationError`.

```bash
# Check stopped task reason
aws ecs describe-tasks \
  --cluster wordpress-cluster \
  --tasks $(aws ecs list-tasks \
    --cluster wordpress-cluster \
    --desired-status STOPPED \
    --region $REGION \
    --query 'taskArns[0]' \
    --output text) \
  --region $REGION \
  --query 'tasks[0].{StopReason:stoppedReason,Containers:containers[].{Name:name,Reason:reason}}'
```

**Fix checklist:**
- Confirm private subnets are associated with `$PRI_RT_ID` (the NAT route table, **not** the public one)
- Confirm `0.0.0.0/0` in `$PRI_RT_ID` points to `nat-xxx` not `igw-xxx`
- Confirm NAT Gateway is in a **public subnet** and its Elastic IP is allocated
- Confirm ECS service uses `assignPublicIp=DISABLED` with `subnets=$PRI_SUBNET1,$PRI_SUBNET2`

```bash
# Quick route check — must show NatGatewayId, not GatewayId
aws ec2 describe-route-tables \
  --route-table-ids $PRI_RT_ID \
  --query 'RouteTables[0].Routes[?DestinationCidrBlock==`0.0.0.0/0`]'
```

### WordPress shows `http://` links / mixed content warnings

**Symptom:** Site loads at `https://` but images, CSS, or admin redirects use `http://`. Browser shows mixed content errors.

**Root cause:** ALB terminates TLS and forwards plain HTTP to the container. WordPress sees `$_SERVER['HTTPS']` as empty and uses HTTP for all generated URLs.

**Fix — add to `wp-config.php` before `require_once ABSPATH . 'wp-settings.php'`:**

```php
// Fix for WordPress behind an HTTPS ALB/reverse proxy
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}
define('FORCE_SSL_ADMIN', true);
```

After adding the fix, rebuild the Docker image and redeploy. Then in `wp-admin → Settings → General` update both URLs to `https://your-domain.com` and save.

### WordPress admin redirect loop

**Symptom:** `wp-admin` keeps redirecting between http and https.

Same fix as above — the `HTTP_X_FORWARDED_PROTO` block must be present in `wp-config.php` and the WordPress/Site Address must both be `https://`.

### EFS mount fails

- Confirm mount targets are in `$PRI_SUBNET1` and `$PRI_SUBNET2` (same AZ as tasks)
- Confirm EFS SG allows TCP 2049 from ECS SG
- Confirm task role has `elasticfilesystem:ClientMount` and `elasticfilesystem:ClientWrite`
- Confirm `transitEncryption=ENABLED` and `iam=ENABLED` in task definition volumes

### MySQL fails — "data directory not empty"

```bash
# ⚠️ Deletes all MySQL data — only use if database is fresh/recoverable
aws efs delete-access-point --access-point-id $EFS_AP_DB --region $REGION

EFS_AP_DB=$(aws efs create-access-point \
  --file-system-id $EFS_ID \
  --posix-user Uid=999,Gid=999 \
  --root-directory "Path=/mysql-data-v2,CreationInfo={OwnerUid=999,OwnerGid=999,Permissions=700}" \
  --tags Key=Name,Value=mysql-data-ap \
  --region $REGION \
  --query 'AccessPointId' --output text)

echo "✅ New MySQL AP: $EFS_AP_DB"
# Re-render and re-register the task definition with the new EFS_AP_DB value
```

### ALB target group unhealthy

- First boot (MySQL init + WordPress) takes 40–60s — set `--health-check-grace-period-seconds 240`
- Confirm `/` returns HTTP 200–399
- Check logs: `aws logs tail /ecs/wordpress --follow --region $REGION`

---

## 19. 🧹 Cleanup

```bash
# ECS
aws ecs update-service --cluster wordpress-cluster --service wordpress-service --desired-count 0 --region $REGION
sleep 15
aws ecs delete-service --cluster wordpress-cluster --service wordpress-service --region $REGION
aws ecs delete-cluster --cluster wordpress-cluster --region $REGION

# ALB
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN --region $REGION
sleep 15
aws elbv2 delete-target-group --target-group-arn $TG_ARN --region $REGION

# ACM
aws acm delete-certificate --certificate-arn $CERT_ARN --region $REGION

# EFS
aws efs delete-access-point --access-point-id $EFS_AP_WP --region $REGION
aws efs delete-access-point --access-point-id $EFS_AP_DB --region $REGION

MT_IDS=$(aws efs describe-mount-targets \
  --file-system-id $EFS_ID --region $REGION \
  --query 'MountTargets[].MountTargetId' --output text)
for mt in $MT_IDS; do
  aws efs delete-mount-target --mount-target-id $mt --region $REGION
done
sleep 30
aws efs delete-file-system --file-system-id $EFS_ID --region $REGION

# ECR
aws ecr delete-repository --repository-name docker-wordpress --force --region $REGION

# EC2 Bastion
aws ec2 terminate-instances --instance-ids $BASTION_ID
aws ec2 wait instance-terminated --instance-ids $BASTION_ID
aws ec2 delete-key-pair --key-name $KEY_NAME
rm -f ${KEY_NAME}.pem

# Networking (wait for NAT to delete, ~2 min)
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW_ID
echo "⏳ Waiting for NAT Gateway to delete..."
sleep 120
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
- [ ] All credentials in Secrets Manager — not hardcoded anywhere
- [ ] ECS tasks in **private subnets** with `assignPublicIp=DISABLED`
- [ ] Private subnets route `0.0.0.0/0` via **NAT Gateway**, not IGW
- [ ] MySQL has no `portMappings` — port 3306 unreachable outside the task
- [ ] ALB SG allows only 80/443 from `0.0.0.0/0`
- [ ] ECS SG allows only port 80 from the ALB SG
- [ ] EFS SG allows only port 2049 from the ECS SG
- [ ] Bastion SG SSH restricted to your IP (not `0.0.0.0/0`) in production
- [ ] `wp-config.php` includes `HTTP_X_FORWARDED_PROTO` fix — WordPress generates `https://` links
- [ ] WordPress Address and Site Address both set to `https://your-domain.com`
- [ ] ECR scan-on-push enabled
- [ ] CloudWatch log retention set (30 days)
- [ ] ACM certificate covers both `domain.com` and `www.domain.com`
