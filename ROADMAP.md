# ROADMAP.md

> The roadmap outlines iterative phases and deliverables for Open Share. Each phase has concrete outcomes, acceptance criteria, and suggested tests. Dates are placeholders — align them with your sprint cadence.

---

## Roadmap overview

The roadmap is intentionally iterative: small, demoable increments that each deliver user value and reduce risk.

Phases (high level):

* Phase 0 — Project setup & infra
* Phase A — Discovery POC (mDNS & broadcast)
* Phase B — Registration server & device certs (Django)
* Phase C — Handshake & secure channel (Rust client)
* Phase D — Single-file transfer & resume
* Phase E — Chunking, dedup & caching
* Phase F — Offline pairing & CRL propagation
* Phase G — QUIC transport, mobile packaging, performance tuning
* Phase H — Security hardening & production readiness

---

## Phase details, deliverables & acceptance

### Phase 0 — Project setup & infra

**Deliverables**

* Repo skeleton, CI scaffold, development environment docs.
* Basic linting, code formatting rules (rustfmt, black), and pre-commit hooks.

**Acceptance criteria**

* Devs can run server and client locally following README steps.
* CI runs unit tests on PRs.

**Risks / mitigations**

* Risk: environment differences — mitigate via containerized dev (docker-compose).

---

### Phase A — Discovery POC

**Deliverables**

* Rust client can broadcast and discover peers via mDNS and UDP broadcast.
* Simple CLI to list found devices.

**Acceptance criteria**

* Devices discover each other on a home LAN within 3s.
* Discovery works across at least two OSs in test matrix (Linux, macOS).

**Tests**

* Multicast allowed scenario
* Multicast blocked scenario with broadcast fallback

---

### Phase B — Registration server

**Deliverables**

* Django server with `/device/register`, `/trustroot`, `/crl` endpoints.
* Admin UI to view/register/revoke devices.

**Acceptance**

* Server issues `cert_blob` that clients verify using `trustroot`.
* Admin revocation adds device to CRL.

**Tests**

* Registration, cert verification, CRL fetch.

---

### Phase C — Handshake & mutual auth

**Deliverables**

* Rust client implements handshake: exchange certs, sign ephemeral pubkeys, derive session key.
* Unit tests for HKDF derivation and signature verification.

**Acceptance**

* Two clients can establish AEAD channel and exchange short messages securely.
* Malformed or revoked certs are rejected.

**Tests**

* Replay test for ephemeral pubkeys
* CRL-enforced rejection

---

### Phase D — Single-file transfer & resume

**Deliverables**

* Manifest format, chunk streaming and per-chunk ACK.
* Resume support and final signed receipt.

**Acceptance**

* 100MB file transfer completes with verified integrity.
* Interrupt transfer and resume successfully.

**Tests**

* Interrupt at different chunk boundaries, verify resume.

---

### Phase E — Chunking, dedup & caching

**Deliverables**

* Content-addressed chunk store with cache hits preserved across transfers.
* Dedup with manifest-level dedup checks.

**Acceptance**

* Re-sending same file shows cache hits and reduced transfer time.

**Tests**

* Send duplicate files and measure transfer reduction.

---

### Phase F — Offline pairing & CRL caching

**Deliverables**

* QR pairing flows and TOFU fallback documented.
* CRL caching and verification improvements.

**Acceptance**

* Devices can pair offline using QR and later register with server when online.
* Revoked device cannot handshake after CRL update is propagated.

**Tests**

* QR pairing roundtrip
* CRL propagation and enforcement test

---

### Phase G — QUIC & mobile packaging

**Deliverables**

* Integrate a QUIC library (e.g., quinn for Rust) and validate on mobile platforms.
* Packaging for Android and iOS (Tauri + mobile-specific shims or platform-native wrappers).

**Acceptance**

* QUIC connection is the default transport and shows improved latency and stability.
* Mobile build runs with acceptable battery/network behavior.

**Tests**

* QUIC vs TCP performance benchmarks
* Mobile background transfer tests

---

### Phase H — Security hardening & production readiness

**Deliverables**

* HSM/KMS integration, key rotation playbooks.
* Production-grade TLS setup, observability and incident response procedures.

**Acceptance**

* Security audit findings resolved or tracked.
* Production smoke tests for registration, handshake, CRL, and transfer pass.

**Tests**

* Penetration test
* Incident scenario runbook drills

---

## Release & deployment plan

* **Alpha:** internal builds with Phase A–C features; limited beta testers.
* **Beta:** public opt-in, Phase D–F stable; begin mobile packaging.
* **GA:** Phase G–H complete; comprehensive security review and monitoring.

Each release should include a migration plan for server trust root rotations and CRL changes.

---

## Risks & mitigation

* **Multicast-blocked networks:** Mitigate with QR pairing, broadcast fallback and clear UX.
* **Server key compromise:** Use KMS/HSM and fast rotation; maintain offline signing recovery.
* **Mobile platform limits:** Build minimal background-transfer shim and provide clear UX for background restrictions.

---

## Metrics & success criteria

* Discovery reliability: >=95% on home networks.
* Handshake success rate: >=99% for non-revoked devices.
* Transfer integrity failures: <0.1%.
* Relay usage: measured and minimized; used only when direct P2P fails or user opts in.

---

