# Vault Web - E2EE v1 (Basic ECDH)

This document explains the current E2EE v1 implementation (Basic ECDH + AES-GCM), step by step, for readers who are new to encryption.

---

## 1. What Is End-to-End Encryption?

**End-to-End Encryption (E2EE)** means:

- Messages are encrypted on the sender's device.
- Only the intended recipients can decrypt them.
- The server only stores and forwards **ciphertext**, never plaintext.

So even if the server is compromised, it cannot read the content.

---

## 2. What Is a "Device" in Vault Web?

In Vault Web, a **device** is any client instance, for example:

- Chrome on your laptop
- Firefox on your laptop
- A mobile browser
- An incognito window
- A different browser profile

Each of those is a separate **device** with its own keys.

Each device has:

- `deviceId` (a random UUID stored in localStorage)
- an **ECDH key pair** (public + private key)

---

## 3. What Is ECDH?

**ECDH** stands for **Elliptic Curve Diffie-Hellman**.

It is a way for two devices to calculate the **same shared secret** without sending that secret over the network.

Each device has:

- **Public Key** (shared)
- **Private Key** (never shared)

Both sides can compute:

- `sharedSecret = ECDH(myPrivateKey, otherPublicKey)`

The result is the same on both sides and can be used as a secret encryption key.

---

## 4. High-Level Flow (Private Chat)

### 4.1 Login

1. User logs in with username + password.
2. Backend returns a **JWT access token**.
3. Frontend stores the token in `localStorage`.

### 4.2 Device Registration

When a user opens a private chat:

1. The client checks if it already has a `deviceId` + key pair.
2. If not, it generates them.
3. It sends:
   - `deviceId`
   - `publicKey`
   - `Authorization: Bearer <token>`

to:

```
POST /api/devices/register
```

Backend saves the device for that user.

### 4.3 Device Discovery

Client asks the server for all devices in the chat:

```
GET /api/private-chats/devices?privateChatId=...
```

Server responds with device IDs + public keys of both participants.

### 4.4 Encrypting a Message

Before sending a message:

1. The sender loads the latest device list.
2. For **each recipient device**, the sender:
   - Computes ECDH shared secret
   - Derives an AES-GCM key (via HKDF)
   - Encrypts the message
3. This creates **one ciphertext per device**.

The payload looks like:

```
{
  senderDeviceId,
  senderPublicKey,
  recipients: {
    deviceIdA: { iv, salt, ciphertext },
    deviceIdB: { iv, salt, ciphertext }
  }
}
```

### 4.5 Sending

The sender sends the payload over WebSocket:

```
/app/chat.private.send
```

### 4.6 Server Storage

The server stores:

- `e2eePayload`
- sender metadata
- timestamps

The server **cannot** decrypt the message.

### 4.7 Receiving and Decrypting

When a recipient receives the payload:

1. It checks if there is an entry for its own `deviceId`.
2. If yes:
   - It computes ECDH shared secret
   - Derives AES key
   - Decrypts the message
3. If no:
   - It shows **"Encrypted message"** (fallback)

This fallback appears when a device was not yet registered at the time of encryption.

---

## 5. Group Chat E2EE (Current Approach)

Group chats use the same E2EE logic as private chats, but with **more devices**:

1. Sender loads **all devices of all group members**:
   ```
   GET /api/groups/{id}/devices
   ```
2. Sender encrypts **one ciphertext per device**.
3. Payload is sent over:
   ```
   /app/chat.send
   ```

This is simple and secure, but **scales linearly** with number of devices.

---

## 6. Why Multiple Devices Matter

If a user logs in on another browser or device:

- A **new deviceId** and **new key pair** are created.
- The sender must include that device in encryption.

That is why the sender **refreshes the device list before every send**.

---

## 7. Why "Encrypted message" Sometimes Appears

This happens when:

- The recipient device was not registered yet.
- The sender encrypted only for older devices.

Fix:

1. Open the chat on both devices (to register).
2. Reload the chat.
3. Send a new message.

---

## 8. Cryptography Details (Current Defaults)

This section documents the exact primitives used today so the implementation is reproducible.

- Key agreement: **ECDH** using the client key pair and the recipient public key
- Key derivation: **HKDF** to derive an AES-GCM key from the shared secret
- Message encryption: **AES-GCM** with a fresh `iv` per message
- Per-recipient encryption: **one ciphertext per device**

If any of these parameters change in the code (curve, HKDF salt/info, AES key length),
this document should be updated in the same change set.

---

## 9. Key Storage and Lifetime

- Each device generates and stores its own key pair locally.
- `deviceId` and the private key are stored client-side (currently `localStorage`).
- There is no automatic key rotation in v1.
- Reinstalling/clearing storage creates a new device identity.

---

## 10. Limitations and Threat Model (v1)

This version prioritizes simplicity and clarity. It does **not** yet include:

- Forward secrecy (no Double Ratchet)
- Asynchronous session setup (no X3DH)
- Signed prekeys or device identity signatures
- Verified device safety numbers / QR verification

If the server or a device is compromised **before** messages are sent, the attacker
may be able to impersonate a device. Later versions will address these gaps.
