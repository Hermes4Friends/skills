---
name: hermes-container-environment
description: How to operate inside the hardened Hermes multi-tenant container — no sudo, no local dockerd, Docker-in-Docker available, Cloudflare tunnels for exposure.
version: 1.0.0
author: Hermes4Friends
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [infrastructure, multi-tenant, docker, container, environment]
    related_skills: [hermes-dind-patterns]
---

# Hermes Container Environment

You run inside a **hardened Hermes container** with deliberate restrictions. Everything that looks "broken" is actually by design. You have MORE power than a normal container — it's just routed through Docker-in-Docker.

## Core Environment Facts

| What You Can't Do | Why | What You Do Instead |
|---|---|---|
| `sudo` / `su` | No sudo binary, no root. UID 10000 non-root. | Launch a dind container — you're root there. |
| `apt-get install` | Read-only rootfs, no lock permissions. | `docker run` an image that has the tool, or build one. |
| `systemctl` / `service` | No init system. | Run daemons in their own dind containers. |
| Local `dockerd` | No Docker socket in this container. | **dind-executor on the Docker network.** |

## Directory Architecture

The container filesystem is deliberately partitioned — some paths are read-only for security, others are writable and survive restarts.

```
/                          Read-only rootfs (security hardening)
├── /opt/data/             HERMES_HOME — writable Docker volume [survives restart]
│   ├── .hermes/           Agent state directory (writable)
│   │   ├── config.yaml    ❌ RO (bind-mounted from host, secrets)
│   │   ├── .env           ❌ RO (bind-mounted from host, API keys)
│   │   ├── skills/        ✅ RW (Docker volume — skill updates, taps, hub state)
│   │   ├── sessions/      ✅ RW (Docker volume — conversation state)
│   │   ├── state.db       ✅ RW (session database)
│   │   ├── memories/      ✅ RW (persistent memory)
│   │   └── logs/          ✅ RW (agent + gateway logs)
│   └── workspace/         ✅ RW (Docker volume — project files)
│
├── /home/hermes/          ✅ RW (bind-mounted → Storage Box SMB share)
│   ├── projects/          User project files (shared with other friends)
│   ├── data/              User data (shared with other friends)
│   └── *.md               User documents
│
├── /tmp/                  tmpfs — LOST ON RESTART
└── /var/                  Read-only rootfs
```

### What You DO Have

```
Docker-in-Docker:    tcp://dind:2375   (on hermes-net)
Persistent home:     /home/hermes/     (Storage Box SMB — shared, survives everything)
Agent state:         ~/.hermes/        (local Docker volume — survives container restart)
Cloudflare tunnels:  cloudflared       (pre-installed or installable)
Network:             hermes-net        (shared with dind, gluetun VPN)
Python:              python3 + pip     (userspace tooling)
Git:                 git               (clone, commit, push)
```

### Read-Only vs Read-Write

| Path | Writable | Backed By | Survives |
|---|---|---|---|
| `/opt/data/.hermes/` | ✅ Yes | Docker volume | Container restart |
| `config.yaml` / `.env` | ❌ No (file mounts) | Host filesystem | N/A (read-only) |
| `/opt/data/skills/` | ✅ Yes | Docker volume | Container restart |
| `/opt/data/sessions/` | ✅ Yes | Docker volume | Container restart |
| `/home/hermes/` | ✅ Yes | Storage Box SMB | Everything — even host rebuild |
| `/tmp/` | ✅ Yes | tmpfs | ❌ Gone on restart |
| `/` (rootfs) | ❌ No | Image layer | ❌ (immutable) |

### Storage Box Details

`/home/hermes/` is mounted from a Hetzner Storage Box via SMB/CIFS. This means:

- **Shared across friend containers** — `/home/alice/` and `/home/bob/` are on the same Storage Box
- **Max 5 concurrent SMB connections** — cache aggressively, don't poll
- **Slower than local NVMe** — use `~/.hermes/` (local volume) for sessions, state, logs. Use `/home/hermes/` for user files, projects, data.
- **Survives everything** — host rebuild, container recreate, volume prune. Storage Box is the durable layer.

## First Thing: Set Your Docker Host

```bash
export DOCKER_HOST=tcp://dind:2375
```

Add to `~/.bashrc` or `~/.profile` so it persists. Every Docker command in this document assumes this is set.

## Pattern 1: Install a Package

You can't `apt-get`. You run a container that has the tool.

```bash
# Need jq?
docker run --rm -v $(pwd):/work alpine sh -c "apk add jq && cat /work/data.json | jq ."

# Need to keep it around? Build a tool image.
cat > /tmp/Dockerfile.tools << 'EOF'
FROM alpine:latest
RUN apk add --no-cache jq curl yq
ENTRYPOINT ["/bin/sh"]
EOF

docker build -t my-tools -f /tmp/Dockerfile.tools .
docker run --rm -v $(pwd):/work my-tools -c "jq . /work/data.json"
```

## Pattern 2: Build and Run a Service

Standard Docker workflow, just through dind.

```bash
docker build -t my-app /home/hermes/project/
docker run -d --name my-app -p 8080:80 my-app
```

The service is now reachable inside `hermes-net` at `http://my-app:80` or from the host at `http://localhost:8080`.

## Pattern 3: Expose to the Internet (Cloudflare Tunnel)

```bash
# Quick tunnel (ephemeral, random URL):
cloudflared tunnel --url http://localhost:8080 &

# Named tunnel (persistent URL, requires Cloudflare login):
cloudflared tunnel login
cloudflared tunnel create my-tunnel
# ... configure DNS in Cloudflare dashboard ...
cloudflared tunnel run my-tunnel
```

Quick tunnels are perfect for demos and testing. Named tunnels are for production. The `cloudflared` binary may be at `/tmp/cloudflared` if not installed system-wide.

## Pattern 4: Persistent File Storage

Only `/home/hermes/` survives container restarts. Mount it into dind containers:

```bash
docker run --rm -v /home/hermes/data:/data alpine sh -c "echo persisted > /data/proof.txt"
```

**Do not** store important data in `/tmp/`, `/var/`, or anywhere outside `/home/hermes/`.

## Pattern 5: Run a Python Script with Dependencies

```bash
# Option A: Install deps in a container
docker run --rm -v /home/hermes/scripts:/scripts python:3.12-alpine \
  sh -c "pip install requests && python /scripts/myscript.py"

# Option B: Build an image
echo 'FROM python:3.12-alpine
RUN pip install requests beautifulsoup4
WORKDIR /app' > Dockerfile.python

docker build -t python-tools -f Dockerfile.python .
docker run --rm -v /home/hermes/scripts:/app python-tools python myscript.py
```

## What NOT To Do

- **Don't try `sudo`** — it doesn't exist. You'll waste turns.
- **Don't try `apt-get` on the host** — read-only rootfs blocks it.
- **Don't try to install system Python packages globally** — use containerized Python.
- **Don't download tools to `/tmp/` expecting them to persist** — `/tmp/` is tmpfs.
- **Don't try `systemctl`** — no systemd. Run daemons as containers.
- **Don't report the environment as "broken"** — it's deliberately restricted. You have dind. Use it.

## Quick Health Check

Run this to confirm everything is wired:

```bash
export DOCKER_HOST=tcp://dind:2375
docker version          # Should show client AND server versions
docker run --rm hello-world  # Confirms dind is working
```

If `docker version` only shows Client and errors on Server, dind is down or unreachable. Check `hermes-net` connectivity and dind container status.

## Environment Variables Reference

| Variable | Typical Value | Purpose |
|---|---|---|
| `DOCKER_HOST` | `tcp://dind:2375` | Docker API endpoint |
| `HOME` | `/home/hermes` | Persistent home directory |
| `HERMES_SKILL_DIR` | Auto-set | Path to this skill's directory |

## Verification

After following any pattern, confirm:
1. Docker commands return without errors
2. Files written to `/home/hermes/` — run `ls /home/hermes/` to verify
3. Services respond to `curl http://localhost:PORT`
4. Cloudflare tunnels show "Registered tunnel connection" in logs
