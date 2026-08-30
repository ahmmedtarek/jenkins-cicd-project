# CI/CD Pipeline for Dockerized Nginx App on AWS ECS

A fully automated CI/CD pipeline: every push to GitHub triggers a Jenkins pipeline that builds a Docker image, tests it, pushes it to Amazon ECR, and deploys it to Amazon ECS (Fargate) behind an Application Load Balancer — with zero manual steps.

## Architecture

```
GitHub push
     │
     ▼
Jenkins (on EC2)
     │  1. Checkout
     │  2. Build Docker image
     │  3. Test (spin up container, curl health check)
     │  4. Push image to ECR
     │  5. Register new ECS task definition revision
     │  6. Update ECS service (force new deployment)
     ▼
Amazon ECS (Fargate)
     │  Rolling deployment: old task drains, new task starts
     ▼
Application Load Balancer → Target Group
     │  Health checks against the new task
     ▼
Live traffic served from the new version
```

## Tech stack

- **Source control:** GitHub (webhook-triggered)
- **CI/CD orchestration:** Jenkins (Groovy declarative pipeline) on an EC2 instance
- **Containerization:** Docker (Nginx serving a static `index.html`)
- **Registry:** Amazon ECR
- **Orchestration:** Amazon ECS on Fargate
- **Load balancing:** Application Load Balancer + target group health checks
- **Infra tools:** AWS CLI, `jq` for task definition manipulation

## Pipeline stages

| Stage | What happens |
|---|---|
| Checkout | Jenkins pulls the latest code from the `main` branch |
| Build | Builds a Docker image tagged with the Jenkins build number |
| Test | Runs the image as a temporary container and `curl`s it to confirm it serves traffic before anything gets pushed |
| Push to ECR | Authenticates to ECR and pushes the tagged image |
| Deploy to ECS | Fetches the current task definition, swaps in the new image, registers a new revision, updates the ECS service, and waits for the deployment to stabilize |

## Challenges faced and how they were solved

Building this pipeline surfaced a few real ECS/ALB deployment issues worth documenting:

**1. New task registered but never received traffic (stuck as "unused" in the target group)**
With `desiredCount = 1` and default deployment settings, ECS tried to run the old and new task simultaneously. Fixed by tuning `minimumHealthyPercent` / `maximumPercent` and understanding how Availability Zone rebalancing constrains those values.

**2. New task killed right before becoming healthy**
The target group's healthy threshold (5 consecutive checks × 30s interval = 150 seconds) exceeded the service's `healthCheckGracePeriodSeconds` (set to just 2 seconds). ECS's deployment circuit breaker killed the task before the ALB had a chance to mark it healthy. Fixed by raising the grace period and lowering the healthy threshold — the deployment ran clean immediately after.

## Screenshots

**Jenkins pipeline run — start**
![Jenkins console start](./01-jenkins-console-start.png)

**Jenkins pipeline run — success**
![Jenkins pipeline success](./02-jenkins-pipeline-success.png)

**Image pushed to Amazon ECR**
![ECR image pushed](./03-ecr-image-pushed.png)

**ECS service deployment — new task rolled out**
![ECS service deployment](./04-ecs-service-deployment.png)

**ALB target group — target healthy**
![Target group healthy](./05-target-group-healthy.png)

**Application Load Balancer configuration**
![ALB listener](./06-alb-listener.png)

**Live site served through the ALB**
![Live site](./07-live-site.png)

## What's next

- [ ] Move infrastructure (VPC, ECS cluster, ALB, security groups) to Terraform
- [ ] Store AWS credentials via Jenkins Credentials Plugin instead of inline
- [ ] Add HTTPS via an ACM certificate on the ALB
- [ ] Add container image scanning (Trivy or ECR scan-on-push)
- [ ] Add Slack/email notifications on deploy success or failure
