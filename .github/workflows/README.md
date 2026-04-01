# Workflows

GitHub Actions CI/CD pipelines, orchestrated by [Atmos Pro](https://atmos.tools/pro).

| Workflow | Trigger | Action |
|----------|---------|--------|
| `main-branch.yaml` | Push to `main` | Build image → Describe affected → Atmos Pro triggers apply → Draft release |
| `feature-branch.yml` | PR with `deploy` label | Build image → Describe affected (preview) → Atmos Pro triggers plan |
| `atmos-terraform-plan.yaml` | Workflow dispatch (Atmos Pro) | Run `atmos terraform plan` and upload status |
| `atmos-terraform-apply.yaml` | Workflow dispatch (Atmos Pro) | Run `atmos terraform deploy` and upload status |
| `atmos-pro-list-deployments.yaml` | Daily schedule / manual | Sync instance inventory to Atmos Pro |
| `release.yaml` | Published release | Promote image → Deploy to staging and prod |
| `preview-cleanup.yml` | PR closed | Destroy preview environment |
| `validate.yml` | Pull request | Run validation checks |
| `labeler.yaml` | Pull request | Auto-label based on changed files |

## Main Branch Workflow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant GA as GitHub Actions
    participant ECR as AWS ECR
    participant AP as Atmos Pro
    participant ECS as AWS ECS

    Dev->>GH: Push to main
    GH->>GA: Trigger main-branch workflow
    GA->>ECR: Build & push Docker image
    ECR-->>GA: Image pushed (sha-xxx)
    GA->>AP: atmos describe affected --upload
    AP->>GA: Dispatch atmos-terraform-apply workflow
    GA->>ECS: atmos terraform deploy app -s dev
    ECS-->>AP: Upload status
    GA->>GH: Create draft release
```

## Feature Branch Workflow (Preview via Atmos Pro)

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant GA as GitHub Actions
    participant ECR as AWS ECR
    participant AP as Atmos Pro
    participant ECS as AWS ECS

    Dev->>GH: Open PR with 'deploy' label
    GH->>GA: Trigger feature-branch workflow
    GA->>ECR: Build & push Docker image
    ECR-->>GA: Image pushed
    GA->>AP: atmos describe affected --stack preview --upload
    AP->>GA: Dispatch atmos-terraform-plan workflow
    GA->>ECS: atmos terraform plan app -s preview
    ECS-->>AP: Upload plan status
    Note over Dev,AP: Developer reviews plan in Atmos Pro UI
    AP->>GA: Dispatch atmos-terraform-apply workflow (on approval)
    GA->>ECS: atmos terraform deploy app -s preview
    ECS-->>AP: Upload apply status
    Note over Dev,GH: PR closed
    GH->>GA: Trigger preview-cleanup workflow
    GA->>ECS: atmos terraform destroy app -s preview
```

## Release Workflow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant GA as GitHub Actions
    participant ECR as AWS ECR
    participant Atmos as Atmos CLI
    participant TF as OpenTofu
    participant ECS as AWS ECS

    Dev->>GH: Publish release (v1.0.0)
    GH->>GA: Trigger release workflow
    GA->>ECR: Promote image tag (sha-xxx → v1.0.0)
    GA->>Atmos: atmos terraform deploy app -s staging
    Atmos->>TF: tofu apply
    TF->>ECS: Update staging ECS service
    ECS-->>GA: Staging deployed
    GA->>Atmos: atmos terraform deploy app -s prod
    Atmos->>TF: tofu apply
    TF->>ECS: Update prod ECS service
    ECS-->>GA: Production deployed
```

## Environment Promotion Flow

```mermaid
graph LR
    A[Push to main] --> B[Build Image]
    B --> C[Describe Affected]
    C --> D[Atmos Pro: Apply to Dev]
    D --> E[Draft Release]
    E --> F{Publish Release}
    F --> G[Promote Image Tag]
    G --> H[Deploy to Staging]
    H --> I[Deploy to Production]

    J[PR with 'deploy' label] --> K[Build Image]
    K --> L[Describe Affected<br/>--stack preview]
    L --> M[Atmos Pro: Plan Preview]
    M --> N{Approve in Atmos Pro}
    N --> O[Atmos Pro: Apply Preview]
    O --> P{PR Merged/Closed}
    P --> Q[Cleanup Preview]
```
