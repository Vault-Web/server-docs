# Why Headscale In This Project

## Purpose

This document explains what Headscale is, how it works with WireGuard/Tailscale clients, and why it is part of this project architecture.

## 1) Why a VPN is needed

Without a VPN, client devices reach services over public internet paths:

- traffic crosses multiple third-party networks
- metadata (IPs, ports, endpoints) is exposed
- public services increase attack surface

Even with HTTPS for application traffic, direct public exposure still requires strict firewall and service hardening on every endpoint.

## 2) What WireGuard gives you

WireGuard provides:

- encrypted peer-to-peer tunnels
- authenticated peers via key pairs
- private overlay IPs for device-to-device connectivity

WireGuard alone is strong cryptographically, but operationally manual:

- key distribution is manual
- peer inventory and updates are manual
- NAT traversal and multi-device rollout are harder at scale

## 3) What Headscale adds

Headscale is a self-hosted control plane compatible with official Tailscale clients.

Headscale handles:

- device registration
- key coordination
- IP allocation in tailnet
- peer discovery and connectivity metadata

Data plane remains WireGuard:

- payload traffic is still end-to-end encrypted between clients
- Headscale coordinates; it does not proxy your file/app payloads

## 4) Why this fits Vault Web

This project uses Headscale because it enables:

1. Private reachability for services (`vault.example.com`) only inside VPN
2. Split-DNS routing to tailnet IPs without per-device hosts hacks
3. Stable onboarding flow for laptop/mobile clients
4. Clean integration with Caddy for TLS termination and domain routing

In practice:

- `vpn.example.com` stays public for control-plane login
- `vault.example.com` resolves only in VPN
- app traffic stays private while user experience remains domain-based

## 5) Security model summary

- Headscale: coordination plane
- WireGuard (via Tailscale clients): encrypted transport plane
- Caddy: TLS edge and reverse proxy policy
- Firewall + Docker user-chain rules: port exposure control
- Split-DNS: private service name resolution

This combination provides private-by-default access with reproducible operations.

## 6) Where to continue

After reading this concept, continue with runbooks:

1. `runbooks/01-base-tooling.md`
2. `runbooks/02-headscale-caddy-setup.md`
3. `runbooks/04-ddns-cname-connectivity.md`
4. `runbooks/03-device-onboarding.md`
5. `runbooks/05-vault-web-vpn-deployment.md`
