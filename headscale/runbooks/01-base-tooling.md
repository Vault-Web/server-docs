# Step 1 - Base Tooling (Debian)

## Goal

Prepare a fresh Debian host for containerized Headscale deployment.

## Required packages/services

- `git`
- Docker Engine
- Docker Compose plugin

## Verify installation

```bash
git --version
docker --version
docker compose version
systemctl is-active docker
```

## Expected result

- Version output for `git`
- Version output for Docker and Docker Compose
- Docker service status: `active`

## Recorded reference output

```text
git version 2.47.3
Docker version 29.3.1, build c2be9cc
Docker Compose version v5.1.1
active
```
