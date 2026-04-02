# Headscale Documentation

## Purpose

This section documents the Headscale-based VPN control plane used in this project and its integration with Caddy and Vault Web.

## Concepts (Read First)

- [Concept 1 - Why Headscale In This Project](./concepts/01-why-headscale.md)

## Operational Scope

- Headscale server deployment
- Public TLS endpoint for `vpn.<domain>`
- Client onboarding (laptop and mobile)
- DDNS and CNAME stability
- VPN-only Vault Web access through Split DNS

## Runbooks

- [Step 1 - Base Tooling](./runbooks/01-base-tooling.md)
- [Step 2 - Headscale + Caddy Setup](./runbooks/02-headscale-caddy-setup.md)
- [Step 3 - Device Onboarding (Laptop + Android)](./runbooks/03-device-onboarding.md)
- [Step 4 - DDNS + CNAME + Connectivity Validation](./runbooks/04-ddns-cname-connectivity.md)
- [Step 5 - Vault Web Deployment (VPN-Only Access)](./runbooks/05-vault-web-vpn-deployment.md)

For Syncthing user sync (separate concern), use:

- [Syncthing Step S1 - Vault Users Sync](../syncthing/runbooks/01-vault-users-syncthing.md)

## Recommended Execution Order

0. Concept 1
1. Step 1
2. Step 2
3. Step 4
4. Step 3
5. Step 5

## Important Integration Notes

- `vpn.<domain>` remains publicly reachable for control plane access.
- `vault.<domain>` should resolve inside VPN only (Split DNS).
- After recreating Caddy, reconnect it to deploy network if needed:
  - `docker network connect deploy_default headscale-caddy`
- Validate with `dig`, `curl`, and container logs after each major change.
