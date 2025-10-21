# Headscale — Self-Hosted VPN Coordination Server

## 1. Why Do We Need a VPN?

Before we understand what **Headscale** does, it’s important to first see **how communication normally works without a VPN** — and why that’s a potential security risk.

---

### 1.1 Communication Without a VPN

Imagine you have two devices:

* 💻 **Laptop** — connected to a public Wi-Fi in a café
* 🏠 **Home Server** — running at your home or in a data center

If you try to connect to your server directly over the internet, here’s what happens:

1. Your laptop sends data packets to the **public IP address** of your home server.
2. Those packets travel through **multiple routers, ISPs, and gateways** across the internet.
3. Every one of those intermediaries can, in theory, **see, log, or manipulate** your data.

Even if certain applications (like HTTPS or SSH) encrypt some data, your **connection metadata** (e.g., IP addresses, ports, and server endpoints) remains visible.
To make your server accessible, you must usually open ports and forward them through your router — exposing your home network to the internet.

That means:

* Attackers can **scan your public IP** and find open ports.
* They can try **brute-force attacks** or exploit known vulnerabilities.
* You must **secure every single service manually**, which is error-prone and complex.

---

### 1.2 The Problems With Public Exposure

Hosting services on the public internet comes with several downsides:

| Problem                      | Description                                                                  |
| ---------------------------- | ---------------------------------------------------------------------------- |
| 🔓 **Attack Surface**        | Every open port increases the chance of being attacked.                      |
| 🧱 **Firewall Complexity**   | You must manually configure port forwarding and access rules.                |
| 🌍 **No Private Networking** | Devices on different networks can’t communicate securely or directly.        |
| 🕵️ **Metadata Leakage**     | Even if content is encrypted, IPs and endpoints are still visible to others. |

In short, your devices remain **isolated** unless you expose them publicly — which is both **insecure and inconvenient**.

---

## 2. VPNs: Creating Private, Encrypted Networks

A **VPN** allows devices to communicate securely as if they were on the same LAN, even across the internet.
It encrypts all traffic and hides the network from outsiders.

---

### 2.1 How WireGuard VPN Works (Laptop ↔ Home Server Example)

Let’s take our café laptop and home server scenario, using **classic WireGuard VPN only**:

#### Step 1: WireGuard Processes

| Device      | WireGuard Process Role                                                                                                                                                                                 |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Laptop      | - Initiates connection to the home server<br>- Maintains the encrypted tunnel<br>- Sends and receives encrypted packets<br>- Uses its private key to authenticate                                      |
| Home Server | - Listens for incoming WireGuard connections<br>- Maintains server-side end of the tunnel<br>- Uses its private key to authenticate clients<br>- Decrypts and encrypts all traffic for connected peers |

#### Step 2: Key Exchange & Authentication

1. Each device has:

   * **Private Key** (kept secret)
   * **Public Key** (shared with peers)
2. Devices exchange public keys (manually or via config)
3. Each device computes a **shared symmetric session key**
4. All traffic is now encrypted end-to-end

#### Step 3: IP Assignment and Routing

WireGuard assigns **private VPN IP addresses** to each device.

* These IPs are **not public** and only exist within the VPN.
* They can be in any RFC1918 private range (e.g., `10.0.0.0/24`, `192.168.50.0/24`) or in a custom range like `100.64.0.0/10` (commonly used in WireGuard mesh VPNs).
* The IPs allow devices to address each other **like on a local network**.

**Example Configuration:**

```
Laptop:    10.0.0.5
HomeServer: 10.0.0.1
```

* Laptop sends packets to `10.0.0.1`.
* WireGuard encapsulates and encrypts packets, then routes them over the internet to the server’s public IP.
* Server WireGuard decapsulates and decrypts, delivering the packet internally.

This **creates a virtual private LAN** without exposing ports publicly.

---

#### Step 4: VPN Tunnel

1. Laptop authenticates using asymmetric keys
2. Tunnel is established between WireGuard processes
3. Laptop can reach the home server via its VPN IP (e.g., `10.0.0.1`)
4. WireGuard handles:

   * **Encapsulation of packets**
   * **Encryption/decryption**
   * **Routing within VPN**
5. All traffic is **peer-to-peer** and fully encrypted

**Diagram:**

```
Laptop (WireGuard)
  │ Private IP: 10.0.0.5
  │
  │  Encrypted VPN tunnel (over internet)
  │
Home Server (WireGuard)
  │ Private IP: 10.0.0.1
```

> No public ports on the server need to be exposed for VPN communication — only the WireGuard UDP port is open.
> VPN IPs are fully private and only visible within the VPN.

---

### 2.2 Security Foundations

WireGuard uses **asymmetric cryptography**:

* **Private Key**: kept secret on each device
* **Public Key**: shared with peers
* Shared symmetric session key for encryption
* All traffic is authenticated and encrypted end-to-end

Even on public Wi-Fi, **no one can read or tamper with traffic**.

---

## 3. Coordinating Devices: Tailscale and Headscale

So far, we have seen how **classic WireGuard VPN** works: devices exchange keys manually and establish a secure tunnel.  
But there are some limitations with this approach:

---

### 3.1 Limitations Without a Coordination Layer

Without a coordination server:

* Every device must be **configured manually** with the public key and VPN IP of every other device.
* Adding new devices is tedious — you must update all configs.
* NAT traversal or firewalls can prevent direct connections.
* There is no central management of users or devices.

This becomes cumbersome if you have **multiple devices** at home or want a mesh network where all devices communicate peer-to-peer.

---

### 3.2 Tailscale: Automatic Device Discovery

[Tailscale](https://tailscale.com) is a modern VPN solution built on top of WireGuard that **automates key exchange and discovery**:

* Devices register with a **central coordination server** (run by Tailscale Inc.)
* Public keys, VPN IPs, and device metadata are automatically distributed
* NAT traversal is handled automatically
* Devices can connect **peer-to-peer**, but new devices are easy to add

**Key point:** Tailscale simplifies management, but relies on a **third-party cloud server** to coordinate devices.  
Your traffic remains end-to-end encrypted via WireGuard, but the metadata (device names, IPs, keys) is stored by Tailscale.

---

### 3.3 Headscale: Self-Hosted Tailscale Alternative

[Headscale](https://github.com/juanfont/headscale) is an **open-source, self-hosted alternative** to the Tailscale coordination server:

* Implements the **same protocol as Tailscale**  
* Allows the use of **official Tailscale clients** on all devices  
* Runs on **your own server** — you control all keys, IPs, and device metadata  
* Handles **automatic key distribution, device discovery, and NAT traversal**  

**In other words:**  
> Headscale = Tailscale without the cloud.

---

### 3.4 How Headscale Complements WireGuard

WireGuard alone:

* Provides **encrypted tunnels**  
* Requires **manual configuration** of keys and IPs  
* No device discovery or coordination

WireGuard + Headscale:

* Headscale acts as a **control plane**  
* Devices still use WireGuard for **all encryption and data transfer**  
* Headscale only **distributes keys, assigns private VPN IPs, and helps peers find each other**  
* New devices can join easily without manually editing configs

**Installation Requirements:**

* **All devices** still need **WireGuard** installed (it handles encryption/tunnels)  
* Headscale runs **on a single server** for coordination  
* Devices do **not need a full Tailscale server**, only the Tailscale client (which uses WireGuard)  

---

### 3.5 Summary

| Feature | WireGuard Only | WireGuard + Headscale |
|---------|----------------|---------------------|
| Encrypted VPN tunnel | ✅ | ✅ |
| Private IP assignment | Manual | Automatic |
| Device discovery | Manual | Automatic |
| Adding new devices | Manual | Easy via Headscale |
| NAT traversal | Manual | Automatic |
| Third-party cloud dependency | ❌ | ❌ (Headscale is self-hosted) |

> Headscale makes managing a **multi-device WireGuard VPN** simple, automated, and fully private — perfect for home labs and self-hosted networks.