# Step 2 - Headscale + Caddy Setup

## Goal

Deploy Headscale behind Caddy with automatic TLS.

Example placeholders in this doc:

- Domain: `vpn.example.com`
- Email: `admin@example.com`

Replace with real values on production systems.

## 1) Create directories

```bash
mkdir -p /opt/headscale/{config,data,caddy}
cd /opt/headscale
```

## 2) Create `.env`

```bash
cat > /opt/headscale/.env <<'EOF'
HEADSCALE_DOMAIN=vpn.example.com
LETSENCRYPT_EMAIL=admin@example.com
EOF
```

## 3) Create `config/config.yaml`

```bash
cat > /opt/headscale/config/config.yaml <<'EOF'
server_url: https://vpn.example.com
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 0.0.0.0:9090

noise:
  private_key_path: /var/lib/headscale/noise_private.key

prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48

derp:
  server:
    enabled: false
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  paths: []
  auto_update_enabled: true
  update_frequency: 24h

disable_check_updates: true
ephemeral_node_inactivity_timeout: 30m

database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite
    write_ahead_log: true

policy:
  mode: file
  path: /etc/headscale/acl.hujson

dns:
  magic_dns: true
  base_domain: vpn.internal
  override_local_dns: false
  nameservers:
    global:
      - 1.1.1.1
      - 1.0.0.1
EOF
```

## 4) Create open ACL for initial setup

```bash
cat > /opt/headscale/config/acl.hujson <<'EOF'
{
  "acls": [
    { "action": "accept", "src": ["*"], "dst": ["*:*"] }
  ]
}
EOF
```

## 5) Create `Caddyfile`

```bash
cat > /opt/headscale/Caddyfile <<'EOF'
{
  email {$LETSENCRYPT_EMAIL}
}

{$HEADSCALE_DOMAIN} {
  reverse_proxy headscale:8080
}
EOF
```

Do not add Vault Web routing in this step.
Vault routing is configured in Step 5 after deploy services are online and network connectivity is verified.

## 6) Create `docker-compose.yml`

```bash
cat > /opt/headscale/docker-compose.yml <<'EOF'
services:
  headscale:
    image: headscale/headscale:0.26.1
    container_name: headscale
    command: serve
    restart: unless-stopped
    volumes:
      - ./config:/etc/headscale
      - ./data:/var/lib/headscale
    expose:
      - "8080"
      - "9090"

  caddy:
    image: caddy:2.9
    container_name: headscale-caddy
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy/data:/data
      - ./caddy/config:/config
    extra_hosts:
      - "host.docker.internal:host-gateway"
EOF
```

## 7) Start and validate

```bash
cd /opt/headscale
docker compose down
docker compose up -d
docker compose ps
docker compose logs --tail=80 headscale
docker compose logs --tail=80 caddy
```

## Troubleshooting

- `address already in use :80`: stop conflicting webserver (for example apache2).
- `initial DERPMap is empty`: make sure `derp.urls` is configured.
- ACME/TLS errors: verify real domain + DNS + open ports `80` and `443`.
- Configure Vault route only in Step 5 (`reverse_proxy deploy-frontend-1:80`) and ensure Caddy joins `deploy_default`.
