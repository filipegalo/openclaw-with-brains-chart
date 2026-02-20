# 🦞 OpenClaw Helm Chart with 🧠

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/openclaw-with-brain)](https://artifacthub.io/packages/helm/openclaw-with-brain/openclaw-with-brain)
[![Helm 3](https://img.shields.io/badge/Helm-3-0F1689?logo=helm&logoColor=white)](https://helm.sh)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-%3E%3D1.26-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
![AppVersion: 2026.2.17](https://img.shields.io/badge/AppVersion-2026.2.17-informational?style=flat-square)
![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Deploys [OpenClaw](https://github.com/openclaw/openclaw) — an autonomous AI assistant — on Kubernetes, wired to a **git-backed brain repository** for persistent workspaces and configuration across pod restarts.

## Features

- **Git-backed brain** — workspaces and `openclaw.json` are cloned from your repo on startup and synced back every minute
- **`kubectl` on PATH** — read-only ClusterRole + ClusterRoleBinding created automatically so OpenClaw can inspect your cluster
- **`gh` CLI on PATH** — GitHub CLI injected via PAT token for repository operations
- **Headless Chromium** — optional sidecar for browser automation tasks
- **Hardened security** — non-root (UID 1000), read-only root filesystem, all capabilities dropped

## Architecture

Single-instance deployment. Horizontal scaling is not supported — the data volume is `emptyDir` and state lives in Git.

| Component | Kind | Purpose |
|-----------|------|---------|
| `init-workspace` | Init container | Clones brain repo, restores config + workspaces |
| `init-tools` | Init container | Downloads `kubectl` and/or `gh` binary |
| `main` | Container | OpenClaw gateway (HTTP/WebSocket) |
| `workspace-sync` | Sidecar | Commits and pushes changes back to brain repo |
| `chromium` | Sidecar (optional) | Headless browser on port `9222` |

| Port | Protocol | Purpose |
|------|----------|---------|
| `18789` | HTTP/WebSocket | OpenClaw gateway |
| `9222` | HTTP | Chromium remote debugging (optional) |

## Prerequisites

- Kubernetes ≥ 1.26
- Helm ≥ 3.10
- RBAC enabled on your cluster

## Installation

### 1. Add the Helm repository

```bash
helm repo add openclaw-with-brain https://filipegalo.github.io/openclaw-with-brains-chart
helm repo update
```

### 2. Create required secrets

```bash
NAMESPACE=openclaw
kubectl create namespace $NAMESPACE

# Core API keys
kubectl create secret generic openclaw-env-secret \
  --namespace $NAMESPACE \
  --from-literal=OPENCLAW_GATEWAY_TOKEN=<your-gateway-token> \
  --from-literal=ANTHROPIC_API_KEY=<your-anthropic-api-key>

# Telegram bot (if using Telegram)
kubectl create secret generic openclaw-telegram-token \
  --namespace $NAMESPACE \
  --from-literal=TELEGRAM_BOT_TOKEN=<your-bot-token>

# SSH key for brain repo access (ed25519, must have write access)
kubectl create secret generic openclaw-ssh-key \
  --namespace $NAMESPACE \
  --from-file=id_ed25519=./path/to/your/ed25519-key

# GitHub PAT for gh CLI (if githubIntegration.enabled)
kubectl create secret generic openclaw-github-token \
  --namespace $NAMESPACE \
  --from-literal=GITHUB_TOKEN=<your-github-pat>
```

> Add the **public key** as a deploy key with **write access** on your brain repository:
> GitHub → Repository → Settings → Deploy keys → Add deploy key

### 3. Install

```bash
helm install openclaw openclaw-with-brain/openclaw-with-brain \
  --namespace openclaw \
  --create-namespace \
  --set workspace.repo=git@github.com:<org>/<brain-repo>.git \
  --set workspace.gitUserEmail=bot@example.com
```

### 4. Pair your device

```bash
# Port-forward the gateway
kubectl port-forward -n openclaw svc/openclaw-openclaw-with-brain 18789:18789

# Open http://localhost:18789 and enter your Gateway Token, then approve:
kubectl exec -n openclaw deployment/openclaw-openclaw-with-brain -- \
  node dist/index.js devices list

kubectl exec -n openclaw deployment/openclaw-openclaw-with-brain -- \
  node dist/index.js devices approve <REQUEST_ID>
```

## How the Brain Sync Works

```
Pod start
  └── init-workspace (alpine/git)
        ├── git clone --depth=1 <workspace.repo> → /workspace-git/brain
        ├── brain/<configPath>  → ~/.openclaw/openclaw.json
        └── brain/<path>/main/ → ~/.openclaw/workspace
            brain/<path>/<id>/ → ~/.openclaw/workspace-<id>

Runtime (every syncInterval seconds)
  └── workspace-sync sidecar
        ├── ~/.openclaw/openclaw.json → brain/<configPath>
        ├── ~/.openclaw/workspace*    → brain/<path>/*
        └── git commit -m "chore: sync from openclaw [timestamp]"
            git push origin <branch>
```

All runtime changes — including config edits made by OpenClaw — are committed back to Git, surviving pod restarts and ArgoCD syncs.

## Uninstall

```bash
helm uninstall openclaw --namespace openclaw
```

## Security

- Runs as **UID/GID 1000**, non-root
- **Read-only root filesystem** on all containers (except `workspace-sync` which writes to the git clone)
- All Linux capabilities dropped
- ClusterRole grants **read-only** access to cluster resources — **Secrets are excluded**

<details>
<summary>Network Policy (optional)</summary>

Enable `networkPolicy.enabled: true` to apply a default-deny policy with explicit egress rules:

- DNS (UDP/TCP 53) to `kube-system`
- Public internet (blocks RFC-1918: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`)

Add custom ingress/egress rules via `networkPolicy.ingress` and `networkPolicy.egress` in your values.

</details>

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
