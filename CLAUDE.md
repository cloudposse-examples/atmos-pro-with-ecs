# CLAUDE.md

Example containerized Go web application deployed to AWS ECS Fargate using Atmos, Atmos Pro, and OpenTofu.

## Quick Reference

```bash
# Local development
atmos up                                # Start app locally with Podman Compose
atmos down                              # Stop local app

# Deploy with Atmos (local)
atmos terraform plan app -s dev         # Plan changes for dev
atmos terraform deploy app -s dev       # Deploy to dev
atmos terraform deploy app -s staging   # Deploy to staging
atmos terraform deploy app -s prod      # Deploy to production

# Get deployment URL
atmos terraform output app -s dev --skip-init -- -raw url
```

## CI/CD with Atmos Pro

Deployments to dev and preview are orchestrated by Atmos Pro:

1. CI builds the Docker image and tags it with the commit SHA
2. `atmos describe affected --upload` sends affected stacks to Atmos Pro
3. Atmos Pro dispatches `atmos-terraform-plan.yaml` or `atmos-terraform-apply.yaml` via workflow dispatch
4. Plan/apply workflows reconstruct `APP_IMAGE` from the SHA input

Feature branches filter to `--stack preview` only. Main branch uploads all affected stacks.

## Project Structure

- `app/` - Go web application (see `app/README.md`)
- `terraform/components/ecs-task/` - Main Terraform component
- `terraform/stacks/` - Environment configurations
- `terraform/stacks/defaults/atmos-pro.yaml` - Atmos Pro workflow settings
- `.atmos.d/commands.yaml` - Custom Atmos commands
- `.github/workflows/` - CI/CD pipelines (see workflows README)
