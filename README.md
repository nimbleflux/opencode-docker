# OpenCode Docker

> **Docker image and deployment setup for [OpenCode](https://opencode.ai)**

This project provides container deployment options for running [OpenCode](https://opencode.ai) as a web service. OpenCode is an AI-powered development environment - this repository makes it easy to run in containerized environments.

The `latest` tag tracks the newest stable OpenCode v2 release. Coming from a 1.x image? See [Upgrading from a 1.x image](#upgrading-from-a-1x-image).

## What is OpenCode?

OpenCode is an AI coding assistant that provides a terminal-based interface for software development. It supports multiple AI providers (Anthropic, OpenAI, Google, AWS, etc.) and includes tools for file editing, bash commands, and project management.

**For detailed information about OpenCode features and configuration, visit [opencode.ai](https://opencode.ai)**

## Deployment Options

This repository provides:
- **Docker image** - Ready-to-use container with OpenCode and common development tools
- **Docker Compose** - Simple local deployment with persistent volumes
- **Helm chart** - Kubernetes deployment with full lifecycle management

## Quick Start

### Docker

```bash
docker run -p 4000:4000 \
  -v opencode-workspace:/home/opencode/workspace \
  ghcr.io/nimbleflux/opencode-docker:latest
```

### Docker Compose (Recommended)

```bash
# Clone the repository
git clone https://github.com/nimbleflux/opencode-docker.git
cd opencode-docker

# Start OpenCode
docker-compose up -d

# Access at http://localhost:4000
```

### Kubernetes / Helm

```bash
# Install from GHCR
helm install opencode oci://ghcr.io/nimbleflux/opencode

# Or install from local directory
helm install opencode ./helm/opencode
```

---

## Upgrading from a 1.x image

The `latest` tag now ships OpenCode v2. The `1.19.x` tags still pull the last v1 image, but v1 no longer receives updates here. Before you switch:

- **Read the upstream guide.** [Migrate from V1](https://opencode.ai/docs/migrate-v1) lists every change. The ones most likely to affect a deployment: the server API has new contracts, v1 plugins do not run in v2, and `external_directory` is allowed by default (add an `ask` permission rule to restore the v1 behaviour).
- **Back up your volumes.** Config and data stay in the same directories and v2 reads v1 config without rewriting it, but keep a copy of your v1 setup until you have checked models, credentials, agents, permissions and MCP servers.
- **Set a password.** v2 always requires one. Without `OPENCODE_SERVER_PASSWORD` (or `OPENCODE_PASSWORD`) a new password is generated on every start and printed to the container logs.
- **Pin the tag on Kubernetes.** Set `image.tag` to a specific version (for example `2.4.2`) instead of `latest`, so nodes that cached a 1.x `latest` image with `pullPolicy: IfNotPresent` do not keep running v1.

To roll back, pin `ghcr.io/nimbleflux/opencode-docker:1.19.6` and restore your backup.

---

## Configuration

**Important:** OpenCode is configured through the web UI after first launch. Your configuration is automatically persisted to the config volume.

### First-Time Setup

1. Start the container
2. Open `http://localhost:4000` in your browser
3. Configure your AI provider and API keys through the UI
4. Your settings are saved and persist across container restarts

### Environment Variables

Only a few container-level environment variables are available:

| Variable | Description | Default |
|----------|-------------|---------|
| `OPENCODE_PORT` | Port to listen on | `4000` |
| `OPENCODE_SERVER_PASSWORD` | Password for the web UI and API (the username is always `opencode`) | generated |
| `OPENCODE_PASSWORD` | Same password, under the name OpenCode v2 uses | generated |

**Note:** API keys and model configuration are configured through the OpenCode UI, not environment variables.

OpenCode always asks for a password. If neither variable is set, a new password is generated on every start and printed to the container logs (`docker logs opencode`).

### Docker Compose

Create a `.env` file in the repository root:

```bash
# Server password (a new one is generated on every start if unset)
# OPENCODE_SERVER_PASSWORD=your-secure-password

# Custom port (default: 4000)
# OPENCODE_PORT=4000
```

Then start with:

```bash
docker-compose up -d
```

### Kubernetes / Helm

The Helm chart creates persistent volumes for configuration and data. Configure via `values.yaml`:

```yaml
# Service configuration
service:
  type: ClusterIP
  port: 4000

# Ingress (optional)
ingress:
  enabled: false
  className: nginx
  hosts:
    - host: opencode.example.com
      paths:
        - path: /
          pathType: Prefix

# Resources
resources:
  limits:
    cpu: 2
    memory: 4Gi
  requests:
    cpu: 500m
    memory: 1Gi

# Extra environment variables (rarely needed)
extraEnv: []
# - name: CUSTOM_VAR
#   value: "custom-value"

# Extra volumes if needed
extraVolumes: []
extraVolumeMounts: []
```

Install with custom values:

```bash
helm install opencode ./helm/opencode -f custom-values.yaml
```

---

## Persistent Data

OpenCode stores data in three locations that should be persisted:

| Path | Purpose |
|------|---------|
| `/home/opencode/.config/opencode` | Global configuration, user preferences, and UI settings |
| `/home/opencode/.local/share/opencode` | Session storage, cache, and authentication credentials |
| `/home/opencode/workspace` | Working directory for projects |

### Volume Mount Examples

#### Minimal (workspace only)

```bash
docker run -p 4000:4000 \
  -v $(pwd)/workspace:/home/opencode/workspace \
  ghcr.io/nimbleflux/opencode-docker:latest
```

#### Full persistence (recommended)

```bash
docker run -p 4000:4000 \
  -v opencode-config:/home/opencode/.config/opencode \
  -v opencode-data:/home/opencode/.local/share/opencode \
  -v opencode-workspace:/home/opencode/workspace \
  ghcr.io/nimbleflux/opencode-docker:latest
```

---

## Container Features

- **Rootless container**: Runs as non-root user (UID 1000) for enhanced security
- **Pre-installed tools**: Common development tools included
  - **Essentials**: git, curl, bash, ca-certificates
  - **Editors**: vim, nano
  - **Utilities**: jq, yq, findutils
  - **Language runtimes**: Python 3 + pip, Node.js + npm, Go
- **Automatic permission fixing**: Entrypoint script ensures volumes have correct ownership

---

## Security

The container follows security best practices:

- **User**: `opencode` (UID 1000, non-root)
- **Group**: `opencode` (GID 1000)
- **Home directory**: `/home/opencode`
- **Workspace**: `/home/opencode/workspace`
- **Helm enforcement**: `runAsNonRoot: true`, all capabilities dropped

---

## Advanced Helm Configuration

### Gateway API Support

The chart supports Kubernetes Gateway API as an alternative to Ingress:

```yaml
gatewayAPI:
  enabled: true

  gatewaySelector:
    matchLabels:
      gateway: external

  httpRoute:
    hostnames:
      - opencode.example.com

    rules:
      - backendRefs:
          - name: opencode
            kind: Service
            port: 4000
        matches:
          - path:
              type: Prefix
              value: /
```

### Global Labels and Annotations

Apply metadata to all resources:

```yaml
# Global labels applied to all resources
globalLabels:
  environment: production
  team: platform

# Global annotations applied to all resources
globalAnnotations:
  prometheus.io/scrape: "true"
```

### Persistent Volume Claims

The Helm chart creates three PVCs (1Gi each by default):

- `<release-name>-config` - Configuration and preferences
- `<release-name>-data` - Sessions and cache
- `<release-name>-workspace` - Project workspace

Customize in `values.yaml`:

```yaml
persistence:
  enabled: true
  storageClass: ""  # Uses cluster default
  size: 1Gi

  config:
    enabled: true
    existingClaim: ""  # Use existing PVC if provided
```

### Health Probes

Configure liveness, readiness, and startup probes:

```yaml
probes:
  liveness:
    enabled: true
    httpGet:
      path: /health
      port: http
    initialDelaySeconds: 30
    periodSeconds: 10

  readiness:
    enabled: true
    httpGet:
      path: /health
      port: http
    initialDelaySeconds: 10
    periodSeconds: 5

  startup:
    enabled: true
    httpGet:
      path: /health
      port: http
    initialDelaySeconds: 5
    failureThreshold: 30
```

### Common Operations

```bash
# Upgrade release
helm upgrade opencode ./helm/opencode

# Uninstall release
helm uninstall opencode

# Port forward to access locally
kubectl port-forward svc/opencode 4000:4000

# List available Helm chart versions
helm search repo ghcr.io/nimbleflux/opencode --versions
```

---

## Backup Considerations

For production deployments, implement a backup strategy for persistent data:

- **Docker volumes**: Use `docker volume` commands or backup solutions
- **Kubernetes PVCs**: Use Velero, cloud provider backups, or regular snapshots
- **Important data**:
  - `/home/opencode/.local/share/opencode` - Active sessions and credentials
  - `/home/opencode/.config/opencode` - User preferences and provider settings
  - `/home/opencode/workspace` - Project files

---

## Development

### Automated Workflows

1. **Build and Push** (`.github/workflows/build.yml`):
   - Builds and pushes Docker images to GHCR on pushes to `main`
   - Checks daily for new stable OpenCode versions (highest released `vX.Y.Z` tag)
   - Smoke-tests each image (version, startup, API authentication) before publishing
   - Creates releases automatically

2. **Helm Chart Publish** (`.github/workflows/helm-publish.yml`):
   - Publishes Helm chart to GHCR when changes are detected in `helm/opencode/`
   - Uses semantic versioning with auto-detection based on commit messages

### Building Locally

```bash
# Build Docker image
docker build -t opencode-local .

# Build and test with Docker Compose
docker-compose up --build

# Package Helm chart
helm package helm/opencode
```

---

## Links

- **OpenCode**: https://opencode.ai
- **OpenCode Documentation**: https://opencode.ai/docs/config/
- **Docker Image**: ghcr.io/nimbleflux/opencode-docker
- **Helm Chart**: ghcr.io/nimbleflux/opencode
- **Issue Tracker**: https://github.com/nimbleflux/opencode-docker/issues

---

## License

This deployment configuration is provided as-is for running OpenCode. Please refer to the [OpenCode project](https://opencode.ai) for information about the OpenCode software license.
