# Syncthing — Secure Peer-to-Peer File Synchronization

## 1. Why Do We Need Syncthing?

Before diving into how Syncthing works, let’s look at the typical problem it solves:  
You want to keep files synchronized between your **laptop** and your **home server** — but without trusting a cloud provider like Dropbox, Google Drive, or iCloud.

Traditional cloud solutions:

- Store your data on **third-party servers**
- Require **user accounts and authentication** via the internet
- Offer **limited privacy** since your data passes through or lives on someone else’s infrastructure

Syncthing solves this by creating a **fully decentralized, encrypted, peer-to-peer synchronization network** — entirely under your control.

---

## 2. The Problem With Traditional File Sync

| Problem | Description |
|----------|-------------|
| ☁️ **Centralized storage** | Files are stored on third-party servers (Dropbox, Google Drive, etc.) |
| 🔓 **Limited privacy** | Providers can technically access your unencrypted data |
| 🚫 **Single point of failure** | If the provider goes offline, you lose access |
| 💸 **Cost** | Cloud storage is often subscription-based |
| 🌍 **No true peer-to-peer** | All data flows through central servers, not directly between your devices |

---

## 3. What Syncthing Does Differently

Syncthing is **not** a client–server application.  
It’s a **peer-to-peer (P2P)** synchronization tool.

That means:
- Every device runs the **same Syncthing process**
- Each device is **equal** — there is no master or central server
- Files are synchronized **directly between devices**, over encrypted connections

In other words, **your devices talk directly to each other**, without any middleman.

---

## 4. Syncthing Architecture Explained

### 4.1 Equal Peers — No Central Server

Syncthing doesn’t use a dedicated “server” process.  
Instead, **every device** (laptop, server, phone, etc.) runs **its own Syncthing instance** — each acting as both client and server.

| Device | Role |
|---------|------|
| 💻 **Laptop** | Runs Syncthing, syncs selected folders, can send and receive changes |
| 🏠 **Home Server** | Runs the same Syncthing binary, continuously online to ensure files stay in sync |

> Both are equal peers — the home server is not special, except that it’s usually always online.

---

### 4.2 How Devices Find and Trust Each Other

Each Syncthing device generates a **cryptographic identity**:

- A **private key** (kept secret)
- A **public key**, from which a unique **Device ID** is derived

To connect two devices:

1. You add the **Device ID** of your peer (e.g., your server) to your local Syncthing instance.
2. The peer adds **your Device ID** as well.
3. Both devices verify each other and establish a **mutual trust** relationship.

Once paired, devices can **discover each other automatically** and synchronize files securely.

---

### 4.3 How Connection Establishment Works

Depending on your setup, there are two ways devices can reach each other:

#### Case 1: Without VPN

- Syncthing uses **Global Discovery Servers** to find the public IP address of each device.
- It then attempts **direct peer-to-peer connections** via NAT traversal or **relay servers** (if direct connection fails).
- All communication remains **TLS-encrypted**, even over relays.

#### Case 2: With VPN (e.g., WireGuard + Headscale)

- Devices use their **private VPN IPs** (e.g., `100.64.0.2 ↔ 100.64.0.3`).
- Syncthing connects **directly through the private network**.
- No discovery servers or relays are needed.
- Traffic stays **entirely within your private, encrypted VPN**.

---

### 4.4 Folder Synchronization

After devices are paired:

1. You select which folders to share.  
   Example:  
   - Laptop: `/home/user/Documents`  
   - Home Server: `/srv/sync/Documents`
2. Syncthing scans these folders, splits files into small blocks, and computes **hashes**.
3. Only **changed blocks** are transferred between devices.
4. Each file block is encrypted and verified on both ends.

This results in **efficient, secure, bidirectional synchronization**.

---

### 4.5 Encryption and Security

Syncthing’s communication is **fully encrypted** using **TLS** and **asymmetric keys**:

| Layer | Protection |
|--------|-------------|
| 🔐 **Key Pair** | Each device has a public/private key for identification |
| 🔒 **TLS Encryption** | All data in transit is encrypted end-to-end |
| 🧠 **Block Hashing** | Ensures file integrity — no corrupted or tampered data |
| 🚫 **No central data storage** | Files exist only on your devices — not on third-party servers |

Even if you use public relay or discovery servers, they **never see your file content**, only minimal metadata.

---

## 5. Practical Example: Laptop ↔ Home Server

| Step | Description |
|------|--------------|
| 1️⃣ | Install Syncthing on both devices |
| 2️⃣ | Exchange Device IDs and connect |
| 3️⃣ | Share a folder (e.g., “Documents”) |
| 4️⃣ | Files are scanned, hashed, and synchronized |
| 5️⃣ | Changes on one device are automatically replicated to the other |

Diagram:

```

Laptop (Syncthing)
│ Device ID: ABC123
│ Folder: ~/Documents
│
│  🔐 Encrypted P2P Connection
│
Home Server (Syncthing)
│ Device ID: XYZ789
│ Folder: /srv/sync/Documents

```

> The home server simply acts as a reliable, always-online peer.  
> You can add more devices (phones, tablets, etc.) — all will sync via the same P2P mesh.

---

## 6. Why Combine Syncthing With a VPN (like Headscale + WireGuard)?

While Syncthing already encrypts all traffic, using a **VPN overlay** adds further advantages:

| Benefit | Description |
|----------|--------------|
| 🧱 **Private Network Layer** | Devices get stable private IPs for easier direct connections |
| 🌐 **No Public Discovery Needed** | Devices can find each other directly inside the VPN |
| 🔒 **Double Encryption** | VPN + Syncthing encryption |
| 🏡 **Full Control** | Traffic never leaves your own infrastructure |

In practice:
- You install Syncthing on each device (Laptop, Server)
- You connect them via Headscale-managed WireGuard VPN
- Syncthing syncs only through the **private VPN network**

This gives you **maximum privacy, resilience, and self-hosted control**.

---

## 7. Takeaway

> Syncthing turns all your devices into a **secure, decentralized file sync network**, with no central server, no subscriptions, and no third-party access.  
> Every file stays in your control — encrypted, private, and directly synchronized across your trusted devices.