# Deployment

## Prerequisites

Before deploying, you must have the following infrastructure deployed:

1. **VPC** - With public/private subnets
2. **ECS Cluster** - With ALB and DNS records configured
3. **EFS** (optional) - For persistent storage volumes

Then configure dependencies using one of two approaches:

**Option 1: Use `!terraform.state`** - Update `terraform/stacks/deps/*.yaml` to point to your infrastructure's remote state

**Option 2: Hardcode values** - Replace `!terraform.state` lookups in `defaults/app.yaml` with hardcoded values (see `terraform/stacks/defaults/README.md`)

## Dependencies

The app component depends on external infrastructure via `!terraform.state` lookups in `defaults/app.yaml`:

| Dependency | Source | Provides |
|------------|--------|----------|
| `vpc` | `deps/vpc.yaml` | VPC configuration |
| `ecs/cluster` | `deps/ecs.yaml` | ECS cluster, ALB, DNS records |
| `efs` | `deps/efs.yaml` | EFS filesystem ID for volumes |

## Troubleshooting

If deployments fail, check that:
1. The `deps_stage` variable matches the environment where dependencies exist
2. Remote state is accessible (S3 bucket configured in `deps/*.yaml`)
3. IAM role for state access has appropriate permissions
4. The prerequisite components are deployed and their outputs match what `defaults/app.yaml` expects

## CI/CD Pipeline (Atmos Pro)

- Push to `main` → build image → `describe-affected --upload` → Atmos Pro dispatches apply for dev → draft release
- PR with `deploy` label → build image → `describe-affected --stack preview --upload` → Atmos Pro dispatches plan → approve in Atmos Pro UI → apply
- Publish release → promote image, deploy to staging/prod (direct)
- PR close → cleanup preview

The plan/apply workflows (`atmos-terraform-plan.yaml`, `atmos-terraform-apply.yaml`) are dispatched by Atmos Pro. They reconstruct `APP_IMAGE` from the SHA input (the image was already built and tagged as `sha-<commit>` in the triggering workflow).

## Atmos Pro Configuration

- Stack settings are in `terraform/stacks/defaults/atmos-pro.yaml` (imported via `_default.yaml`)
- Configures PR plan/apply workflows, drift detection, and release workflows
- Requires `ATMOS_PRO_WORKSPACE_ID` variable and Atmos Pro GitHub App installed on the repository

## Naming Convention

Labels: `namespace-tenant-environment-stage-name`
Example: `cplive-plat-ue2-dev-app`
