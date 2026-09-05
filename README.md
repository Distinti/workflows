# Description

> **Deprecated:** do not add new consumers. Distinti backend services are being
> migrated to the private `Distinti/backend-workflows` release pipeline. This
> repository remains operational only for existing callers during the staged
> migration and will be archived after the organization-wide caller audit is
> clean.

Repository for general workflows in the getprotocolab repo that handles CI/CD tasks

## How to use

### cd (cd_pr/cd_push)

The CD workflows will build and deploy your repo to the guts argocd clusters. It will also preview your k8s changes when making a PR to the deploy branches.

This workflows only works if your repo includes a Dockerfile, has been included in the k8seks argocd apps and contains a `./deploy` kustomize directory with k8s config.

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
    uses: GETProtocolLab/workflows/.github/workflows/cd_pr.yml@master
    secrets: inherit

  push_workflow:
    if: github.event_name == 'push'
    uses: GETProtocolLab/workflows/.github/workflows/cd_push.yml@master
    secrets: inherit
    with:
      image: ghcr.io/getprotocollab/{{ your_repo }}
      argocd_app_name: { { your_argocd_app_name } }
```

## Lint

```bash
docker run --rm -v $(pwd):/work tmknom/prettier --write .
```
