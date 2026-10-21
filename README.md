# Task Tracker on AWS

A small Spring Boot REST API for managing tasks, deployed on AWS with infrastructure as code (CloudFormation), a CI/CD pipeline, auto scaling and monitoring. Built as an AWS bench-training project.

## What it does

A simple task API with create, read, update and delete operations.

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/tasks` | List all tasks |
| POST | `/api/tasks` | Create a task |
| GET | `/api/tasks/{id}` | Get one task |
| PUT | `/api/tasks/{id}` | Update title, description and status |
| DELETE | `/api/tasks/{id}` | Delete a task |
| GET | `/actuator/health` | Health check used by the load balancer |

## Tech stack

- Java 17, Spring Boot 3.2.3 (Web, Data JPA, Actuator)
- PostgreSQL
- Docker
- AWS: VPC, RDS, ECR, ECS Fargate, Application Load Balancer, Secrets Manager, CloudWatch, SNS, CodeBuild, CodePipeline, S3
- AWS CloudFormation (`template.yaml`)

## Architecture

- **Network:** one VPC with 2 public subnets (load balancer, tasks) and 2 private subnets (database).
- **Database:** RDS PostgreSQL in private subnets, not publicly accessible. The password is managed by AWS Secrets Manager and given to the container at start-up.
- **Compute:** the app runs as a container on ECS Fargate behind an Application Load Balancer.
- **Security groups:** internet to load balancer on port 80 only, load balancer to app on port 8080 only, app to database on port 5432 only.
- **Images:** stored in ECR with scan on push. Only the last 10 images are kept.
- **Auto scaling:** CPU target tracking at 60 percent. Maximum 2 tasks in `dev` and 6 in `prod`.

## CI/CD pipeline

CodePipeline runs three stages:

1. **Source:** `source.zip` uploaded to the artifact S3 bucket.
2. **Build:** CodeBuild builds the Docker image (see `buildspec.yml`) and pushes it to ECR.
3. **Deploy:** the ECS service is updated with the new image.

## Monitoring

- CloudWatch dashboard with ECS, load balancer and RDS metrics.
- Alarms with email alerts through SNS: ECS CPU above 80 percent, load balancer response time above 2 seconds, load balancer 5xx errors, and RDS CPU above 80 percent.
- Container logs in CloudWatch Logs, kept for 14 days.

## Project structure

```
.
├── src/main/java/com/tasktracker   # controller, entity, repository
├── Dockerfile                      # container image
├── buildspec.yml                   # CodeBuild steps
├── template.yaml                   # CloudFormation stack
└── pom.xml                         # Maven build
```

## Deploy

1. Create the stack. `AlertEmail` is required.

   ```bash
   aws cloudformation deploy \
     --template-file template.yaml \
     --stack-name task-tracker \
     --capabilities CAPABILITY_NAMED_IAM \
     --parameter-overrides AlertEmail=you@example.com EnvironmentName=dev
   ```

2. Zip the project as `source.zip` and upload it to the artifact bucket shown in the stack output `ArtifactBucketName`. This starts the pipeline.
3. Open the `ApplicationURL` stack output and call `/api/tasks`.

`ECSDesiredCount` defaults to 0 so the stack can be created before the first image exists.

## Clean up

Delete the stack to remove all resources and avoid cost:

```bash
aws cloudformation delete-stack --stack-name task-tracker
```
