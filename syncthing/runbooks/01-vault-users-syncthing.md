# Step S1 - Syncthing for Vault Users (Server + Clients)

## Goal

Provide VPN-only file synchronization for Vault users so users can edit files directly in the local file explorer while data remains on the server.

How this complements Vault Web Cloud Page:

- Cloud Page reads and writes server-side files under `/data/vault-users/...`
- Syncthing synchronizes those same files to user devices
- No extra cloud storage service is needed

## Final Architecture

Use one Syncthing container per Vault user on the server:

- `syncthing-user-a` -> `/data/vault-users/user-a` -> GUI `100.64.0.10:8100`
- `syncthing-user-b` -> `/data/vault-users/user-b` -> GUI `100.64.0.10:8101`

Reason:

- A single Syncthing instance is not suitable for strict tenant separation
- Per-user containers give file-level isolation by mount scope

## Prerequisites

- Headscale/Tailscale already working
- Server tailnet IP known (examples in this runbook use `100.64.0.10`)
- User directories exist on server (`/data/vault-users/<user>`)

## 1) Remove old single Syncthing setup (if present)

Why:

- prevents accidental use of outdated ports/config
- avoids mixing single-user and multi-user topologies

```bash
sudo -i
set -euo pipefail

if [ -f /opt/syncthing/docker-compose.yml ]; then
  cd /opt/syncthing
  docker compose down --remove-orphans || true
fi

docker rm -f syncthing 2>/dev/null || true
```

## 2) Create multi-user Syncthing stack on server

Why:

- enforces user isolation by container mount scope
- keeps port layout deterministic (`8100`, `8101`, ...)

```bash
sudo -i
set -euo pipefail

TAILNET_IP="100.64.0.10"

mkdir -p /opt/syncthing-multi
mkdir -p /data/syncthing/users/user-a/config
mkdir -p /data/syncthing/users/user-b/config
mkdir -p /data/vault-users/user-a
mkdir -p /data/vault-users/user-b

cat >/opt/syncthing-multi/docker-compose.yml <<EOF
services:
  syncthing-user-a:
    image: lscr.io/linuxserver/syncthing:latest
    container_name: syncthing-user-a
    restart: unless-stopped
    environment:
      - PUID=0
      - PGID=0
      - TZ=Europe/Berlin
    volumes:
      - /data/syncthing/users/user-a/config:/config
      - /data/vault-users/user-a:/vault-user
    ports:
      - "${TAILNET_IP}:8100:8384/tcp"
      - "${TAILNET_IP}:22100:22000/tcp"
      - "${TAILNET_IP}:22100:22000/udp"
      - "${TAILNET_IP}:21100:21027/udp"

  syncthing-user-b:
    image: lscr.io/linuxserver/syncthing:latest
    container_name: syncthing-user-b
    restart: unless-stopped
    environment:
      - PUID=0
      - PGID=0
      - TZ=Europe/Berlin
    volumes:
      - /data/syncthing/users/user-b/config:/config
      - /data/vault-users/user-b:/vault-user
    ports:
      - "${TAILNET_IP}:8101:8384/tcp"
      - "${TAILNET_IP}:22101:22000/tcp"
      - "${TAILNET_IP}:22101:22000/udp"
      - "${TAILNET_IP}:21101:21027/udp"
EOF

cd /opt/syncthing-multi
docker compose up -d
docker compose ps
docker compose logs --tail=120
```

## 3) Secure each server GUI instance

Why:

- default open GUI is a critical security risk
- GUI credentials are mandatory before user onboarding

Open in VPN browser:

- User A: `http://100.64.0.10:8100`
- User B: `http://100.64.0.10:8101`

For each:

1. `Actions -> Settings -> GUI`
2. Set GUI auth user and password
3. Save and restart

## 4) Install and persist Syncthing on Linux client

Why:

- client needs its own local Syncthing node and Device ID
- linger ensures sync service survives reboot/logout

Run on each Linux client:

```bash
sudo apt-get update
sudo apt-get install -y syncthing
systemctl --user enable --now syncthing
loginctl enable-linger "$USER"
systemctl --user status syncthing --no-pager
```

Local client UI:

- `http://127.0.0.1:8384`

## 5) Pair server user-instance with client instance

Why:

- pairing establishes trust and encrypted sync channel
- both sides must explicitly accept to prevent rogue pairing

Per user:

1. On client local UI: `Actions -> Show ID` (copy client Device ID)
2. On server UI (`8100` or `8101`): `Add Remote Device`
3. Paste client Device ID and save
4. On client UI: accept incoming device request

Important:

- Server UI and client local UI are different Syncthing nodes
- Device IDs must be different

## 6) Correct folder mapping (critical)

Why:

- container sees mounted path `/vault-user`, not host path
- wrong path causes empty/out-of-sync folders despite successful scans

In server container UI, folder path must be:

- `/vault-user`

Do not set `/data/vault-users/<user>` inside container UI.

Recommended:

- Folder ID: user name (for example `user-a`)
- Server path: `/vault-user`
- Client path: local user folder (for example `~/Syncthing/user-a`)

Path consistency matrix:

- Host filesystem for Vault user files: `/data/vault-users/<user>`
- Cloud Page backend container path for same data: `/host-cloud/<user>`
- Syncthing user container path for same data: `/vault-user`

## 7) Recommended VPN-only connection settings

Why:

- disables public internet fallback paths
- keeps synchronization traffic inside tailnet

In each Syncthing instance:

- Disable Global Discovery
- Disable Relaying
- Disable NAT traversal (optional)

Since both peers are inside tailnet, direct VPN connectivity is enough.

## 8) Verification

Why:

- verifies process state, listening ports, and actual file propagation
- catches mount/path issues early

Server:

```bash
docker compose -f /opt/syncthing-multi/docker-compose.yml ps
ss -ltnup | egrep '8100|8101|22100|22101|21100|21101'
docker logs --tail=80 syncthing-user-a
docker logs --tail=80 syncthing-user-b
```

Expected:

- Devices show connected in UI
- Folder status is `Up to Date`
- Test file from client appears in server folder:
  - `/data/vault-users/user-a` or `/data/vault-users/user-b`

## 9) Troubleshooting

- `A device with that ID is already added`:
  - device already exists; check and reuse existing remote entry

- No remote device visible:
  - verify correct UIs (`8100/8101` on server, `127.0.0.1:8384` on client)

- Folder seems empty but says `Up to Date`:
  - wrong path; fix server path to `/vault-user`

- Forgot server GUI password:
  - stop user container, back up and reset `/data/syncthing/users/<user>/config`, start again

- Created `/data/vault-users/...` accidentally in container UI:
  - correct folder path to `/vault-user`
  - remove accidental path in container if needed

## 10) Reboot persistence

Why:

- ensures services recover automatically after host restart

Server:

- `restart: unless-stopped` for Syncthing containers
- Docker service enabled:

```bash
sudo systemctl enable --now docker
```

Client:

- user service enabled and linger enabled:
  - `systemctl --user enable syncthing`
  - `loginctl enable-linger "$USER"`
