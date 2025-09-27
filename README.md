# Server Documentation

This repository provides comprehensive documentation for setting up and managing 
a self-hosted server environment with Headscale and Vault-Web. The goal is to create a secure, private, and 
well-documented infrastructure that can be easily replicated and maintained.

## Purpose

The documentation serves as a central knowledge base, covering installation, 
configuration, and usage of the core components in the server stack. Each module 
is documented in both Markdown (`.md`) for quick reference and LaTeX (`.tex`) for 
formal export into PDF manuals.

## Structure

docs/  
├── [headscale/](/headscale/) # Documentation for the Headscale  coordination server  
├── [syncthing/](/syncthing/) # Documentation for Syncthing file synchronization  
└── [vault-web/](/vault-web/) # Documentation for Vault-based secret management

### Headscale
Documentation on installing, configuring, and operating a private coordination 
server compatible with Tailscale clients. Includes user/node management, 
configuration reference, and troubleshooting guides.

### Syncthing
Guides for setting up secure peer-to-peer file synchronization across devices, 
with integration through Headscale for private networking.

### Vault-Web
Covers private chats, web-based cloudpage and secret management, with a focus on integration into the 
self-hosted ecosystem. Explains setup, access control, and best practices for securely handling sensitive data.

---

Each subdirectory contains:
- A **Markdown file (`.md`)** for lightweight, GitHub-friendly documentation.
- A **LaTeX file (`.tex`)** for generating professional PDF documentation.


## Goals

- Provide a transparent and reproducible setup for self-hosted infrastructure.  
- Combine lightweight documentation (Markdown) with formal exports (LaTeX/PDF).