# atmos-pro-with-ecs <a href="https://cloudposse.com/"><img align="right" src="https://cloudposse.com/logo-300x69.svg" width="150" /></a>


[![Latest Release](https://img.shields.io/github/release/cloudposse-examples/atmos-pro-with-ecs.svg?style=for-the-badge)](https://github.com/cloudposse-examples/atmos-pro-with-ecs/releases/latest)
[![Last Updated](https://img.shields.io/github/last-commit/cloudposse-examples/atmos-pro-with-ecs/main?style=for-the-badge)](https://github.com/cloudposse-examples/atmos-pro-with-ecs/commits/main/)
[![Slack Community](https://slack.cloudposse.com/for-the-badge.svg)](https://slack.cloudposse.com)



Example **workload repository** demonstrating monorepo deployments of workloads to AWS ECS using GitHub Actions and [Atmos Pro](https://atmos.tools/pro), built on [Atmos](https://atmos.tools)' native CI integration with [OpenTofu](https://opentofu.org).

[Atmos](https://atmos.tools) provides a framework for managing infrastructure configuration across multiple environments with DRY stack configurations. [Atmos Pro](https://atmos.tools/pro) layers monorepo-aware CI/CD orchestration on top — detecting affected workloads on every change, dispatching targeted plan/apply workflows per stack, and providing drift detection — so teams get visibility and control over every workload deployment without maintaining bespoke pipeline logic.


## Introduction

### Workload

The example workload in this repository is a simple Go web server designed to demonstrate container deployment strategies. Each request increments a counter, and the background color is configurable - making it easy to visualize blue/green deployments and load balancing. The `/dashboard` endpoint displays a grid of auto-refreshing iframes to show traffic distribution across instances. See [`app/`](app/) for details.

### Infrastructure

```mermaid
graph TB
    subgraph "AWS Account"
        subgraph "VPC"
            subgraph "Private Subnets"
                ECS[ECS Fargate Service]
                EFS[(EFS Volume)]
            end
            subgraph "Public Subnets"
                ALB[Application Load Balancer]
            end
        end
        ECR[ECR Registry]
        R53[Route 53]
    end

    Internet((Internet)) --> R53
    R53 --> ALB
    ALB --> ECS
    ECS --> EFS
    ECR -.-> ECS

    subgraph "Dependencies (Pre-existing)"
        VPC_DEP[VPC Component]
        ECS_DEP[ECS Cluster Component]
        EFS_DEP[EFS Component]
    end

    VPC_DEP -.->|vpc_id, subnet_ids| ECS
    ECS_DEP -.->|cluster_arn, alb_listener_arn| ECS
    EFS_DEP -.->|efs_id| EFS
```

This project uses:

- **Workload repository pattern** - One repo per deployable workload, deployed across environments via Atmos stacks
- **[Atmos](https://atmos.tools)** - Configuration orchestration and stack management
- **[OpenTofu](https://opentofu.org)** - Infrastructure as Code (Terraform-compatible)
- **AWS ECS Fargate** - Serverless container orchestration
- **AWS ECR** - Container image registry
- **AWS EFS** - Persistent file storage (optional)



## Usage

### Local Development

Run the application locally using Podman Compose:

```bash
# Start the app locally (builds and runs on http://localhost:8080)
atmos up

# Stop the app
atmos down
```

### CI/CD Workflows

These workflows implement the workload deployment pipeline: build the image, run `atmos describe affected` to detect changed workloads, then let Atmos Pro dispatch plan/apply per affected stack. See [`.github/workflows/`](.github/workflows/) for detailed workflow diagrams.

| Workflow | Trigger | Action |
|----------|---------|--------|
| `feature-branch.yml` | Pull request, merge queue | Build image → Run tests → Describe affected → Atmos Pro plans preview (`deploy` label) or applies dev (merge queue) |
| `validate.yml` | Pull request, merge queue | Lint CODEOWNERS |
| `main-branch.yaml` | Push to `main` | Update draft release notes |
| `release.yaml` | Published release, manual dispatch | Promote image → Deploy to staging and/or prod |
| `atmos-terraform-plan.yaml` | Workflow dispatch (Atmos Pro) | Run `atmos terraform plan` and upload status |
| `atmos-terraform-apply.yaml` | Workflow dispatch (Atmos Pro) | Run `atmos terraform deploy` and upload status |
| `atmos-pro-list-deployments.yaml` | Daily schedule / manual | Sync instance inventory to Atmos Pro |
| `preview-cleanup.yml` | PR closed | Destroy preview environment |

### Deployment

#### Prerequisites

- [Atmos](https://atmos.tools/install) installed
- [OpenTofu](https://opentofu.org/docs/intro/install/) installed
- AWS credentials configured

#### Infrastructure Dependencies

Before deploying, you must have the following infrastructure deployed:

1. **VPC** - With public/private subnets
2. **ECS Cluster** - With an Application Load Balancer (ALB) and DNS records configured
3. **EFS** (optional) - For persistent storage volumes

Then configure the dependencies in `terraform/stacks/`. You have two options:

**Option 1: Use `!terraform.state` (recommended)**

Update the dependency configurations in `terraform/stacks/deps/` to point to your infrastructure's remote state:
- `deps/vpc.yaml` - VPC component remote state location
- `deps/ecs.yaml` - ECS cluster component remote state location
- `deps/efs.yaml` - EFS component remote state location (if using volumes)

**Option 2: Hardcode values (brownfield)**

Replace the `!terraform.state` lookups in `terraform/stacks/defaults/app.yaml` with hardcoded values for your infrastructure. See [`terraform/stacks/defaults/README.md`](terraform/stacks/defaults/README.md) for required variables and a complete example.

#### Local Deployment

```bash
# Deploy to dev environment
atmos terraform deploy app -s dev

# Deploy to staging
atmos terraform deploy app -s staging

# Deploy to production
atmos terraform deploy app -s prod
```

#### CI/CD Deployment (via Atmos Pro)

1. Open a PR with `deploy` label → builds image → `describe-affected --stack preview` uploads to Atmos Pro → Atmos Pro dispatches plan workflow for preview
2. Click "Merge when ready" → merge queue runs build/test/`describe-affected` on the queue commit → Atmos Pro dispatches apply for dev → check-suite status gates the queue → on success, fast-forward to `main`
3. Create a GitHub release → promotes image → deploys to staging and prod (direct)
4. Run `release.yaml` via workflow dispatch with a `tag` and `environment` to roll back or hotfix without cutting a new release

### Configuration

Stack configurations are in `terraform/stacks/`. Each environment imports shared defaults and specifies environment-specific settings:

```yaml
# terraform/stacks/dev.yaml
import:
  - _default.yaml
  - defaults/app.yaml
  - deps/*

vars:
  stage: dev
```

Container configuration is defined in `terraform/stacks/defaults/app.yaml` and can be customized per environment.

### Repository Structure

```
.
├── app/                       # Workload source (Go web server)
│   ├── main.go                # Web server
│   ├── Dockerfile             # Multi-stage container build
│   ├── public/                # Static HTML assets
│   ├── rootfs/                # Container filesystem overlay
│   └── test/                  # Local development (docker-compose)
├── atmos.yaml                 # Atmos configuration
├── .atmos.d/                  # Atmos custom commands
├── terraform/
│   ├── components/            # Terraform/OpenTofu modules
│   │   └── ecs-task/          # Workload's Terraform component (ECS task definition)
│   └── stacks/                # Environment configurations
│       ├── defaults/          # Shared component config
│       ├── deps/              # Dependency references
│       ├── dev.yaml
│       ├── staging.yaml
│       ├── prod.yaml
│       └── preview.yaml
└── .github/
    ├── workflows/             # CI/CD pipelines
    ├── README.yaml            # README source
    └── README.md              # Generated README
```

### Building Documentation

To regenerate the README from this file, run:

```bash
atmos docs generate readme
```







## Related Projects

Check out these related projects.

- [Atmos](https://atmos.tools) - Universal Tool for DevOps and Cloud Automation
- [terraform-aws-components](https://github.com/cloudposse/terraform-aws-components) - Opinionated, self-contained Terraform root modules for Cloud Posse reference architecture



## Slack Community

Join our [Open Source Community](https://slack.cloudposse.com) on Slack. It's **FREE** for everyone! Our "SweetOps" community is where you get to talk with others who share a similar vision for how to rollout and manage infrastructure. This is the best place to talk shop, ask questions, solicit feedback, and work together as a community to build totally *sweet* infrastructure.

## Newsletter

Sign up for [our newsletter](https://cpco.io/newsletter) and join 3,000+ DevOps engineers, CTOs, and founders who get insider access to the latest DevOps trends, so you can always stay in the know. Dropped straight into your Inbox every week — and usually a 5-minute read.

## Office Hours <a href="https://cloudposse.com/office-hours"><img src="https://img.cloudposse.com/fit-in/200x200/https://cloudposse.com/wp-content/uploads/2019/08/Powered-by-Zoom.png" align="right" /></a>

[Join us every Wednesday via Zoom](https://cloudposse.com/office-hours) for your weekly dose of insider DevOps trends, AWS news and Terraform insights, all sourced from our SweetOps community, plus a _live Q&A_ that you can't find anywhere else. It's **FREE** for everyone!

## License

<a href="https://opensource.org/licenses/Apache-2.0"><img src="https://img.shields.io/badge/License-Apache%202.0-blue.svg?style=for-the-badge" alt="License"></a>

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for full details.

## Trademarks

All other trademarks referenced herein are the property of their respective owners.

---
Copyright © 2017-2026 [Cloud Posse, LLC](https://cloudposse.com)
