> ## AI Policy — READ FIRST
>
> Every existing file in this repository was authored by humans. AI-generated
> content is restricted to files named `AGENTS.md` (this file and nested ones)
> and to the one-line `CLAUDE.md` pointer stubs that sit beside them.
>
> Agents working in this repo MUST:
>
> - Treat all non-`AGENTS.md` files as canonical human work
> - **Never modify** `README.md` or any other human doc without explicit human
>   instruction
> - Defer to human docs over `AGENTS.md` on conflict — `AGENTS.md` is a _map_,
>   the human docs are the _territory_
> - When proposing changes that touch human docs, surface that in the plan and
>   request explicit confirmation

# Distinti reusable GitHub Actions

[README.md](README.md) owns consumer setup. This repository contains shared CI/CD,
not application code. PR base is `master`; consumers use a moving `@master` ref.
A workflow change affects their next run after merge. Inspect actual caller refs
rather than assuming all consumers have transferred or use the same registry.

| Workflow                        | Responsibility                                                              |
| ------------------------------- | --------------------------------------------------------------------------- |
| `.github/workflows/cd_push.yml` | Build/push image; deploy on release branches, Dockle scan on other branches |
| `.github/workflows/cd_pr.yml`   | Format/validate deploy overlays and post eligible ArgoCD diffs              |
| `.github/workflows/lint.yml`    | This repository's existing Prettier check                                   |

Only `master` and `production` deploy, to staging and production respectively.
`development` builds/scans through Dockle. Testing deployment and PR ArgoCD
diffs are disabled in the current job conditions.
Deploy writes the image SHA back to the consumer's Kustomize overlay with a
GitHub App token, then syncs ArgoCD with prune. Sync hooks can block rollout.
Read the workflow's current inputs, permissions, job conditions and cluster
mapping before editing. Do not infer a successful deployment from the Git commit.

ECR callers pass their registry and repository-specific AWS OIDC role; legacy
callers can still use GHCR. Private build auth and ArgoCD/GitHub App secrets are
owned by the consuming repository/organization. Never print or commit them.

For a workflow behavior change, a consumer feature branch can call the proposed
workflow branch. Check its triggers and environment first; this docs cleanup
has no authority to change callers or deploy. Use the existing format check:
`npx --yes prettier@3.6.2 --check .`.
