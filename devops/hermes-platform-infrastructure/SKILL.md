---
name: hermes-platform-infrastructure
description: Shared host-level infrastructure for the Hermes multi-tenant platform — Langfuse observability, Storage Box mounts, Nextcloud integration, Coolify DNS, and provisioning scripts.
version: 1.0.0
author: Bane (totalwindupflightsystems)
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [infrastructure, multi-tenant, langfuse, storage, nextcloud, coolify, observability]
    related_skills: [hermes-docker-deployment, hermes-multi-tenant-onboarding]
---

# Hermes Platform Infrastructure

Shared host-level services that run BEFORE and BESIDE friend containers. These are deployed once, shared by all tenants.

## Infrastructure Map

```
Hetzner VPS (37.27.250.128)
├── Coolify (:8000)             Deployment orchestrator (wildcard: *.hermes4friends.wojons.ai)
├── Langfuse (:3000)            Observability — traces, costs, per-friend visibility
├── Storage Box (SMB mount)     Durable shared storage at /mnt/storage-box/
├── Nextcloud                   File access for friends (mounts same Storage Box share)
├── Hermes Demo Stack           The template three-container setup (gluetun + dind + hermes)
└── Friend Stacks               /opt/hermes-<friend>/ per friend
```

## 1. Coolify Wildcard DNS

Coolify uses **Traefik** as its proxy (not Caddy). Wildcard setup requires:

**Cloudflare DNS record:**
- Type: A, Name: `*`, Target: `37.27.250.128`
- Proxy: **Gray cloud** (DNS only). Cloudflare only proxies wildcards on Enterprise plan.
- Traefik handles SSL via Let's Encrypt.

**Wildcard domain in Coolify DB:**
```sql
UPDATE server_settings SET wildcard_domain = 'hermes4friends.wojons.ai' WHERE server_id = 0;
```

**Cloudflare API token** (for DNS-01 wildcard certs):
- Permission: Zone → DNS → Edit
- Zone: restricted to the specific domain

Without the token: per-domain certs via HTTP-01 ✅ (works for individual FQDNs). With the token: one wildcard cert for all subdomains.

**HTTP-01 per-domain FQDN** (works without Cloudflare token):
```sql
UPDATE service_applications 
SET fqdn = 'service.hermes4friends.wojons.ai', required_fqdn = true
WHERE name = 'service-name';
```

Then restart the service via API: `POST /api/v1/services/<uuid>/restart`

## 2. Langfuse Observability

> **Step-by-step deployment guide:** `references/langfuse-deployment.md` — official docker-compose, v3 ClickHouse requirements, healthcheck fixes, API key generation, per-friend project setup.

### Architecture

One Langfuse deployment serves all friends. Each friend gets their own **project** with per-project API keys. Friends only see their own traces, usage, and costs.

```
Langfuse Server (:3000)
├── Project: Hermes Demo
│   └── API keys: pk-lf-... / sk-lf-...
├── Project: Alice
│   └── API keys: pk-lf-... / sk-lf-...
└── Project: Bob
    └── API keys: pk-lf-... / sk-lf-...
```

### Deployment

Uses the official Langfuse docker-compose (v3). Requires ClickHouse — v3 moved from Postgres-only to Postgres + ClickHouse for analytics. See `references/langfuse-deployment.md` for the full compose, healthcheck fixes, and env var reference.

### Wiring Hermes

The built-in `observability/langfuse` plugin connects Hermes as a client. Enable per friend:

```bash
# 1. Create a Langfuse project for the friend (via UI or API)
# 2. Set env vars in the friend's docker-compose:
environment:
  - HERMES_LANGFUSE_PUBLIC_KEY=pk-lf-...
  - HERMES_LANGFUSE_SECRET_KEY=sk-lf-...
  - HERMES_LANGFUSE_BASE_URL=http://<langfuse-host>:3000

# 3. Enable the plugin in the friend's Hermes container:
hermes plugins enable observability/langfuse
```

The agent writes these to its own `.env` on first boot. Traces flow immediately — LLM calls, tool usage, turn timing, token counts.

### Per-Tenant Visibility

Because each friend has their own project + API keys:
- **Langfuse UI**: Alice logs in, sees only her traces
- **Usage dashboards**: Scoped to project, not global
- **Cost tracking**: Per-project, automatic with DeepSeek/OpenAI pricing
- **Revocation**: Delete the project → all traces gone

## 3. Storage Box (Hetzner SMB)

Backs `/home/<user>/` for every friend. Shared between hermes-agent containers, dind containers, and Nextcloud.

### Host Mount

```bash
# /etc/fstab
//uXXXXX.your-storagebox.de/backup /mnt/storage-box cifs \
  username=uXXXXX,password=...,uid=1000,gid=1000,file_mode=0755,dir_mode=0755,vers=3.0,cache=strict 0 0
```

### Container Bind Mounts

Both the hermes-agent container AND the dind container mount the same path:

```yaml
# In docker-compose.yml for hermes-<friend>:
services:
  hermes:
    volumes:
      - /mnt/storage-box/hermes-<friend>:/home/hermes
  dind:
    volumes:
      - /mnt/storage-box/hermes-<friend>:/home/hermes:ro  # dind mounts RO for safety
```

This gives dind containers read access to friend files. If dind needs write access (e.g., building Docker images from friend projects), use rw and rely on filesystem permissions.

### Constraints

- **Max 5 concurrent SMB connections** — all containers + Nextcloud share this budget
- **Slower than NVMe** — sessions, state.db, and logs stay on local Docker volumes
- **Survives everything** — host rebuild, container recreate, volume prune

## 4. Nextcloud Integration

Nextcloud mounts the same Storage Box share so friends see their AI-generated files through a familiar interface.

### Setup

1. Deploy Nextcloud (Coolify one-click or docker compose)
2. Mount `/mnt/storage-box/` into the Nextcloud container
3. Configure Nextcloud "External Storage" app to point at the mount
4. Each friend gets a Nextcloud account mapped to their Storage Box subdirectory

## 5. Email Provider (Agent Inboxes)

Friends' agents need IMAP/SMTP inboxes. Requirements: cheap, unlimited accounts, per-GB pricing (not per-account), IMAP access.

**Recommendation: MXRoute** (~$25/year, 10GB, unlimited accounts/domains)

| Provider | Pricing | Accounts | Notes |
|---|---|---|---|
| MXRoute | ~$25/yr (10GB) | Unlimited | Best fit — DirectAdmin, mature since 2013, Black Friday ~$10-15/yr |
| Migadu | ~$19/yr (5GB soft) | Unlimited domains/mailboxes | Clean UI, soft storage limits |
| Purelymail | $10/yr + usage billing | Unlimited | Cheapest for very low volume, newer |

**Setup pattern:**
```
hermes4friends.wojons.ai   → MX DNS pointed to MXRoute
├── alice@hermes4friends.wojons.ai   → Alice's agent inbox
├── bob@hermes4friends.wojons.ai     → Bob's agent inbox
└── admin@hermes4friends.wojons.ai   → Platform admin
```

Hermes uses **Himalaya CLI** for IMAP/SMTP (already available in the Hermes email tools). Agents read/send email programmatically.

## 6. Provisioning Script

```bash
#!/bin/bash
FRIEND=$1
EMAIL=$2
TIER=${3:-standard}

# 1. Create Nextcloud user
# 2. Create Storage Box directory: /mnt/storage-box/hermes-$FRIEND/
# 3. Generate docker-compose from template
# 4. Generate Ed25519 identity keys
# 5. Create Langfuse project + API keys
# 6. Launch containers
# 7. Send welcome message

echo "Onboarding $FRIEND ($TIER)..."
```

This ensures no half-provisioned state — the friend gets Nextcloud + Storage + Hermes + Langfuse in one atomic operation.

## Verification

```bash
# Langfuse health:
curl http://localhost:3000/api/public/health
# → {"status":"OK","version":"3.175.0"}

# Storage Box mount:
df -h /mnt/storage-box/
# → Should show the SMB share with expected capacity

# Coolify:
curl http://localhost:8000
# → Coolify login page

# Container storage visibility:
docker exec hermes-brain ls /home/hermes/
# → Should show friend's files
```

## Pitfalls

- **Langfuse v3 requires ClickHouse** — the v2 Postgres-only setup no longer works. Use the official docker-compose from `github.com/langfuse/langfuse`.
- **ClickHouse healthcheck quoting** — `CMD-SHELL` with double-quoted passwords breaks. Use `CMD` array format: `["CMD", "clickhouse-client", "-u", "user", "--password", "pass", "-q", "SELECT 1"]`.
- **Storage Box SMB connection limit** — if containers + Nextcloud exceed 5 connections, mounts fail silently. Monitor with `smbstatus` on the Storage Box.
- **DNS-only wildcard** — Cloudflare won't proxy wildcards. Coolify's Traefik handles SSL termination on the host — that's the intended design, not a limitation.
- **Coolify API gaps** — FQDN assignment for Docker Compose sub-apps requires DB-level update (REST returns 404). See `references/coolify-api-quirks.md`.
- **Langfuse v3 single-node** — must set `CLICKHOUSE_SINGLE_NODE_URL` or migrations fail with `ON CLUSTER default`. See `references/langfuse-deployment.md`.

## Support Files

| File | Purpose |
|---|---|
| `references/langfuse-deployment.md` | Langfuse v3 deployment guide — Coolify + raw Docker, ClickHouse fixes, per-friend setup |
| `references/coolify-api-quirks.md` | Coolify API gotchas — FQDN via DB, base64 compose, wildcard domain |
| `scripts/provision-friend.sh` | Atomic friend provisioning — Storage Box + keys + compose + launch |
