# private.id — Privacy-Proxy Verification Service

> A zero-knowledge, in-memory identity verification proxy. Users prove who they are without handing over who they are.

---

## 1. Problem Statement

In Australia (and globally), identity verification is broken:

- Government services increasingly force citizens through third-party ID apps (myGovID, Digital iD™, etc.)
- Private companies (Uber, bars, rental agencies) demand raw license scans, passport photos, and personal data
- That data is stored indefinitely, sold, breached, or farmed
- Citizens have **no control** over where their identity ends up after handing it over
- The current model is: *"Give us everything, trust us to protect it"* — and that trust is routinely violated

**The core issue:** Verification should confirm *attributes* (age ≥ 18, valid license, residency status) — not expose *documents*.

---

## 2. Vision

**private.id** (public-facing brand: **verify.id**) is an in-memory, asymmetric-encryption-based identity proxy service.

- Users verify once with a trusted authority
- They receive a **proxy certificate** — a cryptographically signed token
- Third parties validate the certificate — never the underlying data
- No user PII is stored at rest. Only encrypted verification IDs exist on disk
- The service is a **conduit**, not a **vault**

Think of it as PGP for identity: you publish your public verification, keep your private data to yourself.

---

## 3. Core Principles

| # | Principle | Detail |
|---|-----------|--------|
| 1 | **Zero Storage** | No PII at rest. Raw identity data exists only in-memory during the verification ceremony, then is discarded. |
| 2 | **Asymmetric Trust** | Uses public/private keypair model. The public verification ID is shareable; the private key stays with the user. |
| 3 | **Proxy, Not Vault** | We are a proxy layer, not a data warehouse. This dramatically reduces our threat surface. |
| 4 | **Attribute-Based** | Certificates attest to *attributes* ("age ≥ 18", "valid AU license holder") — not raw document data. |
| 5 | **User Sovereignty** | Users control what attributes are exposed per certificate. They can revoke at any time. |
| 6 | **Government-Funded** | Free for citizens. Funded as public infrastructure, like roads or Medicare. |
| 7 | **Open Protocol** | The verification protocol is open and auditable. Trust comes from transparency, not obscurity. |

---

## 4. How It Works — The Flow

### 4.1 Enrollment (One-Time Verification Ceremony)

```
┌──────────┐         ┌─────────────┐         ┌──────────────────┐
│  Citizen  │────────▶│  verify.id  │────────▶│  Gov Authority   │
│           │  submit │  (in-memory │  verify │  (DVS / myGov /  │
│           │   docs  │   only)     │  docs   │  state registry) │
└──────────┘         └─────────────┘         └──────────────────┘
                            │
                     docs verified ✓
                     generate keypair
                     discard raw docs
                            │
                            ▼
                  ┌───────────────────┐
                  │  Proxy Certificate │
                  │  + Private Key     │──────▶ returned to citizen
                  │  + Public Ver. ID  │──────▶ stored (encrypted, no PII)
                  └───────────────────┘
```

1. Citizen submits identity documents via secure session (TLS + ephemeral encryption)
2. verify.id forwards documents to the **Document Verification Service (DVS)** or equivalent government API — in-memory only
3. Government authority responds: ✓ valid / ✗ invalid
4. On success:
   - An **asymmetric keypair** is generated
   - A **proxy certificate** is created, signed by verify.id's authority key
   - The certificate contains only **attributes** (not raw data): age bracket, license validity, residency, etc.
   - The **private key** is returned to the citizen (we do NOT keep it)
   - The **public verification ID** is stored (encrypted at rest) — this is the only thing on disk
   - All raw documents are **purged from memory**
5. Citizen receives: private key + certificate + public verification ID

### 4.2 Verification (Ongoing Use)

```
┌──────────┐  present cert   ┌──────────┐  check public ID   ┌─────────────┐
│  Citizen  │───────────────▶│  Uber /   │──────────────────▶│  verify.id  │
│           │                │  Bar /    │                    │  public API │
│           │                │  Service  │◀───────────────────│             │
└──────────┘                └──────────┘   ✓ valid + attrs   └─────────────┘
```

1. Citizen presents their **proxy certificate** to a third party (Uber, a bar, a rental agency)
2. Third party sends the **public verification ID** to verify.id's public API
3. verify.id responds with:
   - ✓/✗ certificate validity
   - Confirmed attributes the citizen chose to expose (e.g., "age ≥ 18": true)
   - **Nothing else.** No name, no address, no license number.
4. Third party gets exactly what they need — nothing more

### 4.3 Revocation

- Citizens can **revoke** any certificate at any time via their private key
- Revoked certificates immediately fail verification
- Citizens can **re-enroll** to generate new certificates
- Automatic expiry after configurable TTL (e.g., 12 months, then re-verify)

---

## 5. Architecture — What Lives Where

### 5.1 Data at Rest (Minimal)

| Data | Stored? | Format | Notes |
|------|---------|--------|-------|
| Raw identity documents | ❌ NEVER | — | Only exists in-memory during verification ceremony |
| User PII (name, DOB, address) | ❌ NEVER | — | Extracted as attributes, raw data discarded |
| Public Verification ID | ✅ | Encrypted blob | The public half — allows third parties to validate |
| Certificate metadata | ✅ | Encrypted | Attribute attestations, expiry, revocation status |
| Private key | ❌ NEVER | — | Only the citizen holds this |
| verify.id signing key | ✅ | HSM-backed | Used to sign certificates; stored in hardware security module |

### 5.2 System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        verify.id Platform                        │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  Enrollment   │  │  Verification│  │  Certificate Manager   │ │
│  │  Service      │  │  API (Public)│  │  (revoke/renew/expire) │ │
│  │              │  │              │  │                        │ │
│  │  - accepts   │  │  - receives  │  │  - TTL enforcement     │ │
│  │    docs      │  │    pub ID    │  │  - revocation list     │ │
│  │  - calls DVS │  │  - returns   │  │  - re-enrollment flow  │ │
│  │  - generates │  │    attrs +   │  │                        │ │
│  │    keypair   │  │    validity  │  │                        │ │
│  │  - issues    │  │              │  │                        │ │
│  │    cert      │  │              │  │                        │ │
│  └──────────────┘  └──────────────┘  └────────────────────────┘ │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  Crypto       │  │  Encrypted   │  │  Audit / Transparency  │ │
│  │  Engine       │  │  Store       │  │  Log                   │ │
│  │              │  │              │  │                        │ │
│  │  - keypair   │  │  - pub IDs   │  │  - all operations      │ │
│  │    generation│  │  - cert meta │  │    logged (no PII)     │ │
│  │  - signing   │  │  - NO PII    │  │  - publicly auditable  │ │
│  │  - HSM       │  │              │  │                        │ │
│  └──────────────┘  └──────────────┘  └────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. The Proxy Certificate — Structure

A certificate is a signed, structured token. Conceptually:

```json
{
  "version": 1,
  "certificate_id": "vf_c_8a3b...",
  "public_verification_id": "vf_pub_7f2e...",
  "issued_at": "2026-04-03T00:00:00Z",
  "expires_at": "2027-04-03T00:00:00Z",
  "issuer": "verify.id",
  "attributes": {
    "age_over_18": true,
    "age_over_25": false,
    "au_license_valid": true,
    "au_resident": true,
    "full_name_verified": true
  },
  "signature": "base64(sign(verify_id_private_key, hash(certificate_body)))"
}
```

Key design decisions:
- **Attributes are boolean or categorical** — never raw values. "age_over_18: true", not "dob: 1998-05-12"
- **Selective disclosure** — citizen chooses which attributes to include when presenting to a third party
- **Signature chain** — certificate is signed by verify.id's authority key, verifiable by anyone with the public key

---

## 7. Threat Model — Why We're a Lower-Value Target

### 7.1 What an Attacker Gets if They Breach Us

| Traditional ID Service | verify.id |
|----------------------|-----------|
| Names, DOBs, addresses | ❌ Not stored |
| License/passport scans | ❌ Not stored |
| Phone numbers, emails | ❌ Not stored (or minimal, hashed) |
| Biometric data | ❌ Not stored |
| **Everything needed for identity theft** | **Encrypted public verification IDs + attribute booleans** |

An attacker breaching verify.id gets: a database of encrypted blobs that say things like "this anonymous ID has a valid license" — **useless for identity theft**.

### 7.2 Remaining Attack Vectors

| Vector | Mitigation |
|--------|-----------|
| Compromise of signing key | HSM-backed keys, multi-party key ceremonies, key rotation |
| Interception during enrollment | End-to-end encryption, certificate pinning, ephemeral session keys |
| Fake enrollment (fraudulent docs) | Delegated to government DVS — we don't make the trust decision |
| Certificate forgery | Asymmetric signatures — computationally infeasible without signing key |
| Replay attacks | Nonce-based challenge-response for real-time verification |
| Denial of service | Standard DDoS mitigation; degraded mode returns cached validity |
| Insider threat | No PII to exfiltrate; audit logs; principle of least privilege |
| Government coercion / backdoor | Open protocol, reproducible builds, transparency reports, warrant canary |

---

## 8. Selective Disclosure — How Users Control What's Shared

Not every third party needs the same information:

| Scenario | Attributes Disclosed |
|----------|---------------------|
| Buying alcohol at a bar | `age_over_18: true` |
| Signing up for Uber | `au_license_valid: true`, `age_over_18: true` |
| Renting an apartment | `au_resident: true`, `full_name_verified: true` |
| Age-restricted online content | `age_over_18: true` |
| Financial service (KYC) | `full_name_verified: true`, `au_resident: true`, `age_over_18: true` |

The citizen generates a **scoped certificate** for each use case — a subset of their full attribute set. The third party only sees what's relevant.

---

## 9. Technical Stack (Proposed)

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Backend API** | Rust or Go | Memory safety, performance, no GC pauses during crypto ops |
| **Crypto** | libsodium / ring (Rust) | Audited, modern primitives (Ed25519, X25519, XChaCha20-Poly1305) |
| **Key Storage** | AWS CloudHSM / Azure HSM / self-hosted Nitrokey HSM | Signing keys never leave hardware |
| **Encrypted Store** | SQLite (encrypted via SQLCipher) or PostgreSQL with column-level encryption | Minimal data, encrypted at rest |
| **Citizen App** | Progressive Web App (PWA) | No app store gatekeeping, works on any device, offline capable |
| **Verification API** | REST + optional mTLS for B2B | Simple integration for third parties |
| **Audit Log** | Append-only, Merkle-tree based | Tamper-evident, publicly verifiable |
| **Infrastructure** | Australian-hosted (AU sovereignty) | Data never leaves AU jurisdiction |

---

## 10. User Experience — Citizen Journey

### First Time (Enrollment)
1. Visit **verify.id** on any device
2. Create an account (email or phone — hashed, used only for recovery)
3. Upload identity documents (license, passport, Medicare card)
4. Documents are verified against government DVS **in real-time, in-memory**
5. Receive your **private key** (stored locally on device) and **proxy certificate**
6. Documents are **immediately discarded** — never stored
7. Done. You now have a verified digital identity that reveals nothing.

### Ongoing (Presenting to Third Parties)
1. Third party says "verify your age" or "verify your identity"
2. Open verify.id → select which attributes to share → generate scoped certificate
3. Present certificate (QR code, NFC tap, or digital link)
4. Third party scans/checks → gets a ✓ and the attributes you chose → done
5. No license photo. No home address. No data harvesting.

### Recovery
1. Lost your device? Use recovery email/phone (hashed) to initiate re-enrollment
2. Old certificates are automatically revoked
3. New keypair generated, new certificate issued
4. Previous verification may be fast-tracked (government DVS may cache approval status)

---

## 11. Government Integration — DVS & Existing Infrastructure

Australia already has the **Document Verification Service (DVS)** operated by IDMATCH (Attorney-General's Department). verify.id doesn't replace it — it **wraps** it:

```
Citizen ──▶ verify.id ──▶ DVS (government) ──▶ response ──▶ verify.id ──▶ Citizen
                                                              │
                                                      (discard docs,
                                                       issue certificate)
```

- verify.id becomes a **DVS Access Provider** (already a defined role in AU identity infrastructure)
- We call DVS APIs to validate documents — we don't make trust decisions ourselves
- This means verify.id inherits government-grade verification without storing anything

---

## 12. Revenue & Sustainability Model

| Phase | Funding | Notes |
|-------|---------|-------|
| **Phase 1** (0–3 years) | Government grant / Digital Identity program funding | Position as public infrastructure |
| **Phase 2** (3–5 years) | Government operational funding + B2B API fees | Third parties pay per-verification (small fee, e.g., $0.05) |
| **Phase 3** (5+ years) | Self-sustaining via B2B fees | Free for citizens forever; businesses pay for API access |

Citizens **never pay**. This is a public utility.

Third-party businesses pay a nominal per-verification fee — far cheaper than building their own ID verification, and they get reduced liability (they never hold PII).

---

## 13. Legal & Compliance Considerations

- **Privacy Act 1988 (AU)** — We collect minimal data; our architecture is Privacy by Design
- **Trusted Digital Identity Framework (TDIF)** — Target TDIF accreditation
- **Consumer Data Right (CDR)** — Compatible with CDR principles
- **GDPR (if expanding beyond AU)** — Zero-storage model is inherently GDPR-friendly
- **Digital Identity Act 2024 (AU)** — Align with emerging digital identity legislation
- **Open-source transparency** — Protocol and client code published for public audit

---

## 14. Development Phases

### Phase 0 — Foundation (Now)
- [ ] Finalize protocol specification (this document → formal spec)
- [ ] Define certificate format and crypto primitives
- [ ] Prototype keypair generation and certificate signing
- [ ] Build proof-of-concept enrollment flow (mock DVS)

### Phase 1 — MVP
- [ ] Backend API (enrollment + verification endpoints)
- [ ] Crypto engine (keypair gen, signing, verification)
- [ ] Encrypted store for public verification IDs
- [ ] Basic PWA client (enrollment + certificate presentation)
- [ ] Mock DVS integration (simulate government verification)

### Phase 2 — Government Integration
- [ ] Apply for DVS Access Provider status
- [ ] Integrate with real DVS APIs
- [ ] TDIF accreditation process
- [ ] Security audit (independent third party)
- [ ] Pen testing

### Phase 3 — Public Beta
- [ ] Public launch (AU citizens)
- [ ] B2B API for third-party integrators
- [ ] QR code / NFC presentation flow
- [ ] Certificate management dashboard for citizens

### Phase 4 — Scale
- [ ] Multi-jurisdiction support (NZ, UK, EU)
- [ ] Additional attribute types (professional licenses, certifications)
- [ ] Decentralized verification option (blockchain-anchored certificate hashes)
- [ ] Hardware key support (YubiKey, etc.)

---

## 15. Open Questions

1. **Key recovery** — If a user loses their private key and recovery method, how do we handle re-enrollment without creating a new identity? Can DVS approval be cached/referenced?
2. **Certificate freshness** — How often should attributes be re-verified? License validity can change (suspended, expired).
3. **Offline verification** — Can third parties verify certificates without hitting the API? (Yes, with embedded signatures, but revocation checks need connectivity.)
4. **Anti-fraud** — How do we prevent one person enrolling multiple times to create multiple identities? (Likely: DVS deduplication.)
5. **Accessibility** — How do we serve citizens without smartphones? (Physical card with embedded NFC + QR?)

---

## 16. Summary — The Pitch

> **"Prove who you are without giving yourself away."**
>
> verify.id is a government-funded, zero-knowledge identity proxy. Verify once, prove forever — without handing your license to every app, bouncer, and landlord in the country.
>
> We don't store your data. We can't leak what we don't have.
>
> Your identity. Your control. Full stop.

---

*Document version: 0.1.0 — Planning phase*
*Last updated: 2026-04-03*
