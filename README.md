# AWS DevOps CI/CD Pipeline

<div align="center">

![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-%235835CC.svg?style=for-the-badge&logo=terraform&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-6DA55F?style=for-the-badge&logo=node.js&logoColor=white)

A production-grade CI/CD pipeline on AWS, provisioned entirely with Terraform. A `git push` to `main` builds a Docker image, pushes it to ECR, and deploys to ECS Fargate via a rolling update — with zero manual steps and zero downtime.

[Architecture](#architecture) • [Quick Start](#quick-start) • [Screenshots](#screenshots)

</div>

---

## Overview

This project provisions and operates a complete AWS deployment pipeline from scratch. The infrastructure is defined as code using Terraform, the application runs as a containerized Node.js service on ECS Fargate, and all deployments are triggered automatically via GitHub webhooks.

The goal was to build something that mirrors how engineering teams ship software in production — automated, auditable, and repeatable.

---

## Pipeline

```
git push origin main
        │
        ▼
  GitHub Webhook
        │
        ▼
┌───────────────────────────────────────────────┐
│              AWS CodePipeline                 │
│                                               │
│   SOURCE          BUILD           DEPLOY      │
│  ────────        ────────        ────────     │
│  Pulls code  →  CodeBuild   →  ECS Fargate    │
│  from GitHub    builds +        rolling       │
│  stores in S3   pushes image    update        │
│                 to ECR                        │
└───────────────────────────────────────────────┘
        │
        ▼
   Live in ~5 minutes
```

---

## Architecture

```
Internet
    │
    ▼
Internet Gateway
    │
┌───────────────────────────────────────────────────┐
│                   VPC (10.0.0.0/16)               │
│                                                   │
│   Public Subnets (us-east-1a / us-east-1b)        │
│   ┌─────────────────────────────────────────┐     │
│   │        Application Load Balancer        │     │
│   └──────────────────┬──────────────────────┘     │
│                      │                            │
│   Private Subnets (us-east-1a / us-east-1b)       │
│   ┌──────────────┐   │   ┌──────────────┐         │
│   │ ECS Task 1   │◀──┴──▶│ ECS Task 2   │         │
│   │ (Fargate)    │       │ (Fargate)    │         │
│   └──────┬───────┘       └──────┬───────┘         │
│          └──────────┬───────────┘                 │
│                     ▼                             │
│              NAT Gateway                          │
└───────────────────────────────────────────────────┘
               │              │
               ▼              ▼
            AWS ECR      CloudWatch
```

### Design Decisions

| Decision | Rationale |
|---|---|
| ECS Fargate | Serverless compute — no EC2 instances to patch, scale, or manage |
| Private subnets for tasks | Containers have no public IP; only reachable via ALB |
| NAT Gateway | Allows containers outbound access to ECR and CloudWatch without inbound exposure |
| Multi-AZ deployment | Traffic automatically shifts if one availability zone becomes unavailable |
| ALB health checks | Unhealthy targets are removed from rotation before new ones are registered |
| ECR over Docker Hub | Private registry within AWS network — no rate limits, no authentication latency |
| Terraform | Entire infrastructure reproducible from a single `terraform apply` |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Infrastructure as Code | Terraform 1.x |
| Cloud Provider | AWS (us-east-1) |
| Container Registry | Amazon ECR |
| Container Orchestration | Amazon ECS Fargate |
| Load Balancing | Application Load Balancer |
| CI/CD | AWS CodePipeline + CodeBuild |
| Observability | Amazon CloudWatch |
| Identity & Access | AWS IAM |
| Application Runtime | Node.js 18 + Express |
| Containerization | Docker (multi-stage build) |

---

## Project Structure

```
aws-devops/
├── src/
│   └── server.js               # Express server — serves app, exposes /health
├── public/
│   ├── index.html
│   ├── css/style.css
│   └── js/app.js
├── Dockerfile                  # Multi-stage build: builder + production image
├── buildspec.yml               # CodeBuild spec: login, build, tag, push, handoff
│
└── terraform/
    ├── main.tf                 # Provider configuration and data sources
    ├── variables.tf            # Input variable declarations
    ├── terraform.tfvars        # Variable values (gitignored)
    ├── vpc.tf                  # VPC, subnets, IGW, NAT Gateway, route tables
    ├── ecr.tf                  # ECR repository and lifecycle policy
    ├── iam.tf                  # IAM roles for ECS, CodeBuild, CodePipeline
    ├── ecs.tf                  # Cluster, task definition, service, ALB, security groups
    ├── codebuild.tf            # CodeBuild project configuration
    └── codepipeline.tf         # Pipeline stages, S3 artifact store, webhook
```

---

## Security

- **Least privilege IAM** — four separate roles scoped to exactly what each service requires: ECS execution, ECS task, CodeBuild, and CodePipeline
- **Network isolation** — ECS tasks run in private subnets with no public IP; security group rules restrict inbound traffic to ALB only
- **No stored credentials** — the EC2 ops machine authenticates via IAM instance role; containers authenticate via ECS task role
- **Non-root container** — application process runs as `nodeuser` (uid 1001)
- **ECR vulnerability scanning** — images are scanned on push using AWS native scanning

---

## Screenshots
<img width="1080" height="845" alt="Screenshot 2026-03-04 at 13 15 30" src="https://github.com/user-attachments/assets/bf5afeff-2b40-4b24-be4e-6a24158f0cdb" />
<img width="1080" height="845" alt="Screenshot 2026-03-04 at 13 15 38" src="https://github.com/user-attachments/assets/9fbdcfa7-a865-4615-a087-1b5b67cf878c" />
<img width="1080" height="845" alt="Screenshot 2026-03-04 at 13 15 49" src="https://github.com/user-attachments/assets/68171a90-cb9a-4807-8cd7-16d8c50fce2c" />
<img width="1080" height="845" alt="Screenshot 2026-03-04 at 13 16 09" src="https://github.com/user-attachments/assets/130c0329-d20f-48ee-8376-4a14323d930a" />
<img width="1080" height="647" alt="Screenshot 2026-03-04 at 13 16 21" src="https://github.com/user-attachments/assets/2205fcba-302c-4df5-ab98-f14e37222d30" />


### CI/CD Pipeline
<img width="1425" height="543" alt="Screenshot 2026-03-04 at 12 34 14" src="https://github.com/user-attachments/assets/62791c38-6802-4294-9091-d0ebcb188f5e" />


### ECS Fargate — 2 Tasks Running
<img width="1437" height="633" alt="Screenshot 2026-03-04 at 12 39 55" src="https://github.com/user-attachments/assets/239bc8d5-18d7-47d2-a57d-546047aacb97" />


### Live Application
<img width="1427" height="44" alt="Screenshot 2026-03-04 at 13 10 07" src="https://github.com/user-attachments/assets/80b53955-1547-4d47-a958-64259c14e427" />


---

## Quick Start

**Prerequisites**
- AWS account with admin access
- Terraform >= 1.0
- Git
- GitHub Personal Access Token with `repo` and `admin:repo_hook` scopes

**1. Clone**
```bash
git clone https://github.com/chahelpatwa/aws-devops.git
cd aws-devops/terraform
```

**2. Configure**
```bash
cp terraform.tfvars.example terraform.tfvars
```

```hcl
aws_region    = "us-east-1"
project_name  = "aws-devops"
github_repo   = "your-username/aws-devops"
github_branch = "main"
github_token  = "ghp_your_token_here"
```

**3. Deploy**
```bash
terraform init
terraform plan
terraform apply
```

**4. Trigger first pipeline run**
```bash
git commit --allow-empty -m "ci: trigger initial deployment"
git push origin main
```

The `alb_url` output from Terraform is your application endpoint. Pipeline completes in ~5 minutes.

---

## Cost

| Resource | Approximate Cost |
|---|---|
| ECS Fargate (2 tasks, 0.25 vCPU / 512 MB) | $0.02/hr |
| NAT Gateway | $0.045/hr + data transfer |
| Application Load Balancer | $0.008/hr |
| ECR storage | $0.10/GB/month |
| CodeBuild | First 100 min/month free |
| **Total** | **~$0.08/hr** |

```bash
# Tear down all resources when not in use
terraform destroy
```

---

## Key Engineering Notes

**Docker layer caching** — `package.json` is copied and `npm ci` runs before source code is copied. Dependency installation only re-executes when `package.json` changes, not on every code change. Build time drops from ~2 minutes to ~25 seconds after the first run.

**ECR Public Gallery as base image source** — `public.ecr.aws/docker/library/node:18-alpine` replaces `node:18-alpine` from Docker Hub. CodeBuild environments hit Docker Hub's anonymous pull rate limit quickly. ECR Public has no rate limits and resolves faster within AWS.

**`imagedefinitions.json` as the deploy handoff** — CodeBuild writes `[{"name":"aws-devops-app","imageUri":"...:<commit-hash>"}]` as a build artifact. CodePipeline's ECS deploy action reads this file to register a new task definition revision and update the service. No Lambda, no custom scripts.

**`iam:PassRole` scoping** — CodePipeline's IAM policy grants `iam:PassRole` scoped only to the two ECS roles it needs to pass when registering task definitions. Without this, ECS deploy fails with an authorization error even if all other permissions are correct.

**Health check grace period** — ECS container health checks use a 60-second `startPeriod`. This prevents the service from marking newly started containers as unhealthy before Node.js has finished initializing, which would cause a restart loop during deployments.

---

## License

[MIT](LICENSE) — Chahel Patwa
