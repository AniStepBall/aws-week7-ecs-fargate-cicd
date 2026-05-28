# AWS Week 7: ECS Fargate CI/CD Deployment

## About
This project deploys a Dockerized Flask application to AWS ECS Fargate using:

- Amazon ECR for container image storage
- Application Load Balancer (ALB) for traffic distribution
- GitHub Actions for automated CI/CD deployment workflows

The goal is to move from manually running containers on EC2 toward managed container orchestration with reduced infrastructure management overhead.

---

## Objective
Move from:
Docker containers manually managed on EC2
to:
Managed container deployment using ECS Fargate

Key goals:
- container orchestration
- automated deployment
- reproducible delivery
- health-based traffic routing
- reduced operational overhead
---

## Architecture

User
  ↓
Application Load Balancer
  ↓
ECS Fargate Service
  ↓
Dockerized Flask Application

Infrastructure components:
- ECS Cluster
- ECS Service
- ECS Task Definition
- Amazon ECR
- ALB Target Group
- GitHub Actions CI/CD pipeline

![Architecture](screenshots/architecture-diagram.png)

---
## Why ECS Fargate?

ECS Fargate removes the need to manage EC2 instances directly.

Benefits:
- no server provisioning
- no SSH management
- serverless container execution
- automatic task placement
- simplified scaling and operations

Tradeoff:
- less low-level infrastructure control than ECS on EC2
- higher cost at larger scale
  
---
## CI/CD Flow


### CI/CD Flow Chart
![Architecture](screenshots/cicd.png)

| Step        | Tool               | What Happens                                  |
| ----------- | ------------------ | --------------------------------------------- |
| Code Push   | GitHub             | Triggers GitHub Actions workflow              |
| Build Image | Docker             | Builds container image tagged with commit SHA |
| Push Image  | Amazon ECR         | Stores versioned container image              |
| Deploy      | ECS update-service | Forces rolling deployment of new tasks        |
| Verify      | ECS Wait           | Confirms service reaches healthy state        |

---
### Why Tag Images with Commit SHA?

Container images are tagged with the Git commit SHA to provide:
- deployment traceability
- rollback visibility
- immutable versioning
- stronger CI/CD reproducibility
This makes it easier to identify which application version is currently deployed.

---
## Security Model

| Layer                   | Rule                                                   |
| ----------------------- | ------------------------------------------------------ |
| ALB Security Group      | Accepts HTTP traffic on port 80 from the internet      |
| ECS Task Security Group | Accepts port 5000 traffic from ALB security group only |
| Amazon ECR              | Private registry with IAM-authenticated access         |
| GitHub Secrets          | AWS credentials stored encrypted outside source code   |

Key principle
- Only the ALB can communicate directly with application containers.

---

## Health Check

The app exposes a `/health` endpoint used by both the ALB and ECS
to determine whether containers should receive traffic.

```json
GET /health
{"status": "healthy"}
```
This endpoint is used by:
- ALB target group health checks
- ECS service health validation

Why this matters:
- Unhealthy containers stop receiving traffic
- ECS can replace failed tasks automatically
- enables rolling deployments with health validation
  
---

## How to Deploy

### Prerequisites
- AWS CLI configured
- Docker Desktop running
- GitHub Secrets set: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`



### Manual first push
An initial image push is required before CI/CD pipeline execution.

```bash
aws ecr get-login-password --region ca-central-1 | docker login \
  --username AWS --password-stdin \
  <account-id>.dkr.ecr.ca-central-1.amazonaws.com

docker build -t week7-fargate-app .
docker tag week7-fargate-app:latest \
  <account-id>.dkr.ecr.ca-central-1.amazonaws.com/week7-fargate-app:latest
docker push \
  <account-id>.dkr.ecr.ca-central-1.amazonaws.com/week7-fargate-app:latest
```

### Automated (after first push)
After initial setup:
```bash
git add .
git commit -m "your message"
git push origin main

```
GitHub Actions automatically:
- builds the image
- pushes to ECR
- triggers ECS deployment
- waits for healthy service state
---

## Validation
The deployment was validated through:
- ECS service maintaining desired task count
- ALB target group reporting healthy targets
- successful GitHub Actions pipeline completion
- /health endpoint returning healthy response
- scaling service to 3 tasks and verifying traffic distribution

---
## Troubleshooting Encountered

### Issue 1: 504 Gateway Timeout
**Cause** 
ECS tasks were initially deployed into private subnets without outbound internet access.

**As a result**
- Tasks could not pull images properly
- ALB health checks failed
- requests timed out

**Resolution**
Updated ECS service networking to:
- Use public subnets
- enable Auto-assign Public IP

**Production Improvement**
In production:
- tasks should run in private subnets
- outbound access should use NAT Gateway
  
### Issue 2: Exit Code 137 — Container Killed
**Cause** 
Task memory allocation was too low for:
- Python runtime
- Flask application
- container overhead

**As a result**
Fargate terminated the task with SIGKILL due to memory exhaustion.

**Resolution**
Increased task definition to:
- 0.5 vCPU
- 1 GB memory

**Lesson Learned**
Container memory limits must account for:
- runtime overhead
- application dependencies
- scaling conditions
                    
---

## Design Decisions and Tradeoffs

See [docs/decisions.md](docs/decisions.md)

Topics include:
- ECS Fargate vs EC2
- public vs private subnet deployment
- ALB integration
- operational overhead tradeoffs
  
---

## Screenshots

### ECS Service Running
![ECS Service](screenshots/ecs-service-running.png)

### ALB Targets Healthy
![Healthy Targets](screenshots/alb-targets-healthy.png)

### GitHub Actions Success
![CI/CD Pipeline](screenshots/github-actions-success.png)

### App Response
![App Response](screenshots/app-response.png)

## Future Improvements
- HTTPS using ACM certificates
- Private subnet deployment with NAT Gateway
- ECS auto scaling policies
- CloudWatch log aggregation
- Blue/green deployments
- Secrets Manager integration
- Terraform-based ECS provisioning
- WAF integration
