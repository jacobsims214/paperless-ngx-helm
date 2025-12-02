# Paperless-NGX Helm Chart

A Helm chart for deploying [Paperless-NGX](https://docs.paperless-ngx.com/) - a document management system that transforms your physical documents into a searchable online archive.

## Features

- 📄 OCR and full-text search for all documents
- 🏷️ Automatic tagging and classification
- 📁 Correspondent and document type organization
- 🔒 Secure access via Tailscale (tailnet-only, no public Funnel)
- 🐘 PostgreSQL database (via CloudNative-PG operator)
- ⚡ Valkey for task queue (shared cluster)

## Prerequisites

1. **Kubernetes cluster** with Helm 3+
2. **CloudNative-PG operator** - for PostgreSQL
3. **Valkey operator** - for Redis-compatible cache
4. **Tailscale account** with Funnel enabled
5. **Storage class** for persistent volumes

## Quick Start

### 1. Create the Namespace

```bash
kubectl create namespace paperless-ngx
```

### 2. Deploy PostgreSQL (CloudNative-PG)

Create a PostgreSQL cluster for Paperless in the same namespace:

```yaml
# paperlessngx-db-cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: paperlessngx-db-cluster
  namespace: paperless-ngx
spec:
  instances: 1
  storage:
    size: 10Gi
  bootstrap:
    initdb:
      database: app
      owner: app
```

```bash
kubectl apply -f paperlessngx-db-cluster.yaml
```

This creates the secret `paperlessngx-db-cluster-app` with `username` and `password` keys.

### 3. Valkey (Shared Cluster)

Uses the existing Valkey instance in the `valkey` namespace:
- Service: `valkey.valkey.svc.cluster.local:6379`
- No authentication required

### 4. Create Secrets

```bash
# Create Paperless secrets (admin password, secret key)
cd pve-1/tools/apps
./create-paperless-secrets.sh

# Create Tailscale auth secret
./create-paperless-tailscale-secret.sh tskey-auth-xxxxx
```

### 5. Configure Tailscale ACLs

Add to your [Tailscale ACLs](https://login.tailscale.com/admin/acls):

```json
{
  "tagOwners": {
    "tag:paperless-ngx": ["autogroup:admin"]
  }
}
```

> **Note:** No Funnel attribute needed - this service is tailnet-only.

### 6. Deploy Paperless-NGX

```bash
cd pve-1/paperless-ngx-helm
helm install paperless ./charts/paperless-ngx -n paperless-ngx
```

### 7. Access (Tailnet Only)

- **URL**: `https://paperless.tail108d23.ts.net`
- **Username**: `admin`
- **Password**: (from create-paperless-secrets.sh output)

⚠️ **This URL only works from devices connected to your Tailscale network.**

## Configuration

### External Services

| Parameter | Description | Default |
|-----------|-------------|---------|
| `postgresql.host` | PostgreSQL hostname | `paperlessngx-db-cluster-rw.paperless-ngx.svc.cluster.local` |
| `postgresql.existingSecret.name` | Secret with DB credentials | `paperlessngx-db-cluster-app` |
| `redis.url` | Valkey URL (no auth) | `redis://valkey.valkey.svc.cluster.local:6379` |

### Tailscale (Tailnet-Only)

| Parameter | Description | Default |
|-----------|-------------|---------|
| `tailscale.enabled` | Enable Tailscale | `true` |
| `tailscale.hostname` | Tailscale machine name | `paperless` |
| `tailscale.domain` | Tailscale domain | `tail108d23.ts.net` |
| `tailscale.tags` | ACL tags | `tag:paperless-ngx` |
| `tailscale.funnel` | Enable public Funnel | `false` |

### Application

| Parameter | Description | Default |
|-----------|-------------|---------|
| `paperless.url` | Public URL | `https://paperless.tail108d23.ts.net` |
| `paperless.timezone` | Timezone | `America/New_York` |
| `paperless.ocr.language` | OCR language | `eng` |
| `paperless.admin.username` | Admin username | `admin` |

### Storage

| Parameter | Description | Default |
|-----------|-------------|---------|
| `persistence.data.size` | Data volume size | `5Gi` |
| `persistence.media.size` | Media volume size | `50Gi` |
| `persistence.consume.size` | Consume volume size | `5Gi` |
| `persistence.export.size` | Export volume size | `10Gi` |

## Consuming Documents

### Via Web UI

Upload documents directly through the web interface.

### Via Consume Folder

Documents placed in the consume volume are automatically processed:

```bash
# Copy a document to the consume folder
kubectl cp mydocument.pdf paperless/paperless-ngx-xxx:/usr/src/paperless/consume/
```

### Via Email (Future)

Configure email rules in the Paperless UI to automatically consume documents from email.

## Integrating with ntfy

Configure Paperless workflows to send notifications to ntfy:

1. In Paperless UI, go to **Admin** → **Workflows**
2. Create a workflow with **Webhook** action
3. Set URL to: `http://ntfy.ntfy.svc.cluster.local/paperless`
4. Configure trigger (e.g., on document added)

## Backup

### Export Documents

```bash
kubectl exec -n paperless deployment/paperless-ngx -- \
  document_exporter /usr/src/paperless/export
```

### Backup PostgreSQL

```bash
kubectl cnpg backup paperless-psql-cluster -n paperless
```

## Troubleshooting

### Check Logs

```bash
# Main application
kubectl logs -n paperless-ngx deployment/paperless-ngx -c paperless-ngx

# Tailscale sidecar
kubectl logs -n paperless-ngx deployment/paperless-ngx -c tailscale
```

### Database Connection

```bash
kubectl exec -n paperless-ngx deployment/paperless-ngx -- \
  python manage.py dbshell
```

### OCR Issues

Check if Tesseract languages are installed:

```bash
kubectl exec -n paperless-ngx deployment/paperless-ngx -- \
  tesseract --list-langs
```

## Uninstall

```bash
helm uninstall paperless -n paperless-ngx
kubectl delete pvc -n paperless-ngx --all
kubectl delete namespace paperless-ngx
```

