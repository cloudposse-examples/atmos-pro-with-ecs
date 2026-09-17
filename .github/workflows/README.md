# Workflows

The Voyager AWS demo was decommissioned in September 2026 after its original
contract work ended. Cloud deployment, preview, release-promotion, and Atmos
Pro dispatch workflows were removed so the retired service cannot be recreated
by a pull request, merge, release, schedule, or manual workflow dispatch.

The remaining workflows provide repository maintenance only:

| Workflow | Trigger | Action |
|----------|---------|--------|
| `validate.yml` | Pull request, merge queue | Validate repository metadata |
| `labeler.yaml` | Pull request | Apply repository labels |
| `main-branch.yaml` | Push to `main` | Update draft release notes |

The application and Terraform component remain as historical reference. The
`app` stack definition is marked abstract and Atmos Pro is disabled. Restoring
an AWS deployment requires an intentional code change that reverses those two
guards and reintroduces reviewed deployment workflows.
