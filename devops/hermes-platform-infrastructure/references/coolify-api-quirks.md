# Coolify API Quirks

Lessons from deploying services through the Coolify REST API (v4.1.0).

## Authentication

```bash
curl -H "Authorization: Bearer <api_key>" http://localhost:8000/api/v1/...
```

API keys created in Coolify UI → Settings → API Keys.

## Docker Compose Deployment

### Creating a Service

```bash
curl -X POST http://localhost:8000/api/v1/services \
  -H "Authorization: Bearer ..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "service-name",
    "project_uuid": "...",
    "environment_uuid": "...",
    "server_uuid": "...",
    "docker_compose_raw": "<base64-encoded-compose>"
  }'
```

- `docker_compose_raw` must be **base64 encoded**
- Do NOT include `type` when using `docker_compose_raw` (mutually exclusive)
- Docker Compose `.env` file is NOT auto-loaded — pass env vars via service settings or include in compose

### Service Lifecycle

```
POST /api/v1/services/<uuid>/start    # Start all containers
POST /api/v1/services/<uuid>/restart  # Restart (may stop but not restart — use start if stuck)
POST /api/v1/services/<uuid>/stop     # Stop all containers
```

There is no `deploy` endpoint — use `restart` or `start`.

## FQDN / Domain Configuration

### The Problem

Docker Compose services create sub-applications (`service_applications` table) that don't expose FQDN through the main REST API. The `PATCH /api/v1/applications/<uuid>` returns 404.

### The Fix

Set FQDN directly in the database:

```sql
-- Find the service application
SELECT id, name, uuid, fqdn FROM service_applications WHERE name = 'service-name';

-- Set FQDN
UPDATE service_applications 
SET fqdn = 'service.domain.com', required_fqdn = true
WHERE uuid = '<uuid>';

-- Restart the service for Traefik to pick up the FQDN
-- Then curl POST /api/v1/services/<uuid>/restart
```

### After FQDN Change

The service restart should regenerate Traefik labels. If not, check:

```bash
docker inspect <container> --format '{{range $k,$v := .Config.Labels}}{{$k}}={{$v}}{{end}}' | grep traefik.http.routers
```

The Traefik rule should be: `Host('service.domain.com')`

If the FQDN ended up in `PathPrefix` instead of `Host` (observed bug), use a manual Traefik dynamic config file.

## Traefik Manual Route (Escape Hatch)

When Coolify's label generation puts the FQDN in the wrong Traefik field:

```yaml
# /data/coolify/proxy/dynamic/service-name.yml
http:
  routers:
    service-name-https:
      entryPoints:
        - https
      rule: Host(`service.domain.com`)
      service: service-name-backend
      tls:
        certResolver: letsencrypt

  services:
    service-name-backend:
      loadBalancer:
        servers:
          - url: http://<container-name>:<port>
```

Then restart the Coolify proxy: `docker restart coolify-proxy`

## Wildcard Domain

```sql
-- Set wildcard domain for auto-generated subdomains
UPDATE server_settings SET wildcard_domain = 'domain.com' WHERE server_id = 0;
```

For wildcard SSL certs (DNS-01 challenge), a Cloudflare API token is required:
- Permission: Zone → DNS → Edit
- Configured in Coolify UI → Servers → Proxy → DNS Challenge

Without the token, individual domain certs work via HTTP-01.

## Server Management

```bash
# Get server details
GET /api/v1/servers/<uuid>

# Get server domains
GET /api/v1/servers/<uuid>/domains
# Returns: [{"ip":"37.27.250.128","domains":[]},{"ip":"2a01:..."}]

# Server FQDN cannot be set via PATCH (field not allowed)
# Use the wildcard_domain in server_settings instead
```

## Database Access

```bash
docker exec coolify-db psql -U coolify -d coolify
```

Key tables:
- `servers` — server config
- `server_settings` — wildcard_domain, proxy settings
- `service_applications` — per-app FQDN, ports, status
- `applications` — standalone apps (NOT Docker Compose sub-apps)
- `projects` — project hierarchy
- `environments` — per-project environments
