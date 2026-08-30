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
<img width="1920" height="1080" alt="Screenshot from 2026-08-30 22-33-42" src="https://github.com/user-attachments/assets/0ef22c9c-c35b-487d-9ac8-3359cc187ae9" />


**Jenkins pipeline run — success**
<img width="1920" height="1080" alt="Screenshot from 2026-08-30 22-33-49" src="https://github.com/user-attachments/assets/b5a76bbd-cd36-493f-8f10-4b640e1c81d3" />

**Image pushed to Amazon ECR**
<img width="1920" height="1080" alt="Screenshot from 2026-08-30 22-32-52" src="https://github.com/user-attachments/assets/d34a7579-b04a-4aa4-884e-49e900422013" />

**ECS service deployment — new task rolled out**
<img width="1920" height="1080" alt="Screenshot from 2026-08-30 22-31-46" src="https://github.com/user-attachments/assets/7918c4e4-3a41-4888-8a16-2fe4855c948a" />

**ALB target group — target healthy**
<img width="1920" height="1080" alt="Screenshot from 2026-08-30 22-32-05" src="https://github.com/user-attachments/assets/07e77719-1634-4c35-9c5d-936b82521de6" />

**Application Load Balancer configuration**
<img width="1920" height="1080" alt="Screenshot from 2026-08-30 22-31-57" src="https://github.com/user-attachments/assets/08ac0ccf-b8fb-44d4-b248-4cfee66af8d4" />

**Live site served through the ALB**
<img width="1920" height="1080" alt="Screenshot from 2026-08-30 22-34-54" src="https://github.com/user-attachments/assets/09daab09-7a12-4950-a395-14376ad6b13d" />
