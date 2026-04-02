# Server Documentation Index

> Security note: all domains, IPs, usernames, UUIDs, and tokens shown in this documentation are examples and must be replaced in production.

## Scope

This documentation covers the production deployment path for:

- Headscale control plane and TLS reverse proxy
- VPN-only Vault Web access
- Cloud Page filesystem mapping
- Daily backups of Vault user data
- Syncthing user synchronization

## Recommended Execution Order

1. [Headscale Concept 1 - Why Headscale](./headscale/concepts/01-why-headscale.md)
2. [Headscale Step 1 - Base Tooling](./headscale/runbooks/01-base-tooling.md)
3. [Headscale Step 2 - Headscale + Caddy Setup](./headscale/runbooks/02-headscale-caddy-setup.md)
4. [Headscale Step 4 - DDNS + CNAME + Connectivity](./headscale/runbooks/04-ddns-cname-connectivity.md)
5. [Headscale Step 3 - Device Onboarding](./headscale/runbooks/03-device-onboarding.md)
6. [Headscale Step 5 - Vault Web VPN Deployment](./headscale/runbooks/05-vault-web-vpn-deployment.md)
7. [Syncthing Concept 1 - Why Syncthing](./syncthing/concepts/01-why-syncthing.md)
8. [Syncthing Step S1 - Vault Users Sync](./syncthing/runbooks/01-vault-users-syncthing.md)

## Key References

- Main deploy guide:
  - [deploy/README.md](../../README.md)
- Headscale overview:
  - [headscale/README.md](./headscale/README.md)
- Syncthing overview:
  - [syncthing/README.md](./syncthing/README.md)
- Vault Web service docs:
  - [vault-web/README.md](./vault-web/README.md)

## Critical Operational Notes

- `vault.example.com` must resolve only inside VPN (Split DNS).
- Caddy must be connected to `deploy_default` network after recreate.
- Cloud Page user root paths in DB must use container path prefix `/host-cloud/...`.
- Backups are defined on server as daily incremental snapshots and should be tested regularly.

## Post-Reboot Health Check

Use this quick validation block after server restart or maintenance:

```bash
sudo -i
set -euo pipefail

systemctl is-active docker
systemctl is-active tailscaled
systemctl is-active dnsmasq || true
systemctl status backup-vault-users.timer --no-pager || true

docker compose -f /opt/deploy/docker-compose.deploy.yml ps
docker compose -f /opt/headscale/docker-compose.yml ps
docker compose -f /opt/syncthing-multi/docker-compose.yml ps || true

ss -tulpen | grep -E ':80|:443|:8080|:8081|:5433|:8100|:8101' || true
```
