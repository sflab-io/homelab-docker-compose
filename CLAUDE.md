# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

A collection of Docker Compose service stacks for a homelab. Each service lives in its own subdirectory (`<service-name>-docker-compose/`) with an isolated Compose file and credential management.

## Common Commands

```bash
# Install tooling for a service directory
mise install

# Run a service stack
docker compose up -d

# Check secrets for accidental leaks before committing
pre-commit run --all-files
```

## Architecture

### Directory Structure per Service

Each service subdirectory follows this layout:

```
<service>-docker-compose/
├── compose.yml              # Main Compose file
├── .gitignore               # Excludes data/, .env, unencrypted secrets
├── .sops.yaml               # SOPS age encryption config
├── .creds.env.yaml          # SOPS-encrypted credentials (committed)
└── mise.toml                # Loads .env, ~/.env, and .creds.env.yaml into shell
```

### Secrets & Credentials

Secrets are managed with [SOPS](https://github.com/getsops/sops) + [age](https://age-encryption.org/):

- `.creds.env.yaml` is encrypted with the age public key defined in `.sops.yaml`
- `mise.toml` loads it automatically via `_.file` with `redact = true`
- Required secrets in Compose files use `${VAR:?error message}` — the stack will fail to start if they are missing
- Optional env vars use `${VAR:-default}`

### Networking & Traefik

Services that need external access attach to an external `proxy` Docker network and declare Traefik labels:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.<name>.rule=Host(`<service>.docker.home.sflab.io`)"
  - "traefik.http.routers.<name>.entrypoints=websecure"
  - "traefik.http.routers.<name>.tls.certresolver=vault"
networks:
  - proxy

networks:
  proxy:
    external: true
```

Internal service communication uses a dedicated bridge network per stack (e.g. `backend`, `semaphore_network`, `scanopy`).

### Container Naming Convention

Containers are named `<directory-name>-<service-name>`, e.g. `authentik-docker-compose-server`.

### Pre-commit Hook

`gitleaks` runs on every commit via `.pre-commit-config.yaml` to prevent accidental secret commits. Never bypass with `--no-verify`.
