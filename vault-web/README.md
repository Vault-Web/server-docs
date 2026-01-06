# Vault Web

## Introduction

Vault Web is the core of the Vault Web ecosystem and acts as a central, self-hosted portal for private services in home server environments. Instead of running every tool in isolation, Vault Web bundles the essential capabilities into one unified web interface and connects additional services through clear API boundaries.

This file provides a short positioning statement and feature overview. For full development and stack details, see the main repository:
https://github.com/Vault-Web/vault-web

## Core Features

- Central dashboard as the entry point for all integrated services
- User, session, and role management for daily home-server use
- Central authentication with JWT (single entry point, consistent access logic)
- Built-in communication tools like internal chats
- Frontend integration of external services (e.g., file management via Cloud Page)
- Modular structure: services stay independent and connect only via APIs

## Why This Matters

Vault Web is not meant to be a monolithic "everything-in-one" server, but a stable, extensible base. This keeps the ecosystem flexible and allows it to grow without tightly coupling or blocking existing services.
