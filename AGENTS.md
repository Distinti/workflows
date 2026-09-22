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

# workflows — reusable GitHub Actions (CI/CD) for GETProtocolLab services

Agent navigation lives in `AGENTS.md` files. Every `AGENTS.md` has a one-line
`CLAUDE.md` beside it containing `@AGENTS.md` — Claude Code reads `CLAUDE.md`
rather than `AGENTS.md`, so the stub imports its sibling automatically. Edit
the `AGENTS.md`; the stubs are plumbing and never carry content.

Three reusable workflows that back-end service repos call remotely via
`uses: GETProtocolLab/workflows/.github/workflows/<file>@master`. This repo
holds **no application code** — it _is_ the org's shared deploy pipeline.
Tracks **`master`** (not `development`).

**The single most important fact:** every consumer pins **`@master`** — there
are no version tags. A merge to `master` here changes CI/CD behaviour for
_every consumer repo's next workflow run_, org-wide, immediately. Treat any
change like a production deploy.

## Canonical human docs (do not modify)

- `README.md` — consumer prerequisites and the copy-paste integration snippet

## The three workflows

| File                            | Runs when (in a consumer repo)       | What it does                                                              |
| ------------------------------- | ------------------------------------ | ------------------------------------------------------------------------- |
| `.github/workflows/cd_push.yml` | every push                           | build + push image; on deploy branches, deploy via ArgoCD (see below)     |
| `.github/workflows/cd_pr.yml`   | every pull request                   | prettier-check `deploy/`, validate overlays with kuberc, post ArgoCD diff |
| `.github/workflows/lint.yml`    | pull request (this repo uses it too) | plain `prettier --check .`                                                |

Inputs to `.github/workflows/cd_push.yml`: `image` (required),
`argocd_app_name` (required), `registry` (optional, defaults to `ghcr.io`),
`aws_role_arn` (optional, defaults to the legacy Docker-cache role), and
`dockle_whitelist` (optional). For ECR, pass the registry host and the
repository-specific OIDC role created in `k8seks`; the build and Dockle jobs
then authenticate without a long-lived registry secret.

## What a deploy actually is (`.github/workflows/cd_push.yml`)

Deploying = pushing/merging to a deploy branch in a consumer repo. The branch
name selects the environment:

| Branch        | Cluster      | ArgoCD server                    | Overlay edited                 |
| ------------- | ------------ | -------------------------------- | ------------------------------ |
| `development` | euc1-testing | `argocd.euc1.t.get-protocol.dev` | `deploy/overlays/euc1-testing` |
| `master`      | euc1-staging | `argocd.euc1.s.get-protocol.dev` | `deploy/overlays/euc1-staging` |
| `production`  | euc1 (prod)  | `argocd.euc1.get-protocol.cloud` | `deploy/overlays/euc1`         |

Flow on a deploy-branch push:

1. **build** — `Distinti/docker-build-action` builds and pushes
   `<image>:<branch>` + `<image>:<sha>` to the configured registry (`REPO_AUTH`
   build-arg = `GH_ACCESS_TOKEN` for private packages). Transferred Distinti
   services should use ECR with a repository-specific AWS OIDC role; existing
   consumers remain on GHCR until they are transferred.
2. **deploy-argocd** — mints a GitHub-App token (so it can push to protected
   branches), runs `kustomize edit set image … :<sha>` inside the consumer's
   overlay directory, prettier-formats the kustomization, and **auto-commits
   it back to the branch** (`[skip ci] Deploy <branch> (<sha>) to <overlay>`).
   The repo itself is the GitOps source of truth — every deploy is a commit.
3. `argocd app sync -l app.kubernetes.io/name=<argocd_app_name> --prune`
   against the cluster's ArgoCD API.
4. ArgoCD sync then runs any Sync-hook Jobs first (e.g. entrails
   `migrate-db`; hook failure aborts the rollout), then rolls out the new
   Deployments.

On **non-deploy branches** step 2–3 are replaced by a **Dockle** container
security scan (`failure-threshold: fatal`; whitelist files via the
`dockle_whitelist` input).

## PR-side checks (`.github/workflows/cd_pr.yml`)

- `prettier --check ./deploy`
- **kuberc** validates all three overlays (`kubectl kustomize` → kuberc v5 via
  deno). Skip-lists for secrets/configmaps/services live inline — when a
  consumer adds a new external Secret or ConfigMap, it must be added there
  (see PR #16 for the pattern).
- **ArgoCD diff** posted as a PR comment via `GETProtocolLab/argocd-diff-action`
  — currently only for PRs targeting `master`/`production`; the
  `development`-base diff is disabled because the testing ArgoCD server is
  unreachable (HTTP 503) — see the inline comment in the job's `if:`.

## Consumers

13 service repos call these via a thin `.github/workflows/cd.yml` shim
(`secrets: inherit` + the two inputs): entrails, gcs, ggs, sphincter,
oesophagus, message-service, newskool, rumen, openhaggler, mollie-adapter,
stripe-adapter, stripeout, shitcoins. (entrails adds its own `build_dev` job;
gcs adds config-manager workflows.)

A consumer needs: a Dockerfile, a `./deploy` kustomize directory with the
three overlays, and registration in the k8seks ArgoCD apps (see `README.md`).

**Not routed through this repo:** vuejones/vuescera (own
`.github/workflows/cloudflare-deploy.yml` → Cloudflare Pages + S3), derriere
(own `.github/workflows/aws-deploy.yml` → S3), cloaca (build-only Go lib —
nothing to deploy), couche/colonoscopy (React-Native app builds), kurk and
dev-stack (nothing to deploy).

## Org-level secrets consumed (via `secrets: inherit`)

| Secret                                                 | Used for                                                        |
| ------------------------------------------------------ | --------------------------------------------------------------- |
| `GH_ACCESS_TOKEN`                                      | private-package auth inside docker builds                       |
| `ARGOCD_EUC1TESTING_API_AUTH` / `…STAGING…` / `…EUC1…` | ArgoCD API sync + diff, one per cluster                         |
| `WORKFLOWS_GITHUB_APP_ID` + `…_PRIVATE_KEY`            | GitHub App for deploy commits and private workflow dependencies |

## Notes for agents

- **`@master` is live org-wide the moment it merges.** To test a change
  safely, point one consumer repo's feature branch at your branch:
  `uses: GETProtocolLab/workflows/.github/workflows/cd_push.yml@<your-branch>`
  — non-deploy branches only build + Dockle-scan, so nothing ships.
- The `[self-hosted, ubuntu20.04-self]` runners are **ephemeral EC2 Spot
  instances**; sporadic "runner lost communication" / no-capacity failures are
  Spot reclaims — re-run the job before debugging anything.
- Sibling repos this one depends on: `GETProtocolLab/docker-build-action`,
  `GETProtocolLab/argocd-diff-action`.
- This repo's own PR CI is `prettier --check .` — keep this file
  prettier-formatted.

