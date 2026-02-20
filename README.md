# openclaw-with-brains-chart

Helm chart repository for deploying [OpenClaw](https://github.com/openclaw/openclaw)
with a git-backed brain repository.

## Charts

| Chart | Description |
|-------|-------------|
| [openclaw-with-brain](./charts/openclaw-with-brain/README.md) | OpenClaw AI agent with bidirectional Git workspace sync |

## Usage

```bash
helm repo add openclaw-with-brain https://filipegalo.github.io/openclaw-with-brains-chart
helm repo update
helm install openclaw openclaw-with-brain/openclaw-with-brain --namespace openclaw --create-namespace
```

## Development

### Prerequisites

- [Helm](https://helm.sh/docs/intro/install/) ≥ 3.10
- [helm-docs](https://github.com/norwoodj/helm-docs) ≥ 1.14
- [pre-commit](https://pre-commit.com/) ≥ 3.x

### Setup

```bash
pre-commit install
```

### Lint

```bash
helm lint charts/openclaw-with-brain
```

### Generate docs

```bash
helm-docs --chart-search-root charts
```

## Release Process

Releases are automated via GitHub Actions when changes are pushed to `main`
under the `charts/` directory. The release workflow:

1. Bumps the patch version if a release tag already exists
2. Runs `helm-docs` and commits updated docs
3. Generates `values.schema.json`
4. Packages and signs the chart with GPG
5. Publishes to GitHub Pages via chart-releaser
6. Pushes the OCI image to GHCR (`ghcr.io/filipegalo/openclaw-with-brain`)
