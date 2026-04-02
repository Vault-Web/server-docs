# Step 5 - Vault Web Deployment (HTTPS + VPN-Only Access)

## Goal

Deploy Vault Web with HTTPS on `vault.example.com`, but allow content access only for clients inside the Headscale/Tailscale tailnet.

## Final Idea (Important)

This setup intentionally keeps:

- `vpn.example.com` publicly reachable on `443` (required for Tailscale/Headscale control-plane).
- `vault.example.com` without public DNS record.
- `vault.example.com` resolves only inside VPN via Split-DNS to server tailnet IP (for example `100.64.0.10`).

So HTTPS stays clean and Vault is reachable only in VPN, without per-device hosts-file hacks.

## A) Prepare deploy stack

Why:

- initializes deployment baseline and environment variables
- allows controlled, repeatable startup

```bash
cd /opt
git clone --recurse-submodules https://github.com/Vault-Web/deploy.git
cd /opt/deploy
cp -n .env.example .env
```

Set/verify in `/opt/deploy/.env`:

```env
FRONTEND_PORT=127.0.0.1:8080
```

Notes:

- Frontend stays local-only on host.
- Public traffic must go through Caddy.

## B) Start deploy services

Why:

- brings up application services before exposing traffic routes
- avoids reverse proxy pointing to unavailable targets

```bash
cd /opt/deploy
docker compose -f docker-compose.deploy.yml up -d --build
docker compose -f docker-compose.deploy.yml ps
```

Cloud Page startup validation:

```bash
cd /opt/deploy
docker compose -f docker-compose.deploy.yml ps cloud-page-backend frontend
docker compose -f docker-compose.deploy.yml logs --tail=80 cloud-page-backend
```

Expected:

- `cloud-page-backend` is running
- no startup exception about datasource or missing root path mount

## C) Connect Headscale Caddy to deploy network

Why:

- Headscale and deploy stacks are separate Docker projects
- Caddy needs deploy network membership for name resolution (for example `deploy-frontend-1`)

`headscale-caddy` and deploy services are different compose projects, so Caddy must join deploy network:

```bash
cd /opt/headscale
docker network connect deploy_default headscale-caddy 2>/dev/null || true
```

## D) Configure Caddy for Vault frontend

Why:

- centralizes TLS termination and domain routing
- keeps internal services private (HTTP only behind proxy)

Edit `/opt/headscale/Caddyfile`:

```caddy
{
  email {$LETSENCRYPT_EMAIL}
}

vpn.example.com {
  reverse_proxy headscale:8080
}

vault.example.com {
  reverse_proxy deploy-frontend-1:80
}
```

Apply:

```bash
cd /opt/headscale
docker compose up -d --force-recreate caddy
docker network connect deploy_default headscale-caddy 2>/dev/null || true
docker compose logs --tail=200 caddy
```

Important:

- If Caddy is recreated again, run `docker network connect deploy_default headscale-caddy` again.
- Without this, Caddy cannot resolve deploy service containers.

## E) Configure Split-DNS on Headscale server

Why:

- makes `vault.<domain>` resolve to tailnet IP only for VPN clients
- prevents public DNS resolution and reduces attack surface

Install and configure `dnsmasq` (server side):

```bash
sudo apt-get update
sudo apt-get install -y dnsmasq

sudo tee /etc/dnsmasq.d/10-headscale-splitdns.conf >/dev/null <<'EOF'
bind-dynamic
interface=tailscale0
listen-address=127.0.0.1,100.64.0.10
no-resolv
address=/vault.example.com/100.64.0.10
server=1.1.1.1
server=1.0.0.1
cache-size=10000
EOF

sudo dnsmasq --test
sudo systemctl restart dnsmasq
ss -lupn | grep ':53'
dig +short vault.example.com @127.0.0.1
dig +short vault.example.com @100.64.0.10
```

Set Headscale DNS block in `/opt/headscale/config/config.yaml`:

```yaml
dns:
  magic_dns: true
  base_domain: vpn.internal
  override_local_dns: true
  nameservers:
    global:
      - 1.1.1.1
      - 1.0.0.1
    split:
      example.com:
        - 100.64.0.10
  extra_records:
    - name: vault.example.com
      type: A
      value: "100.64.0.10"
```

Apply:

```bash
cd /opt/headscale
docker compose restart headscale
```
## F) Firewall hardening (mandatory)

Why:

- blocks accidental direct access to internal/debug/database ports
- enforces edge-only access through approved reverse proxy paths

UFW baseline:

```bash
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status verbose
```

Block direct Docker-published admin/debug ports:

```bash
sudo iptables -F DOCKER-USER
sudo iptables -I DOCKER-USER -p tcp --dport 8080 -j DROP
sudo iptables -I DOCKER-USER -p tcp --dport 8081 -j DROP
sudo iptables -I DOCKER-USER -p tcp --dport 5433 -j DROP
sudo iptables -A DOCKER-USER -j RETURN
sudo iptables -S DOCKER-USER
```

## G) Join and verify clients in tailnet

Why:

- confirms endpoint policy from the client perspective
- validates DNS and routing after control-plane enrollment

On server (if not already joined):

```bash
tailscale status
tailscale ip -4
```

On laptop client:

```bash
sudo systemctl restart tailscaled
sudo tailscale up --reset --login-server=https://vpn.example.com --accept-dns=true
tailscale status
tailscale ip -4
dig +short vault.example.com
```

## H) Verification

Why:

- confirms TLS, DNS, routing, and application reachability are consistent
- confirms browser secure context required by cryptographic APIs

Server:

```bash
ss -tulpen | grep -E ':80|:443|:8080|:8081|:5433'
docker compose -f /opt/deploy/docker-compose.deploy.yml ps
docker compose -f /opt/headscale/docker-compose.yml ps
```

VPN client:

```bash
dig +short vault.example.com
curl -vk https://vpn.example.com
curl -vk https://vault.example.com
```

Public resolver check:

```bash
dig +short vault.example.com @1.1.1.1
```

Expected:

- VPN client resolves `vault.example.com` to `100.64.0.10` and gets `200`.
- Public resolver returns no `vault.example.com` record.

In browser console at `https://vault.example.com`:

- `window.isSecureContext` must be `true`
- `!!globalThis.crypto?.subtle` must be `true`

This is required for E2EE features.

## I) Daily backup of `/data/vault-users` (incremental snapshots)

### Why this setup

- First run creates a full backup snapshot.
- Later runs store only changed/new data (`--link-dest` hardlink snapshots).
- All snapshots look like full backups (easy restore).
- Retention keeps only last 5 snapshots.
- Timer runs daily at `23:30`.

### One-time setup

```bash
sudo -i
set -euo pipefail

DISK_DEV="/dev/sdc"
PART_UUID="<backup-partition-uuid>"
MNT="/mnt/backup5tb"

apt-get update
apt-get install -y rsync hdparm ntfs-3g

mkdir -p "${MNT}"
grep -q "${PART_UUID}" /etc/fstab || \
echo "UUID=${PART_UUID} ${MNT} ntfs-3g defaults,uid=0,gid=0,umask=022,nofail 0 0" >> /etc/fstab

mount "${MNT}" || true
```

Use `nofail` only; do not add `x-systemd.automount` if you want the disk mounted only during backup execution.

Create `/usr/local/sbin/backup-vault-users.sh`:

```bash
cat >/usr/local/sbin/backup-vault-users.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

SRC="/data/vault-users/"
MNT="/mnt/backup5tb"
DST_BASE="${MNT}/vault-users-backups"
TS="$(date +%F_%H-%M-%S)"
DST="${DST_BASE}/${TS}"
LATEST="${DST_BASE}/latest"
KEEP=5
DISK_DEV="/dev/sdc"

cleanup() {
  sync || true
  mountpoint -q "${MNT}" && umount "${MNT}" || true
  hdparm -y "${DISK_DEV}" >/dev/null 2>&1 || true
}
trap cleanup EXIT

mountpoint -q "${MNT}" || mount "${MNT}"
mkdir -p "${DST_BASE}"

if [ -L "${LATEST}" ] && [ -d "$(readlink -f "${LATEST}")" ]; then
  PREV="$(readlink -f "${LATEST}")"
  rsync -aH --delete --link-dest="${PREV}" "${SRC}" "${DST}/"
else
  rsync -aH --delete "${SRC}" "${DST}/"
fi

ln -sfn "${DST}" "${LATEST}"

find "${DST_BASE}" -mindepth 1 -maxdepth 1 -type d -printf '%P\n' \
  | sort -r | tail -n +$((KEEP+1)) | while read -r old; do
    rm -rf "${DST_BASE}/${old}"
  done
EOF

chmod +x /usr/local/sbin/backup-vault-users.sh
```

Create systemd units:

```bash
cat >/etc/systemd/system/backup-vault-users.service <<'EOF'
[Unit]
Description=Incremental backup of /data/vault-users to external disk

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/backup-vault-users.sh
EOF

cat >/etc/systemd/system/backup-vault-users.timer <<'EOF'
[Unit]
Description=Daily incremental backup timer for vault-users (23:30)

[Timer]
OnCalendar=*-*-* 23:30:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now backup-vault-users.timer
```

Immediate test run:

```bash
systemctl start backup-vault-users.service
systemctl status backup-vault-users.service --no-pager
journalctl -u backup-vault-users.service -n 80 --no-pager
mount /mnt/backup5tb || true
ls -lah /mnt/backup5tb/vault-users-backups
umount /mnt/backup5tb || true
```

`backup-vault-users.service` is a `oneshot` unit. A successful run ends with `inactive (dead)` and `status=0/SUCCESS`.

Manual restore:

```bash
mount /mnt/backup5tb || true
SNAP="/mnt/backup5tb/vault-users-backups/2026-04-01_23-30-00"
rsync -aH --delete "${SNAP}/" /data/vault-users/
umount /mnt/backup5tb || true
```

## J) Cloud Page user root assignment

Why:

- Cloud Page accesses host storage through container mount `/host-cloud`
- DB paths must reference container-visible path, not host path

`CLOUD_HOST_ROOT` is mounted as `/host-cloud` in container.
DB paths must use `/host-cloud/...`.

Correct:

```sql
UPDATE users
SET root_folder_path = '/host-cloud/alice'
WHERE username = 'alice';
```

Wrong:

- `/alice`
- `/data/vault-users/alice`

Validation:

```bash
ls -ld /data/vault-users /data/vault-users/alice
docker compose -f /opt/deploy/docker-compose.deploy.yml exec cloud-page-backend ls -ld /host-cloud /host-cloud/alice
```

Optional SQL update example directly in DB container:

```bash
docker exec -it vault-web-db psql -U POSTGRES_USER -d cloud_page_db -c \
"UPDATE users SET root_folder_path='/host-cloud/alice' WHERE username='alice';"
```

Replace `POSTGRES_USER` and database name with your actual values from `.env`.

Path consistency matrix:

- Host path (actual storage): `/data/vault-users/<user>`
- Cloud Page container path (DB must use this): `/host-cloud/<user>`
- Syncthing user container path: `/vault-user`

## K) Redeploy workflow

After updates:

```bash
cd /opt/deploy
git pull --ff-only
git submodule sync --recursive
git submodule update --init --recursive --remote
docker compose -f docker-compose.deploy.yml up -d --build --remove-orphans
```

## L) Troubleshooting

- `dial tcp: lookup frontend ... no such host` or `deploy-frontend-1 ... no such host` in caddy logs:
  - run `docker network connect deploy_default headscale-caddy` after Caddy recreate.

- `ERR_SSL_PROTOCOL_ERROR` for `vault.example.com`:
  - check Caddy logs and certificate issuance.
  - verify route exists in `/opt/headscale/Caddyfile`.

- VPN client resolves `vault.example.com` to public IP:
  - reconnect with `sudo tailscale up --reset --accept-dns=true --login-server=https://vpn.example.com`
  - flush local resolver cache.

- `backup-vault-users.service` fails with `rsync: command not found`:
  - install `rsync` package and rerun service.

- Backup mount unmount fails with `target is busy`:
  - inspect blockers: `lsof +f -- /mnt/backup5tb` and `fuser -vm /mnt/backup5tb`

- `Invalid CORS request`:
  - backend CORS rules do not include current frontend origin.

- `Root folder does not exist or is not a directory`:
  - DB path is wrong; use `/host-cloud/<user>`.

## M) Next step (optional): Syncthing user sync

After Vault Web VPN-only deployment is stable, continue with:

- [Syncthing Step S1 - Vault Users Sync](../../syncthing/runbooks/01-vault-users-syncthing.md)
