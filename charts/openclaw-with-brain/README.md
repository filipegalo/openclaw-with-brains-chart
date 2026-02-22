# 🦞 OpenClaw Helm Chart with 🧠

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/openclaw-with-brain)](https://artifacthub.io/packages/helm/openclaw-with-brain/openclaw-with-brain)
[![Helm 3](https://img.shields.io/badge/Helm-3-0F1689?logo=helm&logoColor=white)](https://helm.sh)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-%3E%3D1.26-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
![AppVersion: 2026.2.21](https://img.shields.io/badge/AppVersion-2026.2.21-informational?style=flat-square)
![Version: 0.1.15](https://img.shields.io/badge/Version-0.1.15-informational?style=flat-square)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Deploys [OpenClaw](https://github.com/openclaw/openclaw) — an autonomous AI assistant — on Kubernetes, wired to a **git-backed brain repository** for persistent workspaces and configuration across pod restarts.

## Why "with Brain"?

OpenClaw stores its personality, memory, tools configuration, and session data in a **workspace directory** (`~/.openclaw/workspace`). By default this lives on ephemeral container storage — meaning **every pod restart wipes the agent's memory clean**.

The "brain" is a Git repository that solves this problem:

- **Persistence** — workspaces and `openclaw.json` config survive pod restarts, node evictions, and cluster migrations
- **Version history** — every change the agent makes is a Git commit, giving you a full audit trail
- **Portability** — clone the brain repo and you have your agent's entire state, runnable anywhere
- **GitOps-friendly** — the brain repo is the source of truth; ArgoCD/Flux can redeploy and the agent picks up where it left off

Without the brain repo, OpenClaw works fine — but it forgets everything every time the pod restarts. With it, your agent has long-term memory.

## Features

- **Git-backed brain** — workspaces and `openclaw.json` are cloned from your repo on startup and synced back every minute
- **Plug-in tools** — install any CLI tool (e.g. `kubectl`, `gh`) via `tools.packages`; binaries land on `PATH` inside the main container
- **Headless Chromium** — optional sidecar for browser automation tasks
- **Hardened security** — non-root (UID 1000), read-only root filesystem, all capabilities dropped

## Cluster Access — OpenClaw as SRE

When `kubectl` is included in `tools.packages`, the chart automatically creates a **ClusterRole** and **ClusterRoleBinding** that grant the pod's ServiceAccount **read-only** access to the cluster it is deployed on.

This lets you use OpenClaw as a lightweight SRE assistant: it can inspect deployments, check pod logs, describe failing resources, query events, and reason about the cluster state — all without any manual RBAC setup on your part.

```yaml
tools:
  packages:
    - kubectl   # ← also creates ClusterRole + ClusterRoleBinding
    - gh
```

The ClusterRole is intentionally scoped to read-only verbs (`get`, `list`, `watch`). **Secrets are excluded** — the agent can never read secret values from the cluster. It covers: Pods, Deployments, Services, Ingresses, Events, Nodes, Jobs, HPAs, CertManager, ExternalSecrets, ArgoCD, and more.

This is entirely opt-in — if `kubectl` is not in `tools.packages`, no RBAC resources are created.

## Architecture

Single-instance deployment. Horizontal scaling is not supported — the data volume is `emptyDir` and state lives in Git.

![Architecture](https://raw.githubusercontent.com/filipegalo/openclaw-with-brains-chart/main/charts/openclaw-with-brain/assets/arch.png)

| Component | Kind | Purpose |
|-----------|------|---------|
| `init-workspace` | Init container | Clones brain repo, restores config + workspaces |
| `init-tools` | Init container | Downloads `kubectl` and/or `gh` binary |
| `main` | Container | OpenClaw gateway (HTTP/WebSocket) |
| `workspace-sync` | Sidecar | Commits and pushes changes back to brain repo |
| `chromium` | Sidecar (optional) | Headless browser on port `9222` |

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

> **Deploy key setup:** Add the **public key** as a deploy key with **write access** on your brain repository:
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

## Brain Repository Structure

The brain repo is just a regular Git repository with a specific directory layout:

```
your-brain-repo/
├── openclaw.json              # OpenClaw configuration (synced bidirectionally)
└── workspaces/
    ├── main/                  # Main agent workspace → ~/.openclaw/workspace
    │   ├── SOUL.md            # Agent personality
    │   ├── MEMORY.md          # Long-term memory
    │   ├── AGENTS.md          # Agent instructions
    │   ├── USER.md            # User profile
    │   └── memory/            # Daily memory logs
    │       └── 2026-02-20.md
    └── <agent-id>/            # Additional agent workspaces → ~/.openclaw/workspace-<agent-id>
        └── ...
```

### Setting Up a New Brain Repo

```bash
# Create the repo
mkdir my-openclaw-brain && cd my-openclaw-brain
git init

# Create minimal structure
mkdir -p workspaces/main
echo '{}' > openclaw.json
echo '# Soul' > workspaces/main/SOUL.md

# Push it
git remote add origin git@github.com:<you>/<brain-repo>.git
git add . && git commit -m "Initial brain"
git push -u origin main
```

The agent will populate the workspace with its own files (MEMORY.md, daily logs, etc.) on first run — the sync sidecar will commit them back automatically.

### Migrating from an Existing OpenClaw Setup

If you already have OpenClaw running (e.g. locally or in another deployment), you can carry over its full state — personality, memory, config, and all agent workspaces — into the brain repo.

OpenClaw stores everything under `~/.openclaw/` inside the container:

| Container path | Brain repo path | Description |
|---|---|---|
| `~/.openclaw/openclaw.json` | `openclaw.json` (or `workspace.configPath`) | Main config |
| `~/.openclaw/workspace/` | `workspaces/main/` (or `workspace.path/main/`) | Main agent workspace |
| `~/.openclaw/workspace-<name>/` | `workspaces/<name>/` (or `workspace.path/<name>/`) | Additional agent workspaces |

```bash
# 1. Copy the files out of the running container
kubectl cp <namespace>/<pod>:/home/node/.openclaw/openclaw.json ./openclaw.json
kubectl cp <namespace>/<pod>:/home/node/.openclaw/workspace ./workspaces/main
# Repeat for each additional agent workspace:
kubectl cp <namespace>/<pod>:/home/node/.openclaw/workspace-myagent ./workspaces/myagent

# 2. Or, if running OpenClaw locally (not on Kubernetes):
cp ~/.openclaw/openclaw.json ./openclaw.json
cp -r ~/.openclaw/workspace   ./workspaces/main
cp -r ~/.openclaw/workspace-myagent ./workspaces/myagent

# 3. Commit and push to your brain repo
git add .
git commit -m "chore: import existing openclaw state"
git push -u origin main
```

On the next pod start, `init-workspace` will clone the repo and restore everything exactly as it was.

## How the Brain Sync Works

### Startup (init-workspace)

1. Clones the brain repo (shallow, single branch) via SSH
2. Copies `openclaw.json` → `~/.openclaw/openclaw.json`
3. Maps each subdirectory under `workspaces/`:
   - `main/` → `~/.openclaw/workspace`
   - `<id>/` → `~/.openclaw/workspace-<id>`

### Runtime (workspace-sync sidecar)

Every `syncInterval` seconds (default: 60):

1. Copies workspace files back into the git clone
2. Copies `openclaw.json` back (captures runtime config changes)
3. Fetches and fast-forwards from remote (picks up external changes)
4. Commits and pushes if anything changed

On pod shutdown (SIGTERM), runs one final sync before exiting (90s grace period).

### Conflict handling

The sync uses `--ff-only` merge. If the remote branch has diverged (e.g., someone force-pushed), the sidecar logs a warning and keeps the local version — it will retry next cycle. The agent's in-memory state always takes priority over the remote.

### What gets synced

| Direction | What | When |
|-----------|------|------|
| Git → Pod | Workspaces, `openclaw.json` | Pod startup only |
| Pod → Git | Workspaces, `openclaw.json` | Every `syncInterval` + shutdown |

> **Note:** Only files under `workspaces/` and the config file are synced. Secrets, API keys, and environment variables are **never** committed to the brain repo — they live in Kubernetes Secrets.

## Edge Cases & Troubleshooting

### Pod restarts and data loss

**Scenario:** Pod crashes between sync intervals.
**Impact:** Up to `syncInterval` seconds of workspace changes are lost. The agent re-clones from the last successful push.
**Mitigation:** Lower `syncInterval` (minimum 10s) for critical workloads. Note that very low intervals increase Git server load.

### Empty or missing brain repo

**Scenario:** `workspace.repo` points to an empty repo or wrong branch.
**Impact:** Init container fails → pod stays in `Init:CrashLoopBackOff`.
**Fix:** Ensure the repo exists, the branch exists, and the SSH deploy key has access. Check init container logs:
```bash
kubectl logs -n openclaw <pod> -c init-workspace
```

### SSH key not found or wrong permissions

**Scenario:** Secret `openclaw-ssh-key` is missing or doesn't have an `id_ed25519` field.
**Impact:** Init container fails with SSH authentication error.
**Fix:**
```bash
# Verify the secret exists and has the right key
kubectl get secret openclaw-ssh-key -n openclaw -o jsonpath='{.data.id_ed25519}' | base64 -d | head -1
# Should show: -----BEGIN OPENSSH PRIVATE KEY-----
```

### Non-GitHub Git hosts

The chart hardcodes GitHub's SSH host key for `StrictHostKeyChecking`. If you use GitLab, Bitbucket, or a self-hosted Git server:

1. Get your host's SSH public key: `ssh-keyscan gitlab.com`
2. Fork and modify the `init-workspace.sh` and `workspace-sync.sh` scripts in the ConfigMap, or override via `configMapScripts` (not yet supported — contributions welcome)

### NetworkPolicy blocks workspace sync

**Scenario:** `networkPolicy.enabled: true` but workspace sync fails silently.
**Impact:** Changes are never pushed; brain repo becomes stale.
**Root cause:** The default egress rules block RFC-1918 ranges but allow public internet. If your Git server is on a private IP (self-hosted GitLab, etc.), egress won't reach it.
**Fix:** Add an egress rule for your Git server:
```yaml
networkPolicy:
  egress:
    # ... existing rules ...
    - to:
        - ipBlock:
            cidr: 10.0.0.50/32  # your Git server
      ports:
        - protocol: TCP
          port: 22
```

Also note: if `workspace.enabled: true`, the sync sidecar needs SSH (port 22) egress. The default rules allow this for public IPs but not private ones.

### kubectl RBAC creation fails

**Scenario:** `kubectl` is in `tools.packages` but the cluster uses a restrictive admission controller that blocks ClusterRoleBinding creation.
**Impact:** Helm install fails or `kubectl` commands inside the pod return `Forbidden`.
**Fix:** Either grant the Helm service account permission to create ClusterRoleBindings, or remove `kubectl` from `tools.packages` and create the RBAC resources manually.

### GitHub CLI auth failure

**Scenario:** `gh` commands fail with "authentication required".
**Impact:** Agent can't interact with GitHub repos.
**Checklist:**
- Secret `openclaw-github-token` exists with a `GITHUB_TOKEN` field
- Token has the required scopes (`repo`, `read:org` at minimum; add `workflow` for CI operations)
- Token hasn't expired (classic PATs can expire)

### Chromium sidecar OOM

**Scenario:** Browser automation causes Chromium to exceed memory limits.
**Impact:** Chromium container gets OOMKilled, browser tasks fail.
**Fix:** Increase `chromium.resources.limits.memory` (default 1Gi). Heavy pages or multiple tabs can easily exceed this:
```yaml
chromium:
  resources:
    limits:
      memory: 2Gi
```

### Multiple replicas

**Don't.** `replicaCount` must stay at 1. The data volume is `emptyDir` (not shared), and multiple pods pushing to the same Git branch will conflict. If you need HA, consider an active-passive setup with a PVC instead of emptyDir (not supported by this chart).

### Telegram allowFrom misconfigured

**Scenario:** `telegramAllowFrom` is set but the bot doesn't respond to your messages.
**Fix:** The values must be **string** sender IDs, not integers:
```yaml
# ✅ Correct
telegramAllowFrom: ["455669209"]

# ❌ Wrong — YAML will parse as integer
telegramAllowFrom: [455669209]
```

Check your Telegram user ID by messaging [@userinfobot](https://t.me/userinfobot).

### Sync sidecar can't write to workspace

**Scenario:** Sync logs show permission denied errors.
**Root cause:** The workspace files were created by UID 1000 (main container) but the sync sidecar also runs as UID 1000. The issue is usually `fsGroup` not being applied to dynamically created files.
**Fix:** Ensure `podSecurityContext.fsGroup: 1000` and `fsGroupChangePolicy: OnRootMismatch` are set (they are by default).

## Uninstall

```bash
helm uninstall openclaw --namespace openclaw
```

> **Note:** Uninstalling does **not** delete your brain repo or Kubernetes Secrets. Your agent's state is safe in Git.

## Security

- Runs as **UID/GID 1000**, non-root
- **Read-only root filesystem** on all containers (except `workspace-sync` which writes to the git clone)
- All Linux capabilities dropped
- ClusterRole grants **read-only** access to cluster resources — **Secrets are excluded**
- Chromium runs with `--no-sandbox` — this is standard for containerized Chromium (the container itself is the sandbox). The `--no-sandbox` flag disables Chromium's internal sandbox which requires unprivileged user namespaces, typically unavailable in containers. Security is enforced at the container level instead (non-root, dropped capabilities, read-only rootfs).

<details>
<summary>Network Policy (optional)</summary>

Enable `networkPolicy.enabled: true` to apply a default-deny policy with explicit egress rules:

- DNS (UDP/TCP 53) to `kube-system`
- Public internet (blocks RFC-1918: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`)

Add custom ingress/egress rules via `networkPolicy.ingress` and `networkPolicy.egress` in your values.

**⚠️ If using workspace sync with a private Git server**, add an egress rule for SSH (port 22) to your server's IP. See [Edge Cases](#networkpolicy-blocks-workspace-sync) above.

</details>

## Configuration Examples

### Minimal (no brain, no tools)

```yaml
workspace:
  enabled: false
tools:
  packages: []
chromium:
  enabled: false
```

### Full-featured

```yaml
workspace:
  enabled: true
  repo: git@github.com:myorg/openclaw-brain.git
  branch: main
  syncInterval: 30
  gitUserEmail: openclaw@myorg.com

tools:
  packages:
    - kubectl   # read-only ClusterRole + ClusterRoleBinding created automatically
    - gh        # GitHub CLI; mounts githubIntegration.tokenSecret

githubIntegration:
  tokenSecret: openclaw-github-token

chromium:
  enabled: true
  resources:
    limits:
      memory: 2Gi

telegramAllowFrom: ["455669209"]

networkPolicy:
  enabled: true
```

### Air-gapped / private registry

```yaml
imagePullSecrets:
  - name: my-registry-creds

image:
  repository: registry.internal.com/openclaw/openclaw
  tag: "2026.2.17"

workspace:
  image:
    repository: registry.internal.com/alpine/git
    tag: "2.47.2"

tools:
  image:
    repository: registry.internal.com/alpine
    tag: "3.21"

chromium:
  image:
    repository: registry.internal.com/zenika/alpine-chrome
    tag: "124"
```

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
| githubIntegration | object | `{"tokenSecret":"openclaw-github-token"}` | ------------------------------------------------------------------------- |
| githubIntegration.tokenSecret | string | `"openclaw-github-token"` | Name of the Kubernetes Secret containing a GITHUB_TOKEN field. |
| image | object | `{"pullPolicy":"IfNotPresent","repository":"ghcr.io/openclaw/openclaw","tag":"2026.2.21"}` | ------------------------------------------------------------------------- |
| image.pullPolicy | string | `"IfNotPresent"` | Image pull policy. |
| image.repository | string | `"ghcr.io/openclaw/openclaw"` | OpenClaw container image repository. |
| image.tag | string | `"2026.2.21"` | OpenClaw container image tag. |
| imagePullSecrets | list | `[]` | List of image pull secrets for private registries. |
| ingress | object | `{"annotations":{},"className":"","enabled":false,"hosts":[],"tls":[]}` | ------------------------------------------------------------------------- |
| nameOverride | string | `""` | Override the chart name used in resource names and labels. Defaults to chart name. |
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
| tools | object | `{"image":{"repository":"alpine","tag":"3.21"},"packages":["kubectl","gh"]}` | ------------------------------------------------------------------------- |
| tools.image.repository | string | `"alpine"` | Alpine image used by the init-tools container (mise is bootstrapped at runtime). |
| tools.image.tag | string | `"3.21"` | Alpine image tag. |
| tools.packages | list | `["kubectl","gh"]` | List of tools to install via mise. Any tool supported by mise works here. See https://mise.jdx.dev/registry.html for the full list. |
| workspace | object | `{"branch":"main","configPath":"openclaw.json","enabled":true,"gitUserEmail":"openclaw@example.com","gitUserName":"openclaw","image":{"repository":"alpine/git","tag":"2.47.2"},"path":"workspaces","repo":"","resources":{"limits":{"cpu":"200m","memory":"256Mi"},"requests":{"cpu":"50m","memory":"64Mi"}},"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"add":["DAC_READ_SEARCH"],"drop":["ALL"]},"readOnlyRootFilesystem":false},"sshKeySecret":"openclaw-ssh-key","syncInterval":60}` | ------------------------------------------------------------------------- |
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
