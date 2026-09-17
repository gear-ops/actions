# gear-ops/actions

Reusable GitHub Actions workflows.

## build-deploy.yaml

One call per application, one job per component, one commit, one sync.

```text
deploy        plan → build → bump → sync → cleanup → report
gitops only   plan → build → bump ───────→ cleanup → report
build only    plan → build ──────────────→ cleanup → report
```

| mode | `charts_repo` | `argocd_server` | you get |
| --- | --- | --- | --- |
| deploy | set | set | images, a commit in the GitOps repo, a synced Application |
| gitops only | set | empty | images and the commit, rollout is someone else's job |
| build only | empty | ignored | images only, and no `CHARTS_REPO_TOKEN` |

| job | runs | when |
| --- | --- | --- |
| `plan` | once | always |
| `build` | per component | a changed file matches its `paths`, and in deploy mode an Application exists |
| `bump` | once | `charts_repo` set, every build succeeded |
| `sync` | once | `argocd_server` set, bump succeeded |
| `cleanup` | per component | `cleanup_enabled`, the last stage of the mode succeeded |
| `report` | once | always |
| `notify` | once | `plan`, `build`, `bump` or `sync` failed |

### Branch to env

| branch | env |
| --- | --- |
| `main`, `master`, `release` | `prod` |
| anything else | `stg` |

Tags and other refs are ignored. Re-map with `env_prod_branches`, `env_prod` and
`env_default`; a `workflow_dispatch` run can force `env` directly.

### What it expects from your setup

| piece | needed for | shape |
| --- | --- | --- |
| GitOps repo `charts_repo` | deploy, gitops only | `charts/<team>/<app>/values/<env>.yaml`, each component at `.components.<name>.containers.main.image` with `repository` and `tag`; other layouts via `charts_values_path` |
| ArgoCD Applications | deploy | labels `app`, `env`, `team`, or one name in `argocd_app` |
| Vault | build-args | a KV-v2 mount and a JWT auth mount sharing the name in `vault_mount_args`, with a role that trusts this repository |
| Registry | always | `ghcr.io` by default, anything else via `docker_registry` plus `DOCKER_REGISTRY_PASSWORD` |

Vault is orthogonal to the three modes: with `vault_server` empty the step is
skipped, build-args come only from `docker_build_args`, and the caller does not
need `id-token: write`.

### Caller

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
    paths-ignore: ['.github/**', '**.md']
  workflow_dispatch:
    inputs:
      env:             { type: choice, options: [prod], required: false }
      deploy_indexer:  { type: boolean, default: true }   # deploy_<component>
      deploy_frontend: { type: boolean, default: true }

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

permissions:
  contents: write
  packages: write
  actions: write
  id-token: write      # only for build-args from Vault

jobs:
  deploy:
    uses: gear-ops/actions/.github/workflows/build-deploy.yaml@main
    with:
      team: example-team
      app: example-app
      env: ${{ inputs.env }}
      components: |
        - name: indexer
          docker_context: indexer
          chart_init_containers: migrate
          chart_components_extra: graphql
          paths: indexer/**
        - name: frontend
          docker_context: frontend
          paths: frontend/**
    secrets:
      CHARTS_REPO_TOKEN: ${{ secrets.CHARTS_REPO_TOKEN }}
      ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}
```

A single-component application names its component `main`; its dispatch flag is
then just `deploy`.

### Component keys

| key | needed | default | what it does |
| --- | --- | --- | --- |
| `name` | always | | `.components.<name>` in chart values; `main` for a single-component app |
| `docker_context` | optional | `.` | build context |
| `docker_file` | optional | `Dockerfile` | Dockerfile path inside the context |
| `docker_registry` | optional | see below | image without tag |
| `docker_build_args` | optional | | multiline `KEY=VALUE`, merged under the ones from Vault |
| `paths` | optional | | multiline globs; the component builds only if a changed file matches |
| `chart_init_containers` | optional | | initContainers that get the same image |
| `chart_components_extra` | optional | | other `.components.*` that get the same image |

Default image name:

| component | image |
| --- | --- |
| `main` | `ghcr.io/<owner>/<repo>` |
| named after the repo | `ghcr.io/<owner>/<repo>` |
| anything else | `ghcr.io/<owner>/<repo>-<name>` |

Two components resolving to the same image would overwrite each other, so `plan`
refuses to start: give one its own `docker_registry`, or merge them with
`chart_components_extra`.

### Variables and secrets on the calling repo

| variable | needed | what it is |
| --- | --- | --- |
| `ARGOCD_SERVER` | to deploy | ArgoCD host, e.g. `argocd.example.com` |
| `VAULT_SERVER` | for build-args | Vault URL, e.g. `https://vault.example.com` |

| secret | needed | what it is |
| --- | --- | --- |
| `CHARTS_REPO_TOKEN` | to deploy | token with `contents:write` on `charts_repo` |
| `ARGOCD_AUTH_TOKEN` | to deploy | ArgoCD API token allowed to sync those Applications |
| `DOCKER_REGISTRY_PASSWORD` | optional | only for a registry other than `ghcr.io` |
| `TELEGRAM_TOKEN`, `TELEGRAM_CHAT`, `TELEGRAM_THREAD` | optional | failure alerts |

Both variables only fill the defaults of the `argocd_server` and `vault_server`
inputs, so a caller may pass those directly instead.

### What bump writes

One tag per run, computed once in `plan` as `<UTC timestamp>_<branch>_<short sha>`:

```yaml
# charts/example-team/example-app/values/prod.yaml
components:
  indexer:                       # name
    containers:
      main:
        image:
          repository: ghcr.io/example-org/example-app-indexer
          tag: 2026.09.17_10.15_main_1a2b3c4
    initContainers:
      migrate:                   # chart_init_containers
        image: { repository: same, tag: same }
  graphql:                       # chart_components_extra
    containers:
      main:
        image: { repository: same, tag: same }
```

The commit is plain `git`, so the token never reaches a third-party action.

Build-args are read from `<vault_mount_args>/<team>/<app>[/<component>]/<env>`
over GitHub OIDC. A missing path means the component has no build-args there;
any other Vault error fails the build.

### Image visibility

| `docker_image_visibility` | effect after each push to `ghcr.io` |
| --- | --- |
| `private` (default) | the build fails if the package ended up public |
| `public` | the build fails if the package ended up private |
| empty | no check |

It is a check, not a setting. An image pushed with `GITHUB_TOKEN` inherits the
visibility of its repository, and neither the REST API nor GraphQL can change it
afterwards, so a package is fixed once on its settings page.

### Cleanup

Keeps `cleanup_keep` versions per image, protecting `latest` and everything in
`docker_tags_extra`. Only images built in that run are pruned, so a component
skipped by its `paths` filter keeps its old versions until it builds again.

## Actions used

| action | why |
| --- | --- |
| `dorny/paths-filter` | per-component `paths` filter against the push diff |
| `docker/setup-buildx-action`, `docker/login-action`, `docker/build-push-action` | build and push |
| `actions/upload-artifact`, `actions/download-artifact` | carry each component's result into the single bump job |
| `snok/container-retention-policy` | GHCR prune; unlike `actions/delete-package-versions` it filters by image tag, so `latest` stays protected |
| `appleboy/telegram-action` | failure notification |
