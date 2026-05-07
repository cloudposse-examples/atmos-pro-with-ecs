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

Deployments to dev and preview are orchestrated by Atmos Pro. Nothing in CI calls `atmos terraform deploy` directly for dev/preview — every plan/apply goes through Atmos Pro's `workflow_dispatch` of `atmos-terraform-plan.yaml` / `atmos-terraform-apply.yaml`.

1. CI builds the Docker image and tags it with the commit SHA
2. `atmos describe affected --upload` sends affected stacks to Atmos Pro
3. Atmos Pro dispatches `atmos-terraform-plan.yaml` or `atmos-terraform-apply.yaml` via workflow dispatch
4. Plan/apply workflows reconstruct `APP_IMAGE` from the SHA input and run `atmos terraform … --upload-status` so Atmos Pro reports a check-suite status back on the commit

Trigger contexts:

- **PR with `deploy` label** (`pull_request`): `feature-branch.yml` runs `describe affected --stack preview`, Atmos Pro dispatches plan-then-apply for preview.
- **Merge queue** (`merge_group`): `feature-branch.yml` runs `describe affected` (no stack filter), Atmos Pro dispatches apply for dev. The Atmos Pro check-suite status on the queue commit is the required check that gates the merge — a broken Terraform apply rejects the PR from the queue before it reaches `main`.
- **Push to `main`**: `main-branch.yaml` only updates the draft release. The queue commit was already built/tested/dev-applied, so nothing else runs here.
- **Release published** (or `release.yaml` `workflow_dispatch`): promotes the image and runs direct `atmos terraform deploy` for staging and prod. Direct deploys for these environments preserve queue throughput (env protection rules would otherwise cause head-of-line blocking). `workflow_dispatch` accepts a `tag` and `environment` for rollback / hotfix / selective redeploy without cutting a new release.

## Project Structure

- `app/` - Go web application (see `app/README.md`)
- `terraform/components/ecs-task/` - Main Terraform component
- `terraform/stacks/` - Environment configurations
- `terraform/stacks/defaults/atmos-pro.yaml` - Atmos Pro workflow settings
- `.atmos.d/commands.yaml` - Custom Atmos commands
- `.github/workflows/` - CI/CD pipelines (see workflows README)
