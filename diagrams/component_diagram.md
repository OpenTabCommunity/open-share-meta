# Component Diagram — Open Share


This document contains an enhanced component diagram showing the internal modules of the client and server, their responsibilities, and runtime interactions. Each component includes implementation notes and suggested interfaces.

---

## Diagram 

```mermaid
flowchart TB
  subgraph Client[Device Client (Rust core)]
    UI[UI / UX Layer\n(choices: CLI/GUI/Tauri)]
    Disc[Discovery Module\n(mDNS + broadcast listener)]
    Pair[Pairing Module\n(QR, manual, TOFU)]
    KM[Key Management\n(Ed25519 identity, secure enclave)]
    Crypto[Crypto Suite\n(Ed25519/X25519/HKDF/AEAD)]
    Transport[Transport Layer\n(QUIC primary, TLS fallback)]
    Store[Storage\n(chunk cache, manifests)]
  end

  subgraph Server[Registration Server (Django)]
    Auth[Auth & Accounts]
    API[HTTP API: /device/register, /crl, /trustroot]
    DB[Database: device metadata, audit logs]
    Admin[Admin UI]
    Signer[KMS/HSM signing service]
  end

  UI --> Disc
  UI --> Pair
  Disc --> Crypto
  Pair --> KM
  KM --> Crypto
  Crypto --> Transport
  Transport --> Store
  API -->|issues cert_blob| Signer
  API --> DB
  Admin --> DB
  Signer --> API

  classDef client fill:#f8fafc,stroke:#0f172a
  classDef server fill:#f0f9ff,stroke:#0369a1
  class Client client
  class Server server
```

---

## Component responsibilities 

### UI / UX

* Minimal thin layer showing discovered devices, fingerprints, transfer progress and warnings.
* Must allow viewing the full Ed25519 fingerprint and selecting trust options (temporary vs permanent).

### Discovery Module

* Primary: mDNS TXT advertisement and listening.
* Fallback: UDP broadcast for networks where multicast is filtered.
* Exposes an event stream for the UI to subscribe to.
* Filters discovery results by `acct_hash` to avoid showing foreign accounts.

### Pairing Module

* Implements QR generation and scanning flows for one-time tokens.
* Produces a provisional trust state that learns the new device's pubkey and later reconciles with the server when online.

### Key Management

* Generates and persists Ed25519 private key.
* Uses OS keystore when available (Keychain, Windows CNG, Linux Keyring) or an embedded encrypted file store.
* Exposes APIs for signing ephemeral pubkeys during handshake.

### Crypto Suite

* Implements signature verification, ephemeral key generation (X25519), HKDF derivation and AEAD encryption/decryption.
* Provides a test harness for verifying cryptographic correctness.

### Transport Layer

* QUIC primary: use a mature Rust QUIC library (e.g., `quinn`) for streams, multipath and congestion benefits.
* TLS fallback: plain TLS-on-TCP for platforms where QUIC is unsupported.
* Implements per-chunk acknowledgement and flow control.

### Storage Layer

* Local block store for dedup and resume.
* Manifest store with signed manifests metadata for audit and transfer verification.
* Optionally encrypted on-disk storage if user enables an extra privacy mode.

### Server components

* **Auth & Accounts:** Standard Django auth; optional 2FA for administrative tasks.
* **API:** Lightweight DRF endpoints for `/device/register`, `/crl`, `/trustroot`.
* **DB:** Device table with `device_id`, `account_id`, `pubkey`, `cert_blob`, `revoked`, `last_seen`.
* **Admin:** Django admin for revocation and audit.
* **Signer (KMS/HSM):** Signing chain that produces server signatures for device certs — production traffic should not expose raw signing keys to app servers.

---

## Interfaces & message formats

* **Discovery TXT:** JSON-compressed short record: `{v:1, h:acct_hash, d:dev_id, p:port, f:short_fp}`
* **Device register (client->server):** POST JSON `{device_id, account_id, pubkey_ed25519, metadata}`
* **Cert_blob (server->client):** JSON+sig: `{device_id, account_id, pubkey, issued_at, expires_at, sig}`
* **Manifest (sender->receiver):** `{manifest_id, filename, size, chunk_hashes[], sender_sig}`
* **Chunk stream:** Encrypted AEAD frames with `chunk_id`, `seq`, `payload` and `mac`

---

## Implementation notes & best practices

* Keep interfaces tiny and well-typed; version all APIs (e.g., `/v1/device/register`).
* Use protobuf or CBOR for performance-sensitive wire formats, but keep the API JSON for management endpoints for ease of debugging.
* For the Rust client, expose a small FFI surface or IPC socket so GUI wrappers (Tauri / Electron) or mobile shims can integrate.

---

## Observability

* Client should emit structured logs for discovery events, handshake results and transfer traces.
* Server logs certificate issuance and revocation events with unique IDs for audit.
* Include distributed tracing IDs in manifest metadata for production debugging (redact user-sensitive data).

---
