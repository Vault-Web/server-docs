# Syncthing Documentation

## Purpose

This section documents how Syncthing is used in this project to synchronize Vault user files between server and client devices over VPN.

## Concepts (Read First)

- [Concept 1 - Why Syncthing In This Project](./concepts/01-why-syncthing.md)

## How Syncthing Fits Into Vault Web

- Vault Web Cloud Page stores user files on the server under `/data/vault-users/<username>`.
- Syncthing synchronizes those files to user devices for direct editing in local file explorers.
- Cloud Page does not require separate object storage in this setup; it uses server filesystem paths.

## Security Model

- Syncthing communication is end-to-end encrypted.
- Connectivity is limited to the Headscale/Tailscale network.
- User isolation on server is implemented by running one Syncthing container per user with a single mounted folder.

## Runbook

Use this runbook for full setup, onboarding, and operations:

- [Step S1 - Syncthing for Vault Users (Server + Clients)](./runbooks/01-vault-users-syncthing.md)

## Recommended Operations Scope

- Server:
  - container lifecycle and per-user instance isolation
  - port mapping and restart persistence
- Client:
  - local Syncthing service enablement and pairing
  - folder acceptance and local path mapping

## Common Integration Pitfalls

- Using host paths inside container UI (must use container path `/vault-user`)
- Pairing against the wrong Syncthing UI endpoint
- Missing GUI authentication on server instances
- Assuming one server Syncthing instance can enforce strict multi-user isolation
