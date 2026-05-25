# Hermes4Friends Skills

Multi-tenant agent platform skills for [Hermes Agent](https://github.com/NousResearch/hermes-agent). These skills teach Hermes how to operate inside hardened Docker containers with Docker-in-Docker, provision new friend agents, and manage the platform.

## Skills

| Skill | Purpose |
|---|---|
| `hermes-container-environment` | How to work inside the restricted container — no sudo, no apt, dind is your power |
| `hermes-dind-patterns` | Docker-in-Docker workflows: sub-containers, images, networking, volumes |
| `hermes-multi-tenant-onboarding` | Provisioning new friends: compose generation, identity keys, launch, decommission |

## Install

### As a Hermes Tap

```bash
hermes skills tap add Hermes4Friends/skills
hermes skills install Hermes4Friends/skills/hermes-container-environment
```

### During Container Build

```bash
git clone https://github.com/Hermes4Friends/skills.git /tmp/skills
cp -r /tmp/skills/* ~/.hermes/skills/
```

### Via Dockerfile

```dockerfile
RUN git clone https://github.com/Hermes4Friends/skills.git /opt/hermes-skills
RUN cp -r /opt/hermes-skills/* ~/.hermes/skills/
```

## Structure

```
skills/
├── hermes-container-environment/
│   └── SKILL.md
├── hermes-dind-patterns/
│   └── SKILL.md
├── hermes-multi-tenant-onboarding/
│   └── SKILL.md
└── README.md
```

Each skill is a directory with a `SKILL.md` — the standard Hermes skills format. Skills may also include `scripts/`, `references/`, or `templates/` subdirectories as they grow.

## Contributing

Skills evolve with the platform. When you discover a new pattern or fix a pitfall, PR it back here. Skills loaded from this repo get patched on the next `hermes skills update`.
