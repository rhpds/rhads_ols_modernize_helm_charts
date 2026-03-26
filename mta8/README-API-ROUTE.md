# MTA8 API Route Configuration

## Overview

The MTA8 Helm chart now supports creating a direct OpenShift route to the MTA Hub API. This allows programmatic access to the MTA REST API without requiring OAuth proxy authentication.

## Configuration

### Enable API Route

Add the following to your `values.yaml`:

```yaml
apiRoute:
  enabled: true
  routeName: mta-api          # Name of the route
  serviceName: mta-hub         # Service to route to (default: mta-hub)
  targetPort: api              # Service port (default: api)
```

### Disable Authentication (Optional)

For open API access, disable Tackle authentication:

```yaml
tackleInstance:
  featureAuthRequired: false
```

## Usage Examples

### Deploy with API Route Enabled

```bash
helm upgrade --install mta8 . \
  --namespace user-m8k26-mta \
  --set apiRoute.enabled=true \
  --set tackleInstance.featureAuthRequired=false
```

### Using Values File

```bash
helm upgrade --install mta8 . \
  --namespace user-m8k26-mta \
  -f examples/with-api-route.yaml
```

## Testing the API

Once deployed, the API will be available at:

```
https://mta-api-<namespace>.apps.<cluster-domain>/
```

### Example: List Applications

```bash
# Get the API URL
API_URL=$(oc get route mta-api -n user-m8k26-mta -o jsonpath='{.spec.host}')

# Test the API
curl -k https://$API_URL/applications
```

### With Authentication (if featureAuthRequired: true)

```bash
TOKEN=$(oc whoami -t)
curl -k -H "Authorization: Bearer $TOKEN" https://$API_URL/applications
```

### Common API Endpoints

```bash
# Applications
GET    /applications
POST   /applications
GET    /applications/{id}
PUT    /applications/{id}
DELETE /applications/{id}

# Tasks
GET    /tasks
POST   /tasks
GET    /tasks/{id}

# Assessments
GET    /assessments
POST   /assessments

# Business Services
GET    /businessservices
POST   /businessservices

# Tags
GET    /tags
GET    /tagcategories
```

## Architecture

```
┌─────────────┐
│   Client    │
└─────┬───────┘
      │
      │ HTTPS (Edge TLS)
      │
      ▼
┌─────────────────┐
│  mta-api Route  │  (Created by Helm)
└─────┬───────────┘
      │
      ▼
┌─────────────────┐
│ mta-hub Service │  (Port: api)
└─────┬───────────┘
      │
      ▼
┌─────────────────┐
│  mta-hub Pod    │  (MTA Hub API)
└─────────────────┘
```

## Security Considerations

### With Authentication Enabled (Recommended for Production)

```yaml
apiRoute:
  enabled: true

tackleInstance:
  featureAuthRequired: true  # Requires authentication
```

Users must provide OpenShift token:
```bash
TOKEN=$(oc whoami -t)
curl -k -H "Authorization: Bearer $TOKEN" https://$API_URL/applications
```

### Without Authentication (Development/Testing Only)

```yaml
apiRoute:
  enabled: true

tackleInstance:
  featureAuthRequired: false  # No authentication required
```

Direct access without credentials:
```bash
curl -k https://$API_URL/applications
```

**⚠️ Warning:** Disabling authentication exposes the API publicly. Only use in trusted environments.

## Route vs OAuth Proxy

| Feature | API Route | OAuth Proxy Route |
|---------|-----------|-------------------|
| Authentication | Optional | Required |
| Use Case | API access | Web UI access |
| TLS Termination | Edge | Reencrypt |
| Target Service | mta-hub | mta-oauth-proxy |
| Enabled By | `apiRoute.enabled` | `oauthProxy.enabled` |

You can enable both routes simultaneously:
- `mta-api` → Direct API access (for scripts/automation)
- `mta-secure` → OAuth-protected UI access (for users)

## Comparison with Manual Route Creation

### Before (Manual)
```bash
oc create route edge mta-api \
  --service=mta-hub \
  --port=api \
  -n user-m8k26-mta
```

### After (Helm-managed)
```bash
helm upgrade --install mta8 . \
  --set apiRoute.enabled=true
```

**Benefits of Helm-managed route:**
- Declarative configuration
- Version controlled
- Consistent across deployments
- Automatic cleanup on uninstall
- Easy to enable/disable

## Troubleshooting

### Route not created

Check if enabled in values:
```bash
helm get values mta8 -n user-m8k26-mta
```

### 503 Service Unavailable

Check if MTA Hub is running:
```bash
oc get pods -n user-m8k26-mta -l app=mta-hub
```

### 404 Not Found

Verify the endpoint path (no `/hub` prefix needed):
```bash
# ✓ Correct
curl -k https://$API_URL/applications

# ✗ Wrong
curl -k https://$API_URL/hub/applications
```

## Examples

See `examples/with-api-route.yaml` for a complete configuration.
