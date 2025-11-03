# ARCHITECTURE.md
**XPScerpto — System Architecture (English Edition)**

> **Goal.** XPScerpto is a production‑grade, modular cryptography platform built with modern C++ (C++20/23 Modules). The architecture emphasizes **crypto‑agility**, **zero‑downtime updates**, **defense‑in‑depth**, and **high performance** across multiple CPU architectures.

---

## 1. Architectural Principles

- **Crypto‑Agility by Design**
  - Swap algorithms, key sizes, and even whole implementations **without application downtime**.
  - Policy‑driven enable/disable and preference of algorithms; hybrid (classical + PQC) compositions supported.
- **Strict Modularity & Clear Boundaries**
  - Every primitive lives in its own module (e.g., `xps.crypto.hash.sha256`, `xps.crypto.aead.aes_gcm`).
  - Public importable interfaces are separated from private implementation units.
- **Zero‑Downtime Operations**
  - Hot‑patching of cryptographic modules with signature‑verified payloads and guarded activation.
  - Atomic switchover with rollback and post‑install self‑tests.
- **Defense‑in‑Depth**
  - Constant‑time code paths, clean memory hygiene, integrity checks, policy gates, and attestation hooks.
- **Cross‑Platform Performance**
  - Runtime ISA detection and **SIMD dispatch** (x86: SSE/AVX/AVX2/AVX‑512, ARM: NEON/SHA, RISC‑V: RVV when available).
  - Cache‑friendly data layouts, minimal copies, and concurrency‑safe APIs.
- **Operator‑Friendly**
  - Versioned components, explicit compatibility policy, self‑tests on startup, rich logging/metrics.

---

## 2. Layered Architecture (Overview)

```
Applications / Services
        │
        ▼
[ L5 ] Super Crypto & Unified API
        │  (routing, providers, bridges)
        ▼
[ L4 ] Orchestration & Hot Patch
        │  (zero-downtime updates, key rotation control, policy gates)
        ▼
[ L3 ] Key Management & Security Services
        │  (keyring, storage, policy, attestation, audit)
        ▼
[ L2 ] Core Crypto Modules
        │  (hash, AEAD, MAC, KDF, signatures, KEM/PQC)
        ▼
[ L1 ] Utilities & Platform Abstraction
           (arch detect, SIMD dispatch, memory, random, logging, compat)
```

### Responsibilities by Layer

- **L5 — Super Crypto & Unified API**
  - Single import (e.g., `import xps.crypto.api;`) that re‑exports stable, curated higher‑level interfaces.
  - Intelligent routing: pick best algorithm or path per policy, platform capabilities, and risk profile.
  - Enterprise bridges/providers for external KMS/HSM/cloud or legacy systems.

- **L4 — Orchestration & Hot Patch**
  - Hot‑patch framework (repository, verifier, security gate, installer, rollback).
  - Zero‑downtime coordination for upgrades to algorithms and to key material distribution.
  - Scheduling and coordination services (e.g., distributed key rotation).

- **L3 — Key Management & Security Services**
  - **Keyring**: lifecycle (generate, import/export, wrap/unwrap, rotate, revoke, audit).
  - Policies: allowed algorithms, min key sizes, lifetimes, usages, jurisdictional constraints.
  - Storage backends: secure memory buffers, encrypted persistence; optional external providers (KMS/HSM).
  - Multi‑party cryptography (threshold signing/sharing) and blockchain‑style key use cases.
  - Attestation hooks and audit logging for sensitive operations.

- **L2 — Core Crypto Modules**
  - **Hashes:** SHA‑2 (256/384/512), SHA‑3, SHAKE128/256, BLAKE2b/s, BLAKE3.
  - **AEAD:** AES‑GCM (128/256), ChaCha20‑Poly1305, XChaCha20‑Poly1305.
  - **MAC:** HMAC (SHA‑2/3/BLAKE3), Poly1305.
  - **KDF:** HKDF, PBKDF2, Argon2id.
  - **Signatures & KEM:** RSA (PSS), Ed25519 / X25519, **PQC** (Dilithium, Falcon‑1024, Kyber‑1024).
  - Constant‑time operations where applicable; pluggable implementations per algorithm.

- **L1 — Utilities & Platform Abstraction**
  - Arch/ISA detect, SIMD dispatch registry, secure random, hex/base64, JSON helpers.
  - Memory primitives: `LockedBuffer`, secure wipe, constant‑time compare.
  - Logging/metrics, error model (`expected<T,E>`), compat shims for OS/compiler quirks.

---

## 3. Module Taxonomy & Naming

- `xps.crypto.hash.<name>` — Hash functions and XOFs.
- `xps.crypto.aead.<name>` — Authenticated encryption.
- `xps.crypto.mac.<name>` — Message authentication codes.
- `xps.crypto.kdf.<name>` — Key derivation functions.
- `xps.crypto.ed25519`, `xps.crypto.rsa`, `xps.crypto.pqc.<algo>` — Signatures/KEM.
- `xps.crypto.keyring.*` — Key lifecycle, storage, rotation, providers.
- `xps.crypto.orchestrator.*` / `xps.crypto.hot_patch.*` — Update and rollout control.
- `xps.crypto.api` — Unified, high‑level re‑export.
- `xps.utils.*` / `xps.internal.*` — Utilities and private helpers.

**Rule of thumb:** Interfaces (importable) do not expose implementation details; impls can be swapped or hot‑patched without API breakage.

---

## 4. Runtime Dispatch & Performance

- **ISA Sensing & Dispatch**
  - On startup, detect CPU features; register fast paths (e.g., AVX2 for BLAKE3, AES‑NI for AES‑GCM, ARM SHA for SHA‑2).
  - Fallback to portable scalar code if features are unavailable.
- **Data & Memory**
  - Aligned contexts, contiguous blocks, `std::span`‑based APIs to avoid copies.
  - Secure wiping of ephemeral secrets; constant‑time compares for tags/keys.
- **Concurrency**
  - Stateless APIs or confined state make parallelism straightforward.
  - BLAKE3 and some KDF/AEAD paths can scale across threads when driven by the application.

---

## 5. Security Model & Non‑Goals

**Assumptions**  
- Trusted build pipeline and signed patch payloads.  
- No secret leaves `LockedBuffer` in clear unless explicitly exported under policy.  

**Mitigations**  
- Constant‑time critical code paths (MAC/AEAD/verify/compare).  
- Strict input validation; domain separation where applicable.  
- Memory hygiene (wipe on scope exit, avoid accidental copies), no UB in hot paths.  
- Policy gates + attestation check before dangerous operations (e.g., hot‑patch activation).

**Non‑Goals**  
- Implementing network protocols (TLS/QUIC) end‑to‑end. XPS Crypto provides the *primitives* to build them.  
- Providing a general‑purpose secrets manager UI — integrate with providers instead.

---

## 6. Dataflows (Representative)

### 6.1 Unified AEAD Encryption
1. App calls `api.encrypt(…)` with key, IV/nonce, plaintext, AAD.
2. Unified API selects algorithm per policy (e.g., prefer AES‑GCM if AES‑NI is present; else ChaCha20‑Poly1305).
3. Selected module:
   - Expands key (AES) or seeds state (ChaCha20).
   - Processes blocks; updates GHASH (GCM) or Poly1305 (ChaCha) in constant time.
4. Returns `{ciphertext, tag, iv}`; metrics/logs recorded (if enabled).

### 6.2 Hybrid (Classical + PQC) Key Establishment
1. Generate random session key (symmetric).
2. Encapsulate under RSA/ECDH **and** Kyber (dual encapsulation).
3. Transmit both artifacts; receiver decapsulates both. 
4. Policy mandates: require both successes or KDF‑combine to resist “harvest‑now, decrypt‑later”.

---

## 7. Hot‑Patch Workflow (Zero‑Downtime)

1. **Prepare Patch**: `manifest.json` (+ signer info, targets, hash) and `payload` (e.g., shared object / module impl).
2. **Fetch** (Repository): filesystem / remote provider.
3. **Verify** (Verifier): signature & digest; version/compat checks.
4. **Gate** (Security Gate): policy/time window/allow‑list validation.
5. **Install** (Installer): load payload; swap function table / impl pointer atomically.
6. **Self‑Test**: run KAT/smoke for the target module.
7. **Activate**: new calls use the new impl; in‑flight ops finish on the old impl.
8. **Rollback (if needed)**: revert atomically on failure; keep audit trail.

**Properties**: bounded pause, no process restart, idempotent, auditable.

---

## 8. Key Rotation Workflow (Zero‑Downtime)

1. **Policy**: min key sizes, lifetime (e.g., 90d), overlap window (e.g., 7d), usages.
2. **Monitor**: orchestrator schedules rotation based on age/usage thresholds.
3. **Generate**: create new key material inside Keyring (secure RNG, correct domain params).
4. **Switch‑Over**: new encryptions use new key; old key remains **decrypt‑only** during overlap.
5. **Distribute**: notify/propagate to distributed nodes; ensure version sync.
6. **Retire**: after overlap, wipe/archive old key; update audit logs.

**Guarantees**: no service interruption; decryptability preserved; atomic pointer swap inside Keyring; strong auditing.

---

## 9. API & Error Model

- **Import‑First**: prefer `import xps.crypto.<…>;` interfaces; linkless integration for modern toolchains.
- **Types**: strong types for keys/tags/IVs; sized byte containers; spans over copies.
- **Errors**: `expected<T, Error>` for hot paths; classified `ErrorCode` with actionable context.
- **Logging & Metrics**: structured logs; optional high‑resolution timers; production‑safe redaction of secrets.

---

## 10. Compliance & Self‑Tests

- **Modes**: standard, “FIPS‑like” constrained mode (policy gates enforce approved algorithms/key sizes).
- **Self‑Tests**: power‑on KATs for critical primitives; optional continuous health checks.
- **Provenance**: patch payloads and config are signed; audit trails for key events, rotations, and patches.

---

## 11. Extensibility

- **Providers/Bridges**: adapters for HSM/KMS/cloud, legacy APIs, or blockchain wallets.
- **New Algorithms**: add a new module, register in API catalog, implement self‑tests + metrics hooks.
- **Alternate Implementations**: side‑by‑side impls under same interface (e.g., portable vs AVX2 vs ARM SHA).

---

## 12. Deployment Profiles

- **Server**: maximum throughput, SIMD on, background rotations/patch scans, rich metrics.
- **Client/Desktop**: balanced footprint, reduced telemetry, OS‑integrated storage provider.
- **Embedded/Edge**: minimal build, scalar fallbacks, NEON/RVV when available, static key policies.

---

## 13. Glossary

- **AEAD**: Authenticated Encryption with Associated Data.  
- **KAT**: Known‑Answer Test.  
- **KDF**: Key Derivation Function.  
- **PQC**: Post‑Quantum Cryptography.  
- **Hot‑Patch**: Replace code at runtime with verifiable, signed payloads.  
- **LockedBuffer**: Secure memory that resists swapping/copying and is wiped on release.

---

## 14. Appendix — Example Module Map

```
xps.crypto.api
 ├─ xps.crypto.hash.{sha256,sha384,sha512,sha3,shake128,shake256,blake2b,blake2s,blake3}
 ├─ xps.crypto.aead.{aes_gcm,chacha20_poly1305,xchacha20_poly1305}
 ├─ xps.crypto.mac.{hmac,poly1305}
 ├─ xps.crypto.kdf.{hkdf,pbkdf2,argonn2id}
 ├─ xps.crypto.{rsa,ed25519,x25519}
 ├─ xps.crypto.pqc.{kyber1024,dilithium5,falcon1024}
 ├─ xps.crypto.keyring.{core,policy,storage,auto_rotation,multiparty,distributed}
 └─ xps.crypto.orchestrator.{core,hot_patch,verifier,repo,gate,attest,installer}
```

> This architecture turns XPScerpto into a **future‑proof security substrate**: modular, measurable, updatable in place, and ready for hybrid classical/PQC operations without disrupting applications or users.
