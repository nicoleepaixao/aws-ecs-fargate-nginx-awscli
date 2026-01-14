<div align="center">
  
![AWS ECS](https://img.icons8.com/color/96/amazon-web-services.png)
![Nginx](https://img.icons8.com/color/96/nginx.png)

## AWS ECS Fargate Nginx Deployment – Complete Infrastructure Setup with AWS CLI

**Updated: January 14, 2026**

[![Follow @nicoleepaixao](https://img.shields.io/github/followers/nicoleepaixao?label=Follow&style=social)](https://github.com/nicoleepaixao)
[![Star this repo](https://img.shields.io/github/stars/nicoleepaixao/aws-ecs-fargate-nginx-awscli?style=social)](https://github.com/nicoleepaixao/aws-ecs-fargate-nginx-awscli)
[![Medium Article](https://img.shields.io/badge/Medium-12100E?style=for-the-badge&logo=medium&logoColor=white)](https://nicoleepaixao.medium.com/)

<p align="center">
  <a href="README-PT.md">🇧🇷</a>
  <a href="README.md">🇺🇸</a>
</p>

</div>

---

<p align="center">
  <img src="img/ecs-fargate.png" alt="ECS Architecture" width="1800">
</p>

## **The Problem**

Setting up an ECS Fargate service with Nginx manually involves creating multiple AWS resources in the correct order. Every time you need to spin up infrastructure, you spend **up to 1 hour** creating resources individually and reviewing configurations. Additionally, the chances of making mistakes are high, especially regarding permissions, network settings, and ALB integration.

This project solves that problem by documenting the **complete infrastructure setup** using AWS CLI commands for **two scenarios**: using an existing ALB or creating one from scratch. The result is a **repeatable, production-ready process** that reduces setup time by 95%.

---

## **Objective**

Create a **complete infrastructure setup** that:

- Deploys **Nginx** on **ECS Fargate**
- Behind an **Application Load Balancer (ALB)**
- In **private subnets** (best practice)
- With logs in **CloudWatch Logs**
- Uses **ECR** for container image registry
- Implements proper **security groups** and **IAM roles**

This guide covers **two scenarios**:

**Scenario A** — ALB **already exists**  
**Scenario B** — Create an **ALB from scratch** using AWS CLI

---

## **ECS Fargate Architecture Overview**

**Understanding Amazon ECS Fargate:**

Amazon ECS (Elastic Container Service) with Fargate is a serverless container orchestration platform. Unlike EC2 launch type, Fargate manages the underlying infrastructure automatically, eliminating the need to provision or scale servers.

**Key Components:**

| **Component** | **Purpose** |
|---------------|------------|
| **Cluster** | Logical grouping of tasks and services |
| **Task Definition** | Blueprint defining container specifications (image, CPU, memory, ports) |
| **Service** | Maintains desired number of tasks running and integrates with load balancers |
| **Task** | Running instance of a task definition (one or more containers) |
| **Target Group** | Routes ALB traffic to ECS tasks using IP-based targets |

**Network Mode: awsvpc**

ECS Fargate requires `awsvpc` network mode, which means:
- Each task gets its own Elastic Network Interface (ENI)
- Tasks receive private IP addresses from VPC subnets
- Security groups are applied directly to tasks
- Tasks can communicate with other AWS services via VPC endpoints

---

## **Project Structure**

```text
aws-ecs-fargate-nginx-awscli/
│
├── README.md                     # Complete documentation
├── Dockerfile                    # Nginx container image
├── nginx.conf                    # Custom Nginx configuration
├── html/
│   └── index.html               # Static HTML page
├── aws/
│   ├── taskdef-nginx.json       # Task definition (prod)
│   └── taskdef-nginx-homol.json # Task definition (homol)
└── iam/
    └── ecs-taskrole-trust.json  # Task role trust policy
```

---

## **Container Configuration**

**Dockerfile:**

```dockerfile
FROM nginx:1.27-alpine

# Copy custom Nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Copy static files
COPY html /usr/share/nginx/html

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --retries=3 CMD \
  wget -qO- http://127.0.0.1/healthz || exit 1
```

**nginx.conf:**

```nginx
events {}

http {
  server {
    listen 80;

    location / {
      root   /usr/share/nginx/html;
      index  index.html;
    }

    location /healthz {
      return 200 'ok';
      add_header Content-Type text/plain;
    }
  }
}
```

**html/index.html:**

```html
<!DOCTYPE html>
<html>
<head>
  <title>Nginx ECS - Stage</title>
</head>
<body>
  <h1>Deu bom! Nginx rodando no ECS 🎉</h1>
</body>
</html>
```

---

## **Step 1: CloudWatch Logs Setup**

Create log group for container logs:

```bash
aws logs create-log-group \
  --log-group-name /ecs/nginx \
  --region us-east-1
```

Set retention policy (30 days):

```bash
aws logs put-retention-policy \
  --log-group-name /ecs/nginx \
  --retention-in-days 30 \
  --region us-east-1
```

---

## **Step 2: Create ECR Repository**

Create repository for Nginx images:

```bash
aws ecr create-repository \
  --repository-name ecr_nginx \
  --image-tag-mutability MUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1
```

Get repository URI:

```bash
aws ecr describe-repositories \
  --repository-names ecr_nginx \
  --query 'repositories[0].repositoryUri' \
  --output text \
  --region us-east-1
```

**Save this URI** - you'll need it in task definitions.

---

## **Step 3: IAM Roles for ECS Tasks**

**Create trust policy** `iam/ecs-taskrole-trust.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create the task role:

```bash
aws iam create-role \
  --role-name ecsTaskRole-nginx \
  --assume-role-policy-document file://iam/ecs-taskrole-trust.json
```

**Optional** — attach policies for testing (remove in production):

```bash
aws iam attach-role-policy \
  --role-name ecsTaskRole-nginx \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

**Note:** The execution role (`ecsTaskExecutionRole`) is typically created automatically by AWS, but if needed:

```bash
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://iam/ecs-taskrole-trust.json

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

---

## **Scenario A: Using an Existing ALB**

**Step 4A: Get VPC from existing ALB:**

```bash
aws elbv2 describe-load-balancers \
  --load-balancer-arns <ALB_ARN_EXISTENTE> \
  --query 'LoadBalancers[0].VpcId' \
  --output text \
  --region us-east-1
```

---

**Step 5A: Create Target Group (Fargate requires target-type: ip):**

```bash
aws elbv2 create-target-group \
  --name tg-nginx-stage \
  --protocol HTTP \
  --port 80 \
  --vpc-id <VPC_ID> \
  --target-type ip \
  --health-check-path /healthz \
  --health-check-port traffic-port \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 3 \
  --unhealthy-threshold-count 3 \
  --matcher HttpCode=200 \
  --region us-east-1
```

Get Target Group ARN:

```bash
aws elbv2 describe-target-groups \
  --names tg-nginx-stage \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text \
  --region us-east-1
```

---

**Step 6A: List ALB listeners:**

```bash
aws elbv2 describe-listeners \
  --load-balancer-arn <ALB_ARN_EXISTENTE> \
  --region us-east-1
```

Choose **listener HTTPS :443** (or HTTP :80 for lab) and copy the `ListenerArn`.

---

**Step 7A: Create listener rule (path-based routing):**

```bash
aws elbv2 create-rule \
  --listener-arn <LISTENER_ARN_HTTPS_443> \
  --priority 110 \
  --conditions Field=path-pattern,Values="/nginx/*" \
  --actions Type=forward,TargetGroupArn=<TARGET_GROUP_ARN> \
  --region us-east-1
```

Any request to:
```
https://your-domain.com/nginx/...
```

Will forward to `tg-nginx-stage` → ECS tasks.

---

## **Scenario B: Creating ALB from Scratch**

**Step 4B: Create security group for ALB:**

```bash
aws ec2 create-security-group \
  --group-name alb-nginx-stage-sg \
  --description "Security Group for nginx stage ALB" \
  --vpc-id <VPC_ID> \
  --region us-east-1
```

Allow HTTP and HTTPS traffic:

```bash
# HTTP
aws ec2 authorize-security-group-ingress \
  --group-id <ALB_SG_ID> \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region us-east-1

# HTTPS (optional)
aws ec2 authorize-security-group-ingress \
  --group-id <ALB_SG_ID> \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0 \
  --region us-east-1
```

---

**Step 5B: Create ALB in public subnets:**

```bash
aws elbv2 create-load-balancer \
  --name nginx-stage-alb \
  --type application \
  --scheme internet-facing \
  --security-groups <ALB_SG_ID> \
  --subnets <SUBNET_PUBLIC_1> <SUBNET_PUBLIC_2> \
  --region us-east-1
```

**Save** the `LoadBalancerArn` and `DNSName` from output.

---

**Step 6B: Create target group (same as Scenario A):**

```bash
aws elbv2 create-target-group \
  --name tg-nginx-stage \
  --protocol HTTP \
  --port 80 \
  --vpc-id <VPC_ID> \
  --target-type ip \
  --health-check-path /healthz \
  --health-check-port traffic-port \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 3 \
  --unhealthy-threshold-count 3 \
  --matcher HttpCode=200 \
  --region us-east-1
```

---

**Step 7B: Create HTTP listener (port 80):**

```bash
aws elbv2 create-listener \
  --load-balancer-arn <ALB_ARN_NOVO> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<TARGET_GROUP_ARN> \
  --region us-east-1
```

**Optional** — Create HTTPS listener with ACM certificate:

```bash
aws elbv2 create-listener \
  --load-balancer-arn <ALB_ARN_NOVO> \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=<ACM_CERT_ARN> \
  --default-actions Type=forward,TargetGroupArn=<TARGET_GROUP_ARN> \
  --region us-east-1
```

---

## **Step 8: Security Group for ECS Service**

Create security group for Fargate tasks:

```bash
aws ec2 create-security-group \
  --group-name nginx-stage-service \
  --description "Security Group for nginx-stage-service ECS Fargate" \
  --vpc-id <VPC_ID> \
  --region us-east-1
```

Get ALB security group ID:

```bash
aws elbv2 describe-load-balancers \
  --load-balancer-arns <ALB_ARN> \
  --query 'LoadBalancers[0].SecurityGroups[0]' \
  --output text \
  --region us-east-1
```

Allow traffic **only from ALB** → Service:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id <SERVICE_SG_ID> \
  --protocol tcp \
  --port 80 \
  --source-group <ALB_SG_ID> \
  --region us-east-1
```

---

## **Step 9: Task Definitions**

**Task Definition for Homol** `aws/taskdef-nginx-homol.json`:

```json
{
  "family": "nginx-homol",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "runtimePlatform": {
    "cpuArchitecture": "X86_64",
    "operatingSystemFamily": "LINUX"
  },
  "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskRole-nginx",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ecr_nginx:latest",
      "essential": true,
      "cpu": 256,
      "memory": 512,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/nginx",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "nginx"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -qO- http://127.0.0.1/healthz || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 10
      },
      "environment": [
        {
          "name": "APP_ENV",
          "value": "homol"
        }
      ]
    }
  ]
}
```

**Task Definition for Prod** `aws/taskdef-nginx.json`:

Same as above, change:
- `"family": "nginx"`
- `"value": "prod"` in environment

Register both task definitions:

```bash
aws ecs register-task-definition \
  --cli-input-json file://aws/taskdef-nginx-homol.json \
  --region us-east-1

aws ecs register-task-definition \
  --cli-input-json file://aws/taskdef-nginx.json \
  --region us-east-1
```

---

## **Step 10: Build and Push Docker Image**

Build image:

```bash
docker build -t ecr_nginx:latest .
```

Login to ECR:

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

Tag and push:

```bash
docker tag ecr_nginx:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ecr_nginx:latest
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ecr_nginx:latest
```

---

## **Step 11: Create ECS Cluster**

If you don't have a cluster yet:

```bash
aws ecs create-cluster \
  --cluster-name my-cluster \
  --region us-east-1
```

---

## **Step 12: Create ECS Service**

Create service:

```bash
aws ecs create-service \
  --cluster my-cluster \
  --service-name nginx-service-stage \
  --task-definition nginx-homol \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[<SUBNET_PRIVATE_1>,<SUBNET_PRIVATE_2>],securityGroups=[<SERVICE_SG_ID>],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=<TARGET_GROUP_ARN>,containerName=nginx,containerPort=80" \
  --region us-east-1
```

---

## **Troubleshooting**

**Check ECS service events:**

```bash
aws ecs describe-services \
  --cluster my-cluster \
  --services nginx-service-stage \
  --region us-east-1 \
  --query 'services[0].events[0:10]'
```

**Common errors:**

| **Error** | **Cause** | **Solution** |
|-----------|-----------|-------------|
| `Unable to assume role` | Task role doesn't exist or trust policy wrong | Verify role and trust policy |
| `ResourceInitializationError` | Execution role missing ECR permissions | Attach `AmazonECSTaskExecutionRolePolicy` |
| `CannotPullContainerError` | Image doesn't exist in ECR | Verify image was pushed |
| `Service unhealthy` | Health check failing | Check `/healthz` endpoint |

**Force new deployment:**

```bash
aws ecs update-service \
  --cluster my-cluster \
  --service nginx-service-stage \
  --force-new-deployment \
  --region us-east-1
```

---

## **Validation**

**Test ALB endpoint:**

```bash
curl http://<ALB_DNS_NAME>
# or
curl http://<ALB_DNS_NAME>/nginx/
```

Expected: `<h1>Deu bom! Nginx rodando no ECS 🎉</h1>`

**Check CloudWatch Logs:**

```bash
aws logs tail /ecs/nginx --follow
```

**Verify target group health:**

```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN> \
  --region us-east-1
```

---

## **CI/CD Automation**

This repository focuses on infrastructure setup. For automated deployment pipelines:

**GitHub Actions + OIDC Integration:**
- [AWS GitHub OIDC Setup](https://github.com/nicoleepaixao/aws-github-oidc-pipeline) - Configure OIDC authentication with zero long-lived credentials
- [Complete ECS CI/CD Pipeline](https://github.com/nicoleepaixao/aws-ecs-fargate-nginx-oidc-pipeline) - Full automated workflow for building, pushing, and deploying to ECS

These pipelines eliminate the need for AWS access keys in CI/CD by using OpenID Connect (OIDC) for secure, temporary credentials.

---

## **Additional Resources**

- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/) - Official guide
- [Fargate Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/) - Production guidelines
- [ALB Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/) - Load balancer reference
- [ECR Documentation](https://docs.aws.amazon.com/ecr/) - Container registry guide

---

## **Connect & Follow**

<div align="center">

[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/nicoleepaixao)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?logo=linkedin&logoColor=white&style=for-the-badge)](https://www.linkedin.com/in/nicolepaixao/)
[![Medium](https://img.shields.io/badge/Medium-12100E?style=for-the-badge&logo=medium&logoColor=white)](https://medium.com/@nicoleepaixao)

</div>

---

<div align="center">

**Happy deploying to ECS!**

*Document Created: January 14, 2026*

Made with ❤️ by [Nicole Paixão](https://github.com/nicoleepaixao)

</div>
