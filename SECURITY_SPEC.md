# SECURITY_SPEC

## Introduction
XPScerpto is engineered to meet stringent security requirements for protecting data-at-rest and data-in-motion. This document describes the library’s cryptographic algorithms, key and memory protection, side‑channel safeguards, self‑tests, standards alignment, and operational guidance. It is written for security engineers and assessors who need to understand how XPSI Crypto achieves both present‑day and forward‑looking (post‑quantum) security.

---

## Supported Algorithms & Standards (Secure Defaults)
XPScerpto exposes modern primitives through high‑level, safe APIs and policy‑driven defaults.

### Symmetric Encryption
- **AES‑GCM** (128/192/**256**). Authenticated encryption per **NIST SP 800‑38D** (default AEAD).
- **ChaCha20‑Poly1305** per **RFC 8439**, lightweight AEAD alternative.

### Asymmetric Cryptography
- **RSA** (≥ 3072‑bit recommended for 2030+ horizon), **RSA‑PSS** for signatures.
- **ECC**: X25519 (ECDH), Ed25519 (EdDSA), ECDH/ECDSA on common NIST curves (policy‑controlled).

### Hash Functions
- **SHA‑2** (SHA‑256/384/512), **SHA‑3**.
- **BLAKE2** / **BLAKE3** for high performance hashing.
- **SHAKE128/256** (extendable‑output functions).

### MACs & Signatures
- **HMAC** (SHA‑256/384/512; also BLAKE2/3 variants if enabled).
- **Poly1305** (paired with ChaCha20).  
- **Signatures**: RSA‑PSS, ECDSA, **Ed25519** (default), and PQC signatures where enabled.

### Key Derivation
- **HKDF** (RFC 5869) for session keys from shared secrets.
- **PBKDF2** and **Argon2id** for password‑backed keys (high‑cost defaults; randomized salt).

### Randomness
- **CSPRNG**: CTR‑DRBG (NIST SP 800‑90A). Optional HWRNG seeding (e.g., RDSEED/RDRAND) when available.

### Post‑Quantum (PQC)
- **Kyber** (KEM) and **Dilithium**/**Falcon** (signatures), selectable via policy and available for hybrid schemes.

> **Policy‑driven defaults**: weak/legacy algorithms are disabled by default. Minimum key sizes and allowed modes are enforced centrally.

---

## Security Guarantees & Design Properties

### 1) Side‑channel Resistance
- Constant‑time implementations for critical paths (comparisons, MAC checks, ECDH/EdDSA internals).
- Memory access and control‑flow independent of secret data (to the extent feasible on target ISA).
- SIMD/ISA paths gated by policies to avoid variable‑latency instructions on secret material.
- Pairwise consistency checks for asymmetric keys to avoid oracle leakage.

### 2) Sensitive Memory Protection
- **SecureBuffer**/**LockedBuffer** abstractions for key material, with:
  - Explicit `secure_wipe()` using volatility barriers to prevent compiler elision.
  - Optional **RAM locking** (mlock/VirtualLock) to prevent swapping.
  - Zeroization on scope end (RAII) and on error paths.
- Prohibit accidental logging/serialization of secrets via type separation and policy guards.

### 3) Authenticated Confidentiality & Integrity
- AEAD by default (AES‑GCM / ChaCha20‑Poly1305): decryption fails closed on tag mismatch.
- Constant‑time MAC verification; no partial plaintext on authentication failure.
- Strict separation of “encrypt” vs “sign” keys via Keyring policy attributes.

### 4) Self‑Tests (Power‑up & Conditional)
- **Known Answer Tests (KATs)** at initialization for core primitives (AES, SHA‑2/3, HMAC, DRBG, Ed25519, etc.).
- **Conditional tests**:
  - Pairwise key tests on asymmetric key generation.
  - DRBG continuous health tests.
- Fail‑safe mode on self‑test failure (module enters disabled/error state).

### 5) Crypto‑Agility & Hot Patch
- Signed module updates (signature verification before load).
- **Zero‑downtime** switchover: in‑flight operations complete on the old version, new calls routed to updated module.
- Policy engine can centrally disable algorithms and set minimum strengths at runtime.

### 6) Standards Alignment
- Designed with **FIPS 140‑3** validation path in mind (see mapping file).
- Uses NIST‑approved algorithms and constructions by default; non‑approved are opt‑in with warnings.
- Supports compliance goals for **GDPR/HIPAA/PCI‑DSS** via strong encryption and key lifecycle controls.

### 7) Post‑Quantum Readiness
- Optional hybrid KEM/signature flows (e.g., X25519+Kyber; Ed25519+Dilithium) to mitigate “harvest‑now‑decrypt‑later”.

---

## Threat Model (Summary)
- **In scope**: network adversaries, compromised storage, passive side‑channel observers, software‑only attackers.
- **Partially in scope**: micro‑architectural leakages (mitigated by constant‑time discipline and policy), kernel compromise detection.
- **Out of scope**: physical tampering against non‑HSM deployments (unless combined with platform protections).

---

## Operational Guidance & Hardening

### Key Management (Keyring)
- Key generation with high entropy; tagged usage (Encrypt‑Only, Sign‑Only, Derive‑Only).
- **Rotation** (time/use‑count policies); **revocation** lists; signed key manifests.
- Encrypted at rest with password‑based wrapping (Argon2id/PBKDF2 → AEAD).

### Policies
- Centrally enforced: min key sizes, approved curves/modes, forbidden algorithms.
- Runtime switches to disable legacy algorithms without rebuilds.

### Logging & Audit
- Structured security events: key‑gen, rotation, self‑test results, auth failures, policy denials.
- Redaction by default; no secret bytes in logs.

### Build/Runtime Hardening
- Hardened compiler flags, stack canaries, ASLR, RELRO/PIE where applicable.
- Sanitizers in CI (ASan/UBSan/TSan) and reproducible builds.
- Deterministic tests with KAT vectors; fuzzing and differential tests for crypto code paths.

---

## Security Testing
- **KAT suites** for AES/SHA/HMAC/DRBG/Ed25519/etc.
- **Interoperability** tests vs. reference vectors.
- **Fuzzing** (inputs, state machines) and **property‑based** tests (e.g., AEAD misuse resistance invariants).
- **Performance** tests gated to ensure constant‑time critical paths do not regress into secret‑dependent branches.

---

## Coordinated Disclosure (Optional)
- Security contact, PGP key, and SLA for triage and fixes. Signed advisories and hot‑patches when needed.

---

## Assumptions & Limitations
- XPSI Crypto is a **software module** (no physical tamper‑evidence). Combine with OS/container hardening.
- Side‑channel resistance depends on deployment discipline (pinning cores, constant‑time build flags, disabling frequency scaling on sensitive paths when required).
- PQC algorithms are included for hybrid readiness; formal certification timelines may vary by scheme.

---

## Appendix A — Default Approved Algorithms
- AEAD: AES‑GCM(256), ChaCha20‑Poly1305
- Signatures: RSA‑PSS(≥3072), Ed25519, (PQC: Dilithium/Falcon)
- KEM/ECDH: X25519, (PQC: Kyber)
- Hashing/XOF: SHA‑256/384/512, SHA‑3, SHAKE128/256, BLAKE3
- KDF: HKDF, Argon2id, PBKDF2
- DRBG: CTR‑DRBG (with continuous tests)

---

## Appendix B — Secure Defaults (Illustrative)
- Min RSA size: 3072; ECC: X25519/Ed25519 by default.
- AEAD default: AES‑GCM‑256 with random 96‑bit nonce (unique per message).
- HKDF with SHA‑256 and explicit salt; Argon2id with memory ≥ 128 MiB, time ≥ 3 and parallelism ≥ 2 (tunable by policy).
- Keys in memory: LockedBuffer + mandatory zeroization on scope end.
