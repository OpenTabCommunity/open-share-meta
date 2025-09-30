
# Open Share

Secure, LAN-first, device-to-device file sharing with an optional registration server for device management.

> This README is a **standalone** source of truth for developers and product stakeholders. Detailed architecture diagrams, Mermaid flows and the Gantt roadmap are intentionally located in separate files for clean rendering: `ARCHITECTURE.md`, `ROADMAP.md`, and `docs/diagrams.md`.

---

## Table of contents

1. [What is Open Share?](#what-is-open-share)
2. [Goals & non-goals](#goals--non-goals)
3. [Tech stack](#tech-stack)
4. [Quick start — dev (minimal)](#quick-start--dev-minimal)
5. [High-level design & responsibilities](#high-level-design--responsibilities)
6. [Primary flows (summary)](#primary-flows-summary)
7. [API summary & device certificate format](#api-summary--device-certificate-format)
8. [Data model (summary)](#data-model-summary)
9. [Security checklist (operational)](#security-checklist-operational)
10. [Network test plan (summary)](#network-test-plan-summary)
11. [How to contribute & contacts](#how-to-contribute--contacts)

---

## What is Open Share?

Open Share is a privacy-first, LAN-first file sharing solution designed to move user files directly between devices on the same local network whenever possible. A lightweight registration server provides device management, signed device certificates and a Certificate Revocation List (CRL). The server **never** holds private keys or file payloads by design.

---

## Goals & non-goals
<a id="goals--non-goals"></a>

**Goals**

* Direct, encrypted device-to-device transfers on the LAN.
* A minimal management server for device cert issuance and revocation.
* Simple, auditable trust model (server-signed certs, CRL, explicit pairing).
* Resumable, integrity-checked transfers with dedup-friendly chunking.

**Non-goals (early)**

* Full cloud sync of user files.
* Complex server-side storage of file payloads.

---

## Tech stack

* **Client core:** Rust (core transfer, crypto, transport primitives).
* **Server:** Django (API, device management, CRL, auth). Django chosen for fast iteration, admin, and ecosystem.
* **Transport:** QUIC preferred (UDP-based); TLS/TCP fallback.
* **Crypto primitives:** Ed25519 (identity signatures), X25519 (ECDH), HKDF-SHA256 (KDF), ChaCha20-Poly1305 / AES-GCM (AEAD).

Notes: The client is designed to be portable — consider Tauri or a small GUI wrapper for cross-platform UI. Server uses Django so you get admin UI and batteries-included APIs fast.

---

## Quick start — dev (minimal)
<a id="quick-start--dev-minimal"></a>

This assumes you have Rust and Python/Django environment available.

### 1) Start the Django registration server (development)

```bash
# example: create venv, install requirements, run migrations
python -m venv .venv
source .venv/bin/activate
pip install -r server/requirements.txt
cd server
python manage.py migrate
python manage.py runserver 0.0.0.0:8443
```

This runs a dev-only server on `:8443`. Use `--insecure` options in client for local testing only.

### 2) Build or run Rust client in discovery mode

```bash
cd client
cargo run -- discover
```

### 3) Register a device (example curl)

```bash
curl -k -X POST "https://localhost:8443/device/register" \
  -H "Content-Type: application/json" \
  -d '{
    "device_id":"dev-local-01",
    "account_id":"acct-alice",
    "pubkey_ed25519":"BASE64_PUBKEY",
    "metadata":{"name":"alice-mbp","os":"macos"}
  }'
```

Response should include `cert_blob` and `trust_root`.

> **DEV NOTE:** `-k` / `--insecure` and self-signed TLS must be limited to local/dev. Use a valid cert in staging/prod.

---

## High-level design & responsibilities
<a id="high-level-design--responsibilities"></a>

(Short, human-readable summary — full diagrams & flows live in `ARCHITECTURE.md` and `docs/diagrams.md`.)

**Client responsibilities (Rust core)**

* Generate and protect Ed25519 identity keypair locally.
* Announce presence via mDNS/UDP (when available) or use QR/offline pairing fallback.
* Perform mutual authentication with server-signed device certs and CRL checks.
* Establish ephemeral X25519 handshake; derive session keys with HKDF and spawn AEAD channel for file chunks.
* Send/receive manifests and chunked streams; verify integrity and assemble files.

**Server responsibilities (Django)**

* Authenticated device registration endpoint that signs device public keys into compact certs.
* Provide `trustroot` (server signing public key) and CRL endpoints.
* Admin interface for device revocation and audit logs.

**Ops**

* Keep signing keys in KMS/HSM for production; rotate regularly.
* Monitor CRL publication and server health.

---

## Primary flows (summary)

For diagrammed, step-by-step flows see `docs/diagrams.md`.

1. **Registration:** Device generates Ed25519 locally → POST `/device/register` → server returns `cert_blob` signed by server key.
2. **Discovery:** mDNS advertise (TXT) with account hash, device id, port. If blocked, QR/manual pairing is used.
3. **Handshake:** Exchange certs → verify server signature & CRL → exchange signed ephemeral X25519 pubkeys → derive session key.
4. **Transfer:** Sender posts signed manifest (filenames, chunk hashes) → receiver requests missing chunks → sender streams encrypted chunks → receiver verifies hashes and sends final receipt.
5. **Offline pairing:** One-time QR token signed by a registered device to bootstrap trust when server or multicast is unreachable.
6. **Revocation:** Admin triggers `POST /device/revoke`; server updates CRL; devices fetch CRL and refuse revoked IDs.

---

## API summary & device certificate format
<a id="api-summary--device-certificate-format"></a>

Full API reference belongs in `docs/api.md` (create when server models stabilize). Below is a concise, practical summary.

**Core endpoints**

| Method |              Path | Purpose                                             |
| ------ | ----------------: | --------------------------------------------------- |
| POST   | /account/register | Create user account                                 |
| POST   |       /auth/login | Login & session (token/cookie)                      |
| POST   |  /device/register | Submit device pubkey & metadata → returns cert_blob |
| GET    |  /account/devices | List user's devices (public info)                   |
| POST   |    /device/revoke | Revoke a device (admin)                             |
| GET    |              /crl | Retrieve current CRL                                |
| GET    |        /trustroot | Get server signing public key                       |

**Device certificate (JSON compact example)**

```json
{
  "device_id": "dev-7a3b",
  "account_id": "acct-zz99",
  "pubkey_ed25519": "BASE64...",
  "issued_at": "2025-09-01T12:00:00Z",
  "expires_at": "2027-09-01T12:00:00Z",
  "sig": "BASE64_SIGNATURE_BY_SERVER"
}
```

* Verify `sig` using the `trustroot` before accepting a certificate.
* Certificates are intentionally concise; store the `cert_blob` exactly as returned.

---

## Data model (summary)

A short summary — detailed model and class diagrams live in `ARCHITECTURE.md`.

* **User:** `user_id`, `email_hash`, `pw_hash` (if applicable), `created_at`.
* **Device:** `device_id`, `account_id`, `pubkey_ed25519`, `cert_blob`, `revoked`, `last_seen`.
* **FileManifest:** `manifest_id`, `filename`, `size`, `chunk_hashes[]`, `sender_sig`.
* **Chunk (client-side cache):** `chunk_id`, `size`, `bytes`.
* **CRL:** `revoked_device_ids[]`, `issued_at`.

Storage constraints:

* Server: only public metadata and CRL.
* Client: manifests and chunk caches for resuming and dedup.

---

## Security checklist (operational)

This is an actionable checklist you should track in your CI/CD and release playbooks.

* [ ] Use Ed25519 for device identity signatures.
* [ ] Use X25519 for ephemeral ECDH.
* [ ] Use HKDF-SHA256 for key derivation.
* [ ] Use ChaCha20-Poly1305 or AES-GCM for AEAD.
* [ ] Store device private keys in secure enclave / OS-provided keystore when available.
* [ ] Keep server signing keys in KMS/HSM; limit access.
* [ ] Issue short-lived certs and publish CRL; clients must poll CRL or use cache invalidation.
* [ ] Require explicit user confirmation for unknown device pairings in the UI.
* [ ] Log certificate issuance, revocation, and pairing events; retain logs per compliance requirements.

---

## Network test plan (summary)

Put concrete test results in `NETWORK_TEST_PLAN.md`.

Minimum test matrix:

* Multicast allowed vs blocked (home vs enterprise).
* Client isolation (guest Wi‑Fi).
* Android / iOS hotspot behavior.
* IPv6-only networks.
* NAT behaviour across common consumer routers.
* High-latency / lossy wireless.
* Cross-OS compatibility: Windows, macOS, Linux, Android, iOS.

Acceptance targets (examples):

* Discovery: device appears within 3s on local LAN when multicast is available.
* Handshake: channel established with fingerprint visible to user.
* Transfer: 100MB file completes integrity-verified; interrupted transfers resume.

---

## How to contribute & contacts
<a id="how-to-contribute--contacts"></a>

* Fork → feature branch → PR → review. Add tests for network and crypto flows.
* Issues: tag with `bug`, `proposal`, `spec`, `security`.
* Security issues: disclose privately to the maintainers (add maintainer email here).

**License:** Prefer `Apache-2.0` or `MIT` (decide as a team). Add `LICENSE` file at repo root.

**Maintainer:** Add name and email/contact point.

---
