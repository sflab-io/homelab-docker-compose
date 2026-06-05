# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

A collection of Docker Compose service stacks for a homelab. Each service lives in its own subdirectory (`<service-name>/`) with an isolated Compose file and optional credential management.

## Common Commands

```bash
# Install tooling (from repository root or service directory)
mise install

# Run a service stack (from within the service directory)
docker compose up -d

# Check secrets for accidental leaks before committing
pre-commit run --all-files
```

## Architecture

### Repository Layout

```
homelab-docker-compose/
├── .sops.yaml               # SOPS age encryption config (shared)
├── .creds.env.yaml          # SOPS-encrypted credentials for all stacks (committed)
├── .gitignore               # Excludes .prompts/, data/, letsencrypt/, .env
├── mise.toml                # Root tooling: installs pre-commit, loads .env files
├── .pre-commit-config.yaml  # gitleaks hook
├── scanopy/
│   ├── docker-compose.yml
│   └── data/                # Runtime data (gitignored)
└── vaultwarden/
    └── docker-compose.yml
```

### Directory Structure per Service

New services only need a Compose file — all shared config lives at the root:

```
<service>/
└── docker-compose.yml       # Main Compose file
```

### Secrets & Credentials

Secrets are managed with [SOPS](https://github.com/getsops/sops) + [age](https://age-encryption.org/):

- `.creds.env.yaml` at the repository root holds all service credentials, encrypted with the age public key in `.sops.yaml`
- The root `mise.toml` loads `~/.env`, `.env`, and `.creds.env.yaml` automatically via `_.file` with `redact = true`
- Required secrets in Compose files use `${VAR:?error message}` — the stack will fail to start if they are missing
- Optional env vars use `${VAR:-default}`

### Networking & Traefik

Services handle networking individually. Two patterns are in use:

**Bundled Traefik** (e.g. `vaultwarden`): Traefik runs as a service within the stack on a local `proxy` bridge network. TLS is provided via HashiCorp Vault PKI ACME integration:

```yaml
command:
  - "--certificatesresolvers.vault.acme.caserver=${VAULT_ACME_ENDPOINT}"
  - "--certificatesresolvers.vault.acme.httpchallenge=true"
```

**Host-network daemon** (e.g. `scanopy`): The daemon service uses `network_mode: host` and `privileged: true`. Other services in the stack communicate over a dedicated internal bridge network.

Internal service communication uses a dedicated bridge network per stack (e.g. `scanopy`).

### Container Naming Convention

Containers use explicit `container_name` in the Compose file, following the pattern `<stack-name>-<service-name>`, e.g. `vaultwarden-docker-compose-vaultwarden`.

### Pre-commit Hook

`gitleaks` runs on every commit via `.pre-commit-config.yaml` to prevent accidental secret commits. Never bypass with `--no-verify`.
