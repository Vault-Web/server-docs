# Step 4 - DuckDNS + CNAME + Connectivity Validation

## Goal

Keep `vpn.example.com` stable while public server IP can change.

Approach:

- DuckDNS manages dynamic IP updates for `your-vpn.ddns-provider.example`
- Domain DNS keeps `vpn.example.com` via CNAME pointing to DDNS host

## A) DDNS setup status

Expected DDNS behavior:

- Message like `ip address ... was already <IP> not updated` is normal.
- It means record already points to current public IP.

## B) Configure automatic DDNS updates on server

Create update script:

```bash
sudo mkdir -p /opt/duckdns
sudo tee /opt/duckdns/update.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
DOMAIN="REPLACE_WITH_DUCKDNS_SUBDOMAIN"
TOKEN="REPLACE_WITH_DUCKDNS_TOKEN"
curl -fsS "https://www.duckdns.org/update?domains=${DOMAIN}&token=${TOKEN}&ip="
EOF
sudo chmod 700 /opt/duckdns/update.sh
sudo /opt/duckdns/update.sh
```

Create systemd service/timer:

```bash
sudo tee /etc/systemd/system/duckdns.service >/dev/null <<'EOF'
[Unit]
Description=Update DuckDNS record

[Service]
Type=oneshot
ExecStart=/opt/duckdns/update.sh
EOF

sudo tee /etc/systemd/system/duckdns.timer >/dev/null <<'EOF'
[Unit]
Description=Run DuckDNS update every 5 minutes

[Timer]
OnBootSec=30s
OnUnitActiveSec=5m
Unit=duckdns.service

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now duckdns.timer
sudo systemctl status duckdns.timer --no-pager
```

Validation:

```bash
systemctl list-timers --all | grep duckdns
journalctl -u duckdns.service -n 20 --no-pager
```

## C) DNS CNAME

Create DNS record at your DNS provider:

- Host: `vpn`
- Type: `CNAME`
- Target: `your-vpn.ddns-provider.example`
- TTL: default or low (for example 300)

Optional FQDN target form:

- `your-vpn.ddns-provider.example.`

## D) Keep Headscale URL on your own domain

On server:

```bash
cd /opt/headscale
sed -i 's|^HEADSCALE_DOMAIN=.*|HEADSCALE_DOMAIN=vpn.example.com|' .env
sed -i 's|^server_url:.*|server_url: https://vpn.example.com|' config/config.yaml
docker compose down
docker compose up -d
docker compose ps
docker compose logs --tail=100 caddy
docker compose logs --tail=100 headscale
```

## E) DNS propagation and reachability checks

From laptop/external machine:

```bash
dig +short vpn.example.com
dig +short your-vpn.ddns-provider.example
curl -vk https://vpn.example.com
```

Expected:

- Both `dig` results converge to same public IP
- `curl` reaches Caddy TLS endpoint (no connect timeout)

If `curl` hangs at connect:

- Check server listens on `443`:
  ```bash
  sudo ss -tulpen | grep ':443'
  ```
- Check containers:
  ```bash
  cd /opt/headscale && docker compose ps
  ```
- Check host firewall:
  ```bash
  sudo ufw status verbose
  ```
- Then verify upstream network:
  - router port forwarding `80/443` to server (home/NAT)
  - provider firewall/security group allows inbound `80/443` (VPS/cloud)

## F) Security actions (mandatory when secrets were shared)

- Rotate DuckDNS token if it was posted/shared.
- Rotate Headscale pre-auth keys if they were posted/shared.
- Store secrets only in password manager, never in public screenshots/logs.
