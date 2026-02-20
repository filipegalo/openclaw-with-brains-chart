# openclaw-with-brain

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 2026.2.17](https://img.shields.io/badge/AppVersion-2026.2.17-informational?style=flat-square)

Helm chart for deploying OpenClaw with a git-backed brain repository. Provides an AI agent gateway with bidirectional workspace and config synchronisation, optional Chromium sidecar, GitHub CLI, and read-only Kubernetes cluster access via ClusterRole.

## Prerequisites

- Kubernetes ≥ 1.26
- Helm ≥ 3.10
- RBAC enabled on your cluster

## Required Secrets

All secrets must exist in the same namespace as the release before deploying.

### `openclaw-env-secret`

Core OpenClaw runtime configuration.

```bash
kubectl create secret generic openclaw-env-secret \
  --namespace <namespace> \
  --from-literal=OPENCLAW_GATEWAY_TOKEN=<your-gateway-token> \
  --from-literal=ANTHROPIC_API_KEY=<your-anthropic-api-key>
```

| Key | Description |
|-----|-------------|
| `OPENCLAW_GATEWAY_TOKEN` | Token required to pair clients with the gateway |
| `ANTHROPIC_API_KEY` | Anthropic API key used by OpenClaw agents |

---

### `openclaw-telegram-token`

Telegram bot token (required if using Telegram integration).

```bash
kubectl create secret generic openclaw-telegram-token \
  --namespace <namespace> \
  --from-literal=TELEGRAM_BOT_TOKEN=<your-telegram-bot-token>
```

| Key | Description |
|-----|-------------|
| `TELEGRAM_BOT_TOKEN` | Bot token obtained from [@BotFather](https://t.me/BotFather) |

---

### `openclaw-ssh-key` *(required when `workspace.enabled: true`)*

SSH private key used by the workspace init container and sync sidecar to
clone and push to the brain repository. Must be an `ed25519` key authorised
on the Git host for the account that owns the brain repo.

```bash
# Generate a new key (skip if you already have one)
ssh-keygen -t ed25519 -C "openclaw@yourhost" -f openclaw-id_ed25519 -N ""

# Create the secret
kubectl create secret generic openclaw-ssh-key \
  --namespace <namespace> \
  --from-file=id_ed25519=./openclaw-id_ed25519
```

Add the **public key** as a deploy key with **write access** on the brain
repository (GitHub: Settings → Deploy keys → Add deploy key).

---

### `openclaw-github-token` *(required when `githubIntegration.enabled: true`)*

GitHub Personal Access Token injected as `GITHUB_TOKEN` so the `gh` CLI
can authenticate without interactive login.

Minimum required permissions for a fine-grained PAT:

| Permission | Level |
|-----------|-------|
| Contents | Read & Write |
| Pull requests | Read & Write |
| Metadata | Read (always required) |

```bash
kubectl create secret generic openclaw-github-token \
  --namespace <namespace> \
  --from-literal=GITHUB_TOKEN=<your-github-pat>
```

---

## Quick Start

```bash
helm repo add openclaw-with-brain https://filipegalo.github.io/openclaw-with-brains-chart
helm repo update

helm install openclaw openclaw-with-brain/openclaw-with-brain \
  --namespace openclaw \
  --create-namespace \
  --set workspace.repo=git@github.com:<org>/<brain-repo>.git \
  --set workspace.gitUserEmail=bot@example.com
```

## How Config Persistence Works

```
Pod start
  └── init-workspace (alpine/git)
        ├── git clone --depth=1 brain-repo → /workspace-git/brain
        ├── cp brain/<configPath>   → ~/.openclaw/openclaw.json
        └── cp brain/<path>/main/  → ~/.openclaw/workspace
            cp brain/<path>/<id>/  → ~/.openclaw/workspace-<id>

Runtime
  └── workspace-sync sidecar (every syncInterval seconds)
        ├── cp ~/.openclaw/openclaw.json → brain/<configPath>
        ├── cp ~/.openclaw/workspace*    → brain/<path>/*
        └── git commit + push origin <branch>
```

All configuration and workspace changes made by OpenClaw at runtime are
committed back to the brain repository, surviving pod restarts.

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| filipegalo | <fcostagalo@gmail.com> |  |

## Source Code

* <https://github.com/filipegalo/openclaw-with-brains-chart>

## Requirements

Kubernetes: `>=1.26.0-0`

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` |  |
| chromium | object | `{"debugPort":9222,"enabled":false,"image":{"repository":"zenika/alpine-chrome","tag":"124"},"resources":{"limits":{"cpu":"1000m","memory":"1Gi"},"requests":{"cpu":"100m","memory":"256Mi"}},"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"readOnlyRootFilesystem":true,"runAsGroup":1000,"runAsNonRoot":true,"runAsUser":1000}}` | ------------------------------------------------------------------------- |
| chromium.debugPort | int | `9222` | Remote debugging port exposed by Chromium. |
| chromium.enabled | bool | `false` | Enable the headless Chromium sidecar. |
| env | list | `[]` | Extra environment variables injected into the main container. |
| envFrom | list | `[{"secretRef":{"name":"openclaw-env-secret"}},{"secretRef":{"name":"openclaw-telegram-token"}}]` | List of envFrom sources (secretRef / configMapRef). Add secrets containing your API keys and bot tokens here. |
| fullnameOverride | string | `""` | Override the full resource name (overrides Release.Name + chart name). |
| gateway | object | `{"bind":"lan","port":18789}` | ------------------------------------------------------------------------- |
| gateway.bind | string | `"lan"` | Network interface to bind. Use "lan" for cluster-internal access or    "0.0.0.0" to also expose outside the pod. |
| gateway.port | int | `18789` | Port the OpenClaw gateway listens on. |
| githubIntegration | object | `{"enabled":true,"tokenSecret":"openclaw-github-token"}` | ------------------------------------------------------------------------- |
| githubIntegration.enabled | bool | `true` | Enable gh CLI download. |
| githubIntegration.tokenSecret | string | `"openclaw-github-token"` | Name of the Kubernetes Secret containing a GITHUB_TOKEN field. |
| image | object | `{"pullPolicy":"IfNotPresent","repository":"ghcr.io/openclaw/openclaw","tag":"2026.2.17"}` | ------------------------------------------------------------------------- |
| image.pullPolicy | string | `"IfNotPresent"` | Image pull policy. |
| image.repository | string | `"ghcr.io/openclaw/openclaw"` | OpenClaw container image repository. |
| image.tag | string | `"2026.2.17"` | OpenClaw container image tag. |
| ingress | object | `{"annotations":{},"className":"","enabled":false,"hosts":[],"tls":[]}` | ------------------------------------------------------------------------- |
| kubectlIntegration | object | `{"enabled":true}` | ------------------------------------------------------------------------- |
| kubectlIntegration.enabled | bool | `true` | Enable kubectl download and ClusterRole/Binding creation. |
| nameOverride | string | `""` | Override the chart name used in resource names and labels. |
| networkPolicy | object | `{"egress":[{"ports":[{"port":53,"protocol":"UDP"},{"port":53,"protocol":"TCP"}],"to":[{"namespaceSelector":{"matchLabels":{"kubernetes.io/metadata.name":"kube-system"}},"podSelector":{"matchLabels":{"k8s-app":"kube-dns"}}}]},{"to":[{"ipBlock":{"cidr":"0.0.0.0/0","except":["10.0.0.0/8","172.16.0.0/12","192.168.0.0/16","169.254.0.0/16","100.64.0.0/10"]}}]}],"enabled":false,"ingress":[{"from":[{"namespaceSelector":{"matchLabels":{"kubernetes.io/metadata.name":"ingress-nginx"}}}],"ports":[{"port":18789,"protocol":"TCP"}]}]}` | ------------------------------------------------------------------------- |
| networkPolicy.egress | list | `[{"ports":[{"port":53,"protocol":"UDP"},{"port":53,"protocol":"TCP"}],"to":[{"namespaceSelector":{"matchLabels":{"kubernetes.io/metadata.name":"kube-system"}},"podSelector":{"matchLabels":{"k8s-app":"kube-dns"}}}]},{"to":[{"ipBlock":{"cidr":"0.0.0.0/0","except":["10.0.0.0/8","172.16.0.0/12","192.168.0.0/16","169.254.0.0/16","100.64.0.0/10"]}}]}]` | Egress rules. DNS + external internet (blocks RFC-1918 private ranges). |
| networkPolicy.enabled | bool | `false` | Enable the NetworkPolicy resource. |
| networkPolicy.ingress | list | `[{"from":[{"namespaceSelector":{"matchLabels":{"kubernetes.io/metadata.name":"ingress-nginx"}}}],"ports":[{"port":18789,"protocol":"TCP"}]}]` | Ingress rules. By default allows traffic from ingress-nginx namespace. |
| nodeSelector | object | `{}` | ------------------------------------------------------------------------- |
| podSecurityContext | object | `{"fsGroup":1000,"fsGroupChangePolicy":"OnRootMismatch"}` | ------------------------------------------------------------------------- |
| probes | object | `{"chromium":{"liveness":{"failureThreshold":6,"initialDelaySeconds":10,"periodSeconds":30,"timeoutSeconds":5},"readiness":{"failureThreshold":3,"initialDelaySeconds":5,"periodSeconds":10,"timeoutSeconds":5},"startup":{"failureThreshold":12,"initialDelaySeconds":5,"periodSeconds":5,"timeoutSeconds":5}},"main":{"liveness":{"failureThreshold":3,"initialDelaySeconds":30,"periodSeconds":30,"timeoutSeconds":5},"readiness":{"failureThreshold":3,"initialDelaySeconds":10,"periodSeconds":10,"timeoutSeconds":5},"startup":{"failureThreshold":30,"initialDelaySeconds":5,"periodSeconds":5,"timeoutSeconds":5}}}` | ------------------------------------------------------------------------- |
| replicaCount | int | `1` | Number of replicas. Keep at 1 — the emptyDir data volume is not shared. |
| resources | object | `{"limits":{"cpu":"2000m","memory":"2Gi"},"requests":{"cpu":"200m","memory":"512Mi"}}` | ------------------------------------------------------------------------- |
| securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"readOnlyRootFilesystem":true,"runAsGroup":1000,"runAsNonRoot":true,"runAsUser":1000}` | ------------------------------------------------------------------------- |
| service | object | `{"port":18789,"type":"ClusterIP"}` | ------------------------------------------------------------------------- |
| strategy | string | `"RollingUpdate"` | Deployment strategy. RollingUpdate is safe because volumes are emptyDir. |
| telegramAllowFrom | list | `[]` | List of Telegram sender IDs allowed to interact with the bot. Example: telegramAllowFrom: ["123456789"] |
| tolerations | list | `[]` |  |
| tools | object | `{"image":{"repository":"alpine","tag":"3.21"}}` | ------------------------------------------------------------------------- |
| tools.image.repository | string | `"alpine"` | Alpine image used by the init-tools container to download binaries. |
| tools.image.tag | string | `"3.21"` | Alpine image tag. |
| workspace | object | `{"branch":"main","configPath":"openclaw.json","enabled":true,"gitUserEmail":"openclaw@example.com","gitUserName":"openclaw","image":{"repository":"alpine/git","tag":"2.47.2"},"path":"workspaces","repo":"","resources":{"limits":{"cpu":"200m","memory":"256Mi"},"requests":{"cpu":"50m","memory":"64Mi"}},"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"readOnlyRootFilesystem":false},"sshKeySecret":"openclaw-ssh-key","syncInterval":60}` | ------------------------------------------------------------------------- |
| workspace.branch | string | `"main"` | Branch to clone and push to. |
| workspace.configPath | string | `"openclaw.json"` | Path inside the repo to openclaw.json (synced both ways). |
| workspace.enabled | bool | `true` | Enable git-backed workspace sync. |
| workspace.gitUserEmail | string | `"openclaw@example.com"` | Git committer email used by the sync sidecar. |
| workspace.gitUserName | string | `"openclaw"` | Git committer name used by the sync sidecar. |
| workspace.image.repository | string | `"alpine/git"` | alpine/git image used for init-workspace and workspace-sync containers. |
| workspace.image.tag | string | `"2.47.2"` | alpine/git image tag. |
| workspace.path | string | `"workspaces"` | Path inside the repo whose subdirectories map to OpenClaw workspaces.    Subdirectory "main" maps to ~/.openclaw/workspace,    any other subdirectory <id> maps to ~/.openclaw/workspace-<id>. |
| workspace.repo | string | `""` | SSH URL of the brain repository. Required when workspace.enabled is true. |
| workspace.sshKeySecret | string | `"openclaw-ssh-key"` | Name of the Kubernetes Secret containing the SSH private key.    The secret must have a field named "id_ed25519". |
| workspace.syncInterval | int | `60` | How often (seconds) the sync sidecar commits and pushes changes. |

---

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.14.2](https://github.com/norwoodj/helm-docs/releases/v1.14.2)
