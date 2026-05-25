---
name: hermes-multi-tenant-onboarding
description: How to provision and onboard new friends into the Hermes multi-tenant platform — Docker Compose generation, identity keys, container launch, and welcome flow.
version: 1.0.0
author: Hermes4Friends
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [multi-tenant, onboarding, provisioning, identity, docker]
    related_skills: [hermes-container-environment, hermes-dind-patterns]
---

# Multi-Tenant Onboarding

This skill covers the full lifecycle of bringing a new friend onto the Hermes platform — from initial request to working agent.

## Architecture Recap

Each friend gets three containers:
```
gluetun (VPN) → dind (Docker executor) → hermes-agent (the brain)
```

All three share `hermes-net`. The Hermes container has no root, no sudo, no local Docker — everything goes through dind at `tcp://dind:2375`.

## Onboarding Flow

### Step 1: Receive the Request

Source can be:
- **Nextcloud group addition** (trigger: user added to `hermes-users` group)
- **Coolify API webhook** (manual or automated)
- **Direct request** (someone asks in chat)

Collect:
- Friend's name (used as container prefix: `hermes-<name>`)
- Preferred messaging platform (Telegram, Discord, etc.)
- Trust tier: `basic` (restricted tools) or `full` (owner-level)

### Step 2: Generate Identity Keys

Each friend gets an Ed25519 keypair for cryptographic identity:

```bash
# Inside the dind executor or from host:
docker run --rm -v /home/hermes/keys:/keys alpine sh -c "
  apk add openssh-keygen &&
  ssh-keygen -t ed25519 -f /keys/hermes-<friend>-id_ed25519 -N '' -C 'hermes-<friend>'
"
```

The private key stays in the friend's `~/.hermes/keys/` directory. The public key is registered in the platform's identity registry.

### Step 3: Generate Docker Compose

Create `/opt/hermes-<friend>/docker-compose.yml`:

```yaml
version: "3.8"
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun-<friend>
    cap_add:
      - NET_ADMIN
    environment:
      - VPN_SERVICE_PROVIDER=surfshark
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - WIREGUARD_ADDRESSES=10.14.0.2/16
      - SERVER_REGIONS=us-nyc
    networks:
      - hermes-net
    restart: unless-stopped

  dind:
    image: docker:28-dind
    container_name: dind-<friend>
    privileged: true
    environment:
      - DOCKER_TLS_CERTDIR=
    networks:
      - hermes-net
    volumes:
      - dind-<friend>-data:/var/lib/docker
    restart: unless-stopped

  hermes:
    image: hermes-agent:latest
    container_name: hermes-<friend>
    user: "2001:2001"
    environment:
      - HERMES_HOME=/home/hermes/.hermes
      - DOCKER_HOST=tcp://dind:2375
      # API keys injected as env vars (agent writes its own .env + config.yaml)
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
      - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY}
    volumes:
      - ./hermes-home:/home/hermes
      - ./keys:/home/hermes/.hermes/keys:ro
      - hermes-<friend>-sessions:/home/hermes/.hermes/sessions
      - hermes-<friend>-skills:/home/hermes/.hermes/skills
      - hermes-<friend>-state:/home/hermes/.hermes
    depends_on:
      gluetun:
        condition: service_healthy
      dind:
        condition: service_started

networks:
  hermes-net:
    external: true

volumes:
  dind-<friend>-data:
```

Replace `<friend>` with the friend's name throughout.

### Step 4: Launch (Agent Self-Configures)

The agent starts with API keys injected as environment variables. On first boot, it writes its own `~/.hermes/config.yaml` and `~/.hermes/.env`. No pre-baked config files needed — the agent manages itself.

```bash
cd /opt/hermes-<friend>
docker compose -p hermes-<friend> up -d

# Verify:
docker ps --filter name=hermes-<friend>
docker logs hermes-<friend>
```

### Step 5: Pre-Seed Skills Volume

Before first launch, clone the platform skills into the skills volume so the agent knows its environment:

```bash
git clone https://github.com/Hermes4Friends/skills.git /tmp/h4f-skills
cp -r /tmp/h4f-skills/hermes-* /var/lib/docker/volumes/hermes-<friend>_hermes-<friend>-skills/_data/
rm -rf /tmp/h4f-skills
```

Without this, the agent's first turn will diagnose the hardened container as "broken" because it doesn't know about dind.

### Step 6: Send Welcome Message

After the agent is running, trigger a welcome via the gateway:

```
Welcome to your Hermes agent! 
- You're on the <tier> tier
- Your container: hermes-<friend>
- Your identity key fingerprint: <sha256>
- Type /help to see what you can do
```

## Per-Friend Configuration Checklist

| Item | Location | Purpose |
|---|---|---|
| `docker-compose.yml` | `/opt/hermes-<friend>/` | Container stack |
| `hermes-config/config.yaml` | Bind-mounted | Agent settings |
| `hermes-config/.env` | Bind-mounted | API keys |
| `keys/id_ed25519` | Bind-mounted (ro) | Cryptographic identity |
| `hermes-home/` | Bind-mounted | Persistent files |
| `~/.hermes/profiles/friend-<friend>/` | On container filesystem | Profile data |

## Trust Tiers

| Tier | Toolsets | Terminal Backend | LLM Budget/mo |
|---|---|---|---|
| `basic` | web, skills, clarify | dind-only (no local terminal) | $10-20 |
| `standard` | + terminal, file, session_search | dind + local (read-only rootfs) | $20-35 |
| `full` | all tools, delegation | full access | $35-50 |

## Decommissioning a Friend

```bash
cd /opt/hermes-<friend>
docker compose -p hermes-<friend> down -v   # Stops + removes volumes
rm -rf /opt/hermes-<friend>                 # Removes config
# Revoke API key from LiteLLM if using virtual keys
# Remove from Nextcloud group if using group trigger
```

## Monitoring

```bash
# Check resource usage per friend:
docker stats --filter name=hermes-<friend>

# Check if agent is responsive:
docker exec hermes-<friend> curl -s http://localhost:8080/health || echo "DOWN"

# View recent logs:
docker logs --tail 50 hermes-<friend>
```

## Pitfalls

- **Don't forget `hermes-net: external: true`** — creating a new network isolates the friend from dind
- **WireGuard key reuse** — generate unique keys per friend or they share bandwidth limits
- **Volume naming collision** — always prefix volumes with the friend name
- **Port clashes** — if exposing ports, each friend needs unique host ports
- **The agent will think it can't sudo** — load the `hermes-container-environment` skill on first launch
