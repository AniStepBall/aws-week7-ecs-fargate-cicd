# Design Decisions and Tradeoffs

## ECS Fargate instead of EC2
Fargate removes the need to manage, patch, or size EC2 instances.
The tradeoff is less direct control over underlying compute.

## ECS instead of EKS
ECS is simpler and faster to operationalize. EKS provides more
portability via Kubernetes but adds significant complexity for this
stage.

## ECR instead of Docker Hub
ECR is private by default, integrates natively with ECS via IAM,
and avoids rate limits. Docker Hub requires public images or paid
plans for private repos.

## Target type: IP instead of Instance
Fargate tasks don't run on a fixed EC2 instance. The ALB must
route to the task's IP directly.

## Public subnets for ECS tasks
Private subnets with NAT Gateway would be more secure but adds
~$32/month in cost. For this project, public subnets with locked-down
security groups is an acceptable tradeoff.
