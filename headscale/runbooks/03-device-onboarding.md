# Step 3 - Device Onboarding (Laptop + Android)

## Goal

Onboard a user into the Headscale network on:

- Linux laptop
- Android phone

Template usage:

- Replace `alice` with target username.
- Replace numeric user ID and keys with values from your server output.

## A) Server-side preparation

Why:

- creates explicit user ownership in Headscale
- pre-auth key allows controlled first enrollment

Run on server:

```bash
cd /opt/headscale
docker exec -it headscale headscale users create alice
docker exec -it headscale headscale users list
```

Create a reusable key:

```bash
docker exec -it headscale headscale preauthkeys create --user 1 --reusable --expiration 87600h
docker exec -it headscale headscale preauthkeys list --user 1
```

Notes:

- `--user` expects numeric user ID, not username.
- Replace `1` with Alice's actual user ID from `users list`.

## B) Alice private laptop

### 1) Install Tailscale (if missing)

Why:

- Tailscale client is the endpoint component for Headscale-managed tailnet access

```bash
tailscale version
```

If command not found:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### 2) Join with Headscale

Why:

- enrolls device into your self-hosted control plane
- `--accept-dns=true` applies VPN DNS policy centrally

```bash
sudo tailscale up --reset --login-server https://vpn.example.com --authkey REPLACE_WITH_PREAUTH_KEY --accept-dns=true
```

Notes:

- In this setup, pre-auth keys may appear as plain hex text (not always `tskey-*`).
- Use exactly the value returned by `headscale preauthkeys create`.
- If command hangs, verify control server reachability first:
  ```bash
  curl -vk https://vpn.example.com
  ```

### 3) Validate

Why:

- verifies that enrollment and tunnel setup completed successfully

```bash
tailscale status
tailscale ip -4
```

Expected:

- Status is connected (not logged out)
- IPv4 tailnet address like `100.x.y.z`

## C) Android device

Why:

- validates multi-device user access model (desktop + mobile)

1. Install Tailscale from Play Store.
2. Open app settings/login flow.
3. Set custom/login server to `https://vpn.example.com`.
4. Continue onboarding and enter the pre-auth key if prompted.
5. Confirm app shows connected state and tailnet IP.

## D) Optional manual registration flow (if app provides registration key)

Why:

- some client flows require explicit server-side approval

In some onboarding flows, the client app shows a registration key (for example `prx...`).
If the node appears as pending, register it manually:

```bash
docker exec -it headscale headscale nodes register --user alice --key REPLACE_WITH_REGISTRATION_KEY
```

Notes:

- Use Headscale username in `--user` (for example `alice`).
- This command is only required when node approval is not automatic.

## E) Verify from server

Why:

- confirms node ownership and online status from control-plane view

```bash
docker exec -it headscale headscale nodes list
```

Expected:

- Laptop and Android nodes are listed
- `User` column maps to target username
- `Connected` shows online for active devices

Common observation:

- Android or app-driven registrations can appear with generic hostnames (for example `localhost`).
- If needed, rename later for clarity (command availability depends on Headscale version).

## F) What to record

Why:

- preserves operational context for troubleshooting and audits

- Headscale URL in use
- user ID
- laptop hostname and tailnet IP
- Android hostname and tailnet IP
- key policy decision:
  - keep reusable key for future devices, or
  - rotate/delete key after onboarding

## G) Onboarding additional users

Why:

- provides repeatable, low-variance onboarding pattern

Repeat the same flow for each user:

1. Create user
2. Create pre-auth key
3. Join laptop and phone
4. Verify nodes list

Example command pattern:

```bash
docker exec -it headscale headscale users create USERNAME
docker exec -it headscale headscale users list
docker exec -it headscale headscale preauthkeys create --user USER_ID --reusable --expiration 87600h
```
