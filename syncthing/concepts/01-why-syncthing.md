# Why Syncthing In This Project

## Purpose

This document explains what Syncthing is, how it works, and why it is used together with Headscale and Vault Web in this project.

## 1) Why Syncthing

The core problem is file synchronization between user devices and the home server without relying on third-party cloud storage.

Traditional cloud sync platforms usually imply:

- data hosted on external infrastructure
- account and provider dependency
- additional cost and policy risk

Syncthing solves this with direct, encrypted, peer-to-peer synchronization.

## 2) What Syncthing does differently

Syncthing is not a centralized storage service.  
Each participating device runs its own Syncthing node.

Consequences:

- no central file host is required
- devices exchange data directly whenever possible
- each side can verify integrity and sync state independently

## 3) How trust and connectivity work

Each node has its own cryptographic identity (Device ID derived from key material).

Pairing model:

1. Device A adds Device ID of Device B
2. Device B confirms Device A
3. Synchronization starts only after mutual trust is established

Connectivity model:

- With public internet only: discovery + relay may be used
- With VPN (Headscale/Tailscale): direct private connectivity is preferred

## 4) Why combine Syncthing with Headscale VPN

In this project, Syncthing runs on top of the private tailnet.

Benefits:

- stable private addressing
- reduced exposure to public discovery/relay paths
- simpler policy control and troubleshooting

Even though Syncthing already encrypts transport, VPN overlay improves operational control and privacy boundaries.

## 5) How Syncthing complements Vault Web Cloud Page

Cloud Page and Syncthing are complementary:

- Cloud Page: web UI for server-side user files
- Syncthing: direct device file synchronization for local editing

This allows users to edit files in local file explorers while keeping server-side storage canonical.

Path model in this project:

- host storage: `/data/vault-users/<user>`
- Cloud Page container view: `/host-cloud/<user>`
- Syncthing container view: `/vault-user`

## 6) Security and multi-user isolation

A single Syncthing instance is not used for all users in this setup.  
Instead, one container per user is used, each with exactly one mounted user folder.

Why:

- clearer separation of access scope
- less risk of accidental cross-user folder exposure

## 7) Where to continue

After this concept, follow:

- [Syncthing Runbook S1 - Vault Users](../runbooks/01-vault-users-syncthing.md)

