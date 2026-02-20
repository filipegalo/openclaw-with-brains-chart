# openclaw-with-brain

Helm chart for deploying [OpenClaw](https://github.com/openclaw/openclaw) with a git-backed
brain repository. Provides an AI agent gateway with:

- Bidirectional workspace and config synchronisation via Git
- Optional headless Chromium sidecar for browser automation
- GitHub CLI (`gh`) available on PATH
- `kubectl` available on PATH with read-only cluster access

## Prerequisites

- Kubernetes ≥ 1.26
- Helm ≥ 3.10
- RBAC enabled on your cluster

## Required Secrets

All secrets must exist in the same namespace as the release before deploying.

### `tars-env-secret`

Core OpenClaw runtime configuration.

```bash
kubectl create secret generic tars-env-secret \
  --namespace <namespace> \
  --from-literal=OPENCLAW_GATEWAY_TOKEN=<your-gateway-token> \
  --from-literal=ANTHROPIC_API_KEY=<your-anthropic-api-key>
```

| Key | Description |
|-----|-------------|
| `OPENCLAW_GATEWAY_TOKEN` | Token required to pair clients with the gateway |
| `ANTHROPIC_API_KEY` | Anthropic API key used by OpenClaw agents |

---

### `tars-telegram-token`

Telegram bot token (required if using Telegram integration).

```bash
kubectl create secret generic tars-telegram-token \
  --namespace <namespace> \
  --from-literal=TELEGRAM_BOT_TOKEN=<your-telegram-bot-token>
```

| Key | Description |
|-----|-------------|
| `TELEGRAM_BOT_TOKEN` | Bot token obtained from [@BotFather](https://t.me/BotFather) |

---

### `tars-ssh-key`

SSH private key used by the workspace init container and sync sidecar to
clone and push to the brain repository.

The key must be an `ed25519` key authorised on GitHub (or your Git host)
for the account that owns the brain repository.

```bash
# Generate a new key (skip if you already have one)
ssh-keygen -t ed25519 -C "tars@yourhost" -f tars-id_ed25519 -N ""

# Create the secret
kubectl create secret generic tars-ssh-key \
  --namespace <namespace> \
  --from-file=id_ed25519=./tars-id_ed25519
```

Then add the **public key** (`tars-id_ed25519.pub`) as a deploy key with
**write access** on the brain repository:

> GitHub → Repository → Settings → Deploy keys → Add deploy key

---

### `tars-github-token` *(optional, needed when `githubIntegration.enabled: true`)*

GitHub Personal Access Token (PAT) injected as `GITHUB_TOKEN` so the `gh`
CLI can interact with GitHub without interactive login.

Minimum required permissions for a fine-grained PAT:
- **Contents**: Read & Write (to push commits / create PRs)
- **Pull requests**: Read & Write
- **Issues**: Read & Write (optional)
- **Metadata**: Read (always required)

```bash
kubectl create secret generic tars-github-token \
  --namespace <namespace> \
  --from-literal=GITHUB_TOKEN=<your-github-pat>
```

---

## Quick Start

```bash
helm repo add openclaw-with-brain https://filipegalo.github.io/openclaw-with-brains-chart
helm repo update

helm install tars openclaw-with-brain/openclaw-with-brain \
  --namespace tars \
  --create-namespace \
  --set workspace.repo=git@github.com:<org>/<brain-repo>.git
```

## Configuration

See [values.yaml](./values.yaml) for all available options with inline
comments. Key sections:

| Section | Description |
|---------|-------------|
| `image` | OpenClaw container image |
| `workspace` | Git-backed brain repo sync settings |
| `kubectlIntegration` | Enable/disable kubectl on PATH |
| `githubIntegration` | Enable/disable gh CLI + token secret |
| `chromium` | Optional headless Chromium sidecar |
| `gateway` | Bind address and port |
| `telegramAllowFrom` | Telegram sender ID allowlist |
| `networkPolicy` | Optional egress/ingress network policy |
| `ingress` | Optional Ingress resource |

## How Config Persistence Works

```
Pod start
  └── init-workspace (alpine/git)
        ├── git clone --depth=1 brain-repo → /workspace-git/brain
        ├── cp brain/TARS/openclaw.json → ~/.openclaw/openclaw.json
        └── cp brain/TARS/workspaces/*  → ~/.openclaw/workspace[-<id>]

Runtime
  └── workspace-sync sidecar (every syncInterval seconds)
        ├── cp ~/.openclaw/openclaw.json → brain/TARS/openclaw.json
        ├── cp ~/.openclaw/workspace*    → brain/TARS/workspaces/*
        └── git commit + push origin main
```

All configuration and workspace changes made by OpenClaw at runtime are
committed back to the brain repository, surviving pod restarts.
