# Deploy to on-premise

Build a Docker image, push it to GHCR, and deploy to an on-premise server over a Tailscale network via SSH.

Secrets (Tailscale credentials, SSH key) are fetched at runtime from [Infisical](https://infisical.com) via OIDC — no GitHub Secrets to configure per repository.

## Table of Contents

- [Example](#example)
- [Inputs](#inputs)
- [Secrets Management](#secrets-management)
- [How it works](#how-it-works)
- [Prerequisites](#prerequisites)

## Example

`.github/workflows/deploy.yml`

```yaml
name: Deploy
run-name: Deploy by @${{ github.actor }}

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  deploy:
    uses: <owner>/actions/.github/workflows/deploy-on-premise.yml@main
    with:
      image-name: my-app
      host: '100.x.x.x'
      deploy-path: /path/to/my-app
      infisical-identity-id: '<your-machine-identity-id>'
      infisical-project-slug: '<your-project-slug>'
```

### Permissions

The caller workflow must declare these permissions at the top level:

- `contents: read` — checkout the repository
- `packages: write` — push images to GHCR
- `id-token: write` — Infisical OIDC authentication

## Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `image-name` | `string` | Yes | — | Docker image name (e.g. `my-app`) |
| `host` | `string` | Yes | — | Tailscale IP or hostname of the target machine |
| `deploy-path` | `string` | Yes | — | Working directory on the target machine for deployment |
| `infisical-identity-id` | `string` | Yes | — | Infisical Machine Identity ID (OIDC, public value) |
| `infisical-project-slug` | `string` | Yes | — | Infisical project slug |
| `infisical-env-slug` | `string` | No | `prod` | Infisical environment slug |
| `deploy-command` | `string` | No | `docker compose up -d` | Command to run after `docker pull` |
| `user` | `string` | No | `root` | SSH user on the target machine |
| `ssh-port` | `number` | No | `22` | SSH port on the target machine |
| `dockerfile` | `string` | No | `./Dockerfile` | Dockerfile path relative to repo root |
| `context` | `string` | No | `.` | Docker build context relative to repo root |
| `tailscale-tags` | `string` | No | `tag:ci` | Tailscale ACL tags for ephemeral node |

## Secrets Management

**No GitHub Secrets required per repository.** All secrets are stored in Infisical and fetched at runtime via OIDC.

### Required secrets in Infisical

| Key | Description |
|-----|-------------|
| `TAILSCALE_OAUTH_CLIENT_ID` | Tailscale OAuth client ID |
| `TAILSCALE_OAUTH_SECRET` | Tailscale OAuth client secret |
| `SSH_PRIVATE_KEY` | SSH private key for target machine access |

These are stored once in your Infisical project and shared across all repositories.

## How it works

```
push to main
  → GitHub Actions runner
    1. Fetch secrets from Infisical (OIDC, zero stored credentials)
    2. Build Docker image (linux/amd64, BuildKit layer cache)
    3. Push to ghcr.io/<owner>/<image-name>:latest + :sha
    4. Connect runner to Tailscale as ephemeral node
    5. SSH into target machine → docker pull → deploy
```

### Image tags

Each build produces two tags:

| Tag | Purpose |
|-----|---------|
| `latest` | Used to pull the current version |
| `<commit-sha>` | Immutable reference for rollback |

## Prerequisites

### Infisical setup

1. Create a project in [Infisical](https://app.infisical.com)
2. Add the three secrets (`TAILSCALE_OAUTH_CLIENT_ID`, `TAILSCALE_OAUTH_SECRET`, `SSH_PRIVATE_KEY`) to the `prod` environment
3. Create a **Machine Identity** with OIDC auth method
4. Configure the identity:
   - **OIDC Discovery URL**: `https://token.actions.githubusercontent.com`
   - **Issuer**: `https://token.actions.githubusercontent.com`
   - **Subject**: `repo:<owner>/*:ref:refs/heads/main` (or scope per repo)
   - **Audience**: `https://github.com/<owner>`
5. Add the identity to the project with **Read** permission
6. Copy the Machine Identity ID — this is a public value, safe to commit

### One-time target machine setup

```bash
# 1. Log in to GHCR (only once, credentials are persisted)
docker login ghcr.io -u <github-username> -p <GITHUB_PAT>

# 2. Create deploy directory with docker-compose.yml
mkdir -p /path/to/my-app
```

The PAT only needs the `read:packages` scope.

### Tailscale OAuth client

1. Go to [Tailscale Admin Console](https://login.tailscale.com/admin/settings/oauth) → Settings → OAuth clients
2. Create a new OAuth client
3. Grant the `auth_keys` scope with **Write** permission
4. Assign a tag (e.g. `tag:ci`) and ensure it's allowed in your ACL policy
5. Store the client ID and secret in **Infisical** (not in GitHub Secrets)

### SSH key

1. Generate a key pair: `ssh-keygen -t ed25519 -C "github-actions"`
2. Add the public key to the target machine user's `~/.ssh/authorized_keys`
3. Store the private key in **Infisical** as `SSH_PRIVATE_KEY` (not in GitHub Secrets)
