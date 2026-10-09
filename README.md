# Distinti reusable workflows

Shared build, Kubernetes validation and ArgoCD deployment workflows.

## How to use

### cd (cd_pr/cd_push)

Consumers pin the moving `@master` ref. After a workflow merge, their next run
uses the new behavior. Pushes to `master` and `production` deploy to staging and production.
`development` and other branches build and run Dockle; testing deployment is
currently disabled. PR checks validate overlays; ArgoCD diff runs only for
`master`/`production` bases because the testing diff is disabled.

A consumer needs a Dockerfile, registration in the k8seks ArgoCD apps, and a
`deploy/` Kustomize directory with the expected environment overlays.
Read the workflow inputs before choosing a registry: ECR consumers also pass
`registry` and their repository-specific `aws_role_arn` for OIDC authentication.

To use the CD workflows include them in your repository like this.

```yaml
on:
  push:
  pull_request:

jobs:
  pr_workflow:
    concurrency:
      group: ${{ github.ref }}
      cancel-in-progress: true
    if: github.event_name == 'pull_request'
    uses: Distinti/workflows/.github/workflows/cd_pr.yml@master
    secrets: inherit

  push_workflow:
    if: github.event_name == 'push'
    uses: Distinti/workflows/.github/workflows/cd_push.yml@master
    secrets: inherit
    with:
      image: ghcr.io/distinti/your-repository
      argocd_app_name: your-argocd-app
```

## Lint

```bash
npx --yes prettier@3.6.2 --check .
```
