# gear-ops/actions

Reusable GitHub Actions workflows.

## build-deploy.yaml

Builds a Docker image per component, writes the new tags into a GitOps repo and
syncs the ArgoCD Application.

### Quick start

Build only, the smallest caller, with none of the variables below set:

```yaml
name: Build

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  actions: read
  id-token: write

jobs:
  build:
    uses: gear-ops/actions/.github/workflows/build-deploy.yaml@main
    with:
      app: example-app
      components: |
        - name: main
```

It pushes `ghcr.io/<owner>/<repo>:<tag>`; a public repository also passes
`docker_image_visibility: public`. All four `permissions` are required in every
mode: a called workflow gets no more than its caller grants.

Deploy, with two components and a manual run:

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
    paths-ignore: ['.github/**', '**.md']
  workflow_dispatch:
    inputs:
      env:             { type: choice, options: [prod, stg], required: false }
      deploy_backend:  { type: boolean, default: true }   # deploy_<component>
      deploy_frontend: { type: boolean, default: true }

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

permissions:
  contents: read
  packages: write
  actions: read
  id-token: write

jobs:
  deploy:
    uses: gear-ops/actions/.github/workflows/build-deploy.yaml@main
    with:
      team: example-team
      app: example-app
      env: ${{ inputs.env }}
      components: |
        - name: backend
          docker_context: backend
          chart_init_containers: migrations
          chart_components_extra: worker
          paths: backend/**
        - name: frontend
          docker_context: frontend
          paths: frontend/**
    secrets:
      CHARTS_REPO_TOKEN: ${{ secrets.CHARTS_REPO_TOKEN }}
      ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}
```

`charts_repo` and `argocd_server` come from the variables below.

### Modes

| mode | set | you get |
| --- | --- | --- |
| build only | nothing | images |
| gitops only | `charts_repo` | images and a commit in the GitOps repo |
| deploy | `charts_repo`, `argocd_server` | images, the commit and a synced Application |

Build-args from Vault work in any mode once `vault_server` is set.

### Variables and secrets

Set variables on the organization or the repository; a repository variable
overrides an organization one, and an input of the same name in lower case
overrides both.

| variable | needed for | example |
| --- | --- | --- |
| `TEAM` | GitOps repo, Vault | `example-team` |
| `CHARTS_REPO` | gitops only, deploy | `example-org/gitops` |
| `ARGOCD_SERVER` | deploy | `argocd.example.com` |
| `VAULT_SERVER` | Vault | `https://vault.example.com` |
| `VAULT_AUTH_PATH` | Vault | `github` |
| `VAULT_ROLE` | Vault | `example-org` |
| `VAULT_MOUNT_CI` | Vault | `example-org-ci` |
| `VAULT_MOUNT_RUNTIME` | optional, a link in the report | `example-org-runtime` |

| secret | needed for | what it is |
| --- | --- | --- |
| `CHARTS_REPO_TOKEN` | gitops only, deploy | token with `contents:write` on `charts_repo` |
| `ARGOCD_AUTH_TOKEN` | deploy | ArgoCD API token that can list and sync the Applications |
| `DOCKER_REGISTRY_PASSWORD` | a registry other than `ghcr.io` | registry password |
| `TELEGRAM_TOKEN`, `TELEGRAM_CHAT`, `TELEGRAM_THREAD` | optional | failure alerts |

### Components

`components` is a YAML list, one image per entry. A single-component app names
its component `main`.

| key | default | what it does |
| --- | --- | --- |
| `name` | required | `.components.<name>` in chart values |
| `docker_context` | `.` | build context |
| `docker_file` | `Dockerfile` | Dockerfile path inside the context |
| `docker_registry` | `ghcr.io/<owner>/<repo>-<name>`; without `-<name>` for `main` or a component named after the repo | image without tag |
| `docker_build_args` | | multiline `KEY=VALUE`; Vault wins on the same key |
| `paths` | | multiline globs; on a push the component builds only if a changed file matches |
| `chart_init_containers` | | initContainers that get the same image |
| `chart_components_extra` | | other `.components.*` that get the same image |

Two components with the same image would overwrite each other, so the run stops:
give one its own `docker_registry`, or merge them with `chart_components_extra`.

### Other inputs

| input | default | what it does |
| --- | --- | --- |
| `env` | from the branch | force the environment |
| `env_prod_branches` | `main,master,release` | branches that deploy to `env_prod` |
| `env_prod` | `prod` | environment of those branches |
| `env_default` | `stg` | environment of every other branch |
| `docker_registry_username` | `github.actor` | registry login |
| `docker_tag` | `<UTC timestamp>_<branch>_<short sha>` | image tag |
| `docker_tags_extra` | | multiline extra tags, kept by cleanup |
| `docker_prefix_tag` | `false` | monorepo: prefix tags with `<app>-` |
| `docker_latest_branches` | `main,master` | branches that also push `latest` |
| `docker_cache` | `false` | buildx cache in the GitHub Actions cache |
| `docker_image_visibility` | `private` | fail if the `ghcr.io` package is not this; `public` for a public repository, empty to skip |
| `charts_values_path` | `values/<env>.yaml`, else `values.yaml` | values file inside `charts/<team>/<app>/` |
| `argocd_app` | by labels | Application name instead of the label lookup |
| `argocd_timeout` | `600` | seconds to wait for sync and health |
| `cleanup_enabled` | `true` | prune old `ghcr.io` versions |
| `cleanup_keep` | `3` | versions to keep per image |
| `runs_on` | `ubuntu-latest` | runner label |

### What a run does

1. `plan` picks the environment from the branch (tags are ignored), the tag and
   the components to build: a manual run skips unchecked `deploy_<component>`,
   a push skips components whose `paths` did not change. In deploy mode nothing
   is built if no Application has the labels `app`, `env` and `team`.
2. `build` builds and pushes each component, with build-args from
   `docker_build_args` and Vault.
3. `bump` writes the tag into the GitOps repo and pushes one commit.
4. `sync` syncs the Applications and waits until they are healthy.
5. `cleanup` keeps the last `cleanup_keep` versions of each `ghcr.io` image.
6. `report` writes the run summary; `notify` sends a Telegram alert when `plan`,
   `build`, `bump` or `sync` fails.

### GitOps repo

`bump` edits `charts/<team>/<app>/values/<env>.yaml`:

```yaml
components:
  backend:                       # name
    containers:
      main:
        image:
          repository: ghcr.io/example-org/example-app-backend
          tag: 2026.09.17_10.15_main_1a2b3c4
    initContainers:
      migrations:                # chart_init_containers
        image: { repository: same, tag: same }
  worker:                        # chart_components_extra
    containers:
      main:
        image: { repository: same, tag: same }
```

The commit is plain `git`, so the token never reaches a third-party action.
`sync` downloads the `argocd` CLI from `argocd_server`, so the runner must reach it.

### Vault

Build-args are read from `<vault_mount_ci>/<team>/<app>[/<component>]/<env>`
(no `/<component>` for `main`). Vault needs:

- a JWT auth method at `auth/<vault_auth_path>` for the issuer
  `https://token.actions.githubusercontent.com`;
- a role `vault_role` that accepts the audience `https://github.com/<owner>` and
  this repository, with read on `<vault_mount_ci>/data/*`.

A missing path means no build-args; any other Vault error fails the build.

### Notes

- Visibility: an image pushed with `GITHUB_TOKEN` takes the visibility of its
  repository and no API can change it, so a wrong one is fixed once by hand on
  the package settings page; the check only reports it.
- Cleanup prunes only images built in the run, all under one `ghcr.io` account,
  and keeps `latest` and `docker_tags_extra`.

## Actions used

| action | why |
| --- | --- |
| `actions/checkout` | the source and the GitOps repo |
| `dorny/paths-filter` | per-component `paths` filter |
| `docker/setup-buildx-action`, `docker/login-action`, `docker/build-push-action` | build and push |
| `actions/upload-artifact`, `actions/download-artifact` | component results for bump and report |
| `snok/container-retention-policy` | GHCR prune by tag, so `latest` stays protected |
| `appleboy/telegram-action` | failure alert |
