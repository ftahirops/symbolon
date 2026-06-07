<div align="center">

# ⬡ Symbolon

### _σύμβολον_ — the door is **closed by default**

**An open authentication protocol where login authority lives in an external _Vault_, not in your app.**
The Vault mints short-lived, single-use, cryptographically signed **Login Leases** that your application verifies **offline** before opening a session.

<br>

![status](https://img.shields.io/badge/status-draft%20v0.1-d9824a?style=for-the-badge)
![wire](https://img.shields.io/badge/wire%20version-1-5fb8a8?style=for-the-badge)
![crypto](https://img.shields.io/badge/crypto-Ed25519_%2B_JCS-e8c87a?style=for-the-badge)
![verifier](https://img.shields.io/badge/verifier-under_500_LOC-c98a4f?style=for-the-badge)
![license](https://img.shields.io/badge/license-TBD-lightgrey?style=for-the-badge)

**[🔍 Live showcase →](https://www.xgenstack.com/symbolon)**  ·  **[📜 Read the spec →](./spec/symbolon-protocol-v0.1.md)**  ·  **[🛡 Threat model →](./spec/symbolon-threat-model-v0.1.md)**

</div>

---

> _Greek **σύμβολον**: a clay token broken cleanly in two. Each party kept one half; matching the halves proved identity. The original one-time identity proof — and the etymological root of the word "symbol."_

## ⚡ The one claim worth making

> ### Symbolon doesn't make compromise _impossible_. It makes **single-point compromise insufficient.**

Passwords, API keys, bearer tokens and session cookies all share one fatal shape: a **long-lived secret that grants standing access**. Steal it once and the door stays open — on any device, from anywhere, until someone notices. Symbolon inverts this. The door is closed by default, and the user **mints a lease per session** — one-time, time-boxed, device-bound, app-bound, action-bound.

---

## 🔄 How it works

```text
   🧑  SUBJECT  (the user)
        │
        │  ①  proves locally — biometric / PIN / passkey
        ▼
   🔐  VAULT  (issuer · hardware-backed)
        │
        │  ②  mints + signs the lease with its hardware key
        │      and generates a fresh ephemeral keypair
        ▼
   🌐  USER-AGENT  (hostile carrier)
        │
        │  ③  carries the lease + ephemeral private key
        ▼
   🛡  VERIFIER  (your app)
        │
        │  ④  challenge  ⇄  signed nonce        (proof of possession)
        │  ⑤  verify signature OFFLINE · enforce bindings · consume JTI
        ▼
   ✅  BOUNDED SESSION
```

Two things **never** happen: no long-lived secret ever reaches your app, and the Verifier **never phones home** to a central authority — it checks the signature against the Vault's published public key, locally.

---

## ✨ Features

| | Feature | What it means |
|---|---|---|
| 🔂 | **Single-use** | Each lease has a unique `jti`, consumed atomically on first use. Replays are rejected. |
| ⏱️ | **Time-boxed** | `not_before` / `not_after` — a window measured in **minutes**, not days. |
| 📱 | **Device-bound** | A lease is tied to one user-agent. Lifted to another device, it's dead. |
| 🎯 | **App- & action-bound** | Usable only at one `app_id`, only for its `allowed_actions`. Writes need a new lease. |
| 🌍 | **Geo-fenced** | Optional `geo_constraint` pins login to a country, ASN, or IP range. |
| ✍️ | **Un-forgeable** | Ed25519 signature over the JCS (RFC 8785) canonical form. One changed byte fails. |
| 📡 | **Offline-verifiable** | No callback to any server — verify against a cached public key. |
| 🔒 | **Hardware-backed** | The Vault's signing key never leaves a Secure Enclave / TPM / HSM. |
| 🙅 | **Fail-closed** | Unknown vault, bad sigalg, untrusted clock, or unavailable replay store all reject. |
| 📓 | **Auditable** | Every mint is a hash-chained audit entry. |
| 🚨 | **Revocable** | Panic-lock / revocation epoch can kill outstanding leases. |
| 🔌 | **Transport-agnostic** | HTTP header, QR / deep link, BLE, USB. |

---

## 🆚 Against the alternatives

| Property | Password + 2FA | Passkey / WebAuthn | OAuth token | **Symbolon lease** |
|---|:---:|:---:|:---:|:---:|
| Long-lived secret at server | ✅ has one | ❌ | ✅ refresh | ❌ **none** |
| One-time use | ❌ | ❌ | ❌ | ✅ |
| Time-boxed to minutes | ❌ | ❌ | ◑ | ✅ |
| Device-bound | ❌ | ◑ | ❌ | ✅ |
| Action-bound (fine-grained) | ❌ | ❌ | ◑ | ✅ |
| Offline-verifiable | ✅ | ✅ | ❌ | ✅ |
| Independent of an IdP | n/a | ❌ | ❌ | ✅ |

---

## 🧩 The lease

```jsonc
{
  "v": 1,
  "type": "symbolon.lease",
  "jti": "01JBYZK8R7E1Q3X9TWMV4PG2YH",   // single-use id
  "vault_id": "vault:01JBYZ…",
  "subject_id": "usr_01JBYZ…",
  "app_id": "xgs.admin",                   // app-bound
  "device_id": "dev_01JBYZ…",              // device-bound
  "not_before": "2026-06-04T19:00:00Z",
  "not_after":  "2026-06-04T19:10:00Z",    // 10-minute window
  "max_session_ttl_sec": 1800,
  "allowed_actions": ["login", "read"],    // action-bound
  "geo_constraint": { "countries": ["NL"] },
  "ephemeral_pubkey": "z6Mk…",
  "nonce_challenge_required": true,
  "vault_signature": "z3Ms…"               // Ed25519 over JCS
}
```

The signed form is JCS (RFC 8785) of the object with `vault_signature` removed; signature is Ed25519, multibase base58btc (`z` prefix).

---

## 🎯 Where it fits (honest positioning)

Symbolon is an **authorization gate**, not a secrets manager — and not magic.

- **First-party (your own systems)** → Symbolon sits **directly in the path**: auth, session, API, sensitive actions, data gate. Each boundary verifies a lease locally. This is where it replaces standing secrets outright.
- **Third-party (AWS, SaaS)** → AWS will not verify a lease. Symbolon sits **in front of a broker** (HashiCorp Vault / AWS STS): the broker issues the provider's _native_ short-lived credential only after a valid lease. It **gates issuance — it never replaces IAM and stores no secret.**

> **vs HashiCorp Vault:** Vault is a secrets **store/broker** (its compromise is the jackpot). Symbolon stores **no downstream secrets** — the issuer holds only a signing key, verifiers hold only public keys + spent JTIs. They're complementary: **Vault can _be_ the Symbolon issuer**, taking the server out of the hot path.

---

## 🧨 Breach scenarios — and how each is contained

> Legend: 🟢 defeated · 🟡 strongly contained / mitigated · ⚪ out of scope (adjacent layer)

| Breach / attack | Normal app outcome | How Symbolon contains it | Result |
|---|---|---|:---:|
| **Credential DB dump** | All hashes leak; cracked offline; reused elsewhere | No password / long-lived secret is stored to dump | 🟢 |
| **Phishing for credentials** | Reusable secret captured & replayed | Nothing reusable exists; single-use, app-bound, signed lease | 🟢 |
| **Stolen token / cookie replay** | Works repeatedly, from anywhere | One-time `jti` consumed atomically; bound to a non-exportable session key | 🟢 |
| **Token used past expiry** | Long TTL still valid | Minutes-long window vs a trusted clock | 🟢 |
| **Token lifted to another device** | Bearer works anywhere | Cryptographic device binding | 🟢 |
| **Lease tampering** | Editable claims widen access | Ed25519 over JCS covers every field | 🟢 |
| **Real-time phishing relay (AiTM)** | MFA proxied & bypassed | Mandatory origin/audience binding _(v1)_ + live possession challenge | 🟡 |
| **Physical device theft** | Saved creds handed over | Fresh biometric per mint; signing key sealed in hardware | 🟢 |
| **Verifier / app-server breach** | Secrets + sessions exfiltrated | No secret store; issuer keys pinned via signed config | 🟡 |
| **Issuer / Vault key compromise** | — | Segmented + **co-signed** issuers; revocation epoch; hash-chained audit | 🟡 |
| **Supply chain (CI secret / npm token)** | Poisoned release shipped | Proof-bound, single-use release lease via broker; no standing CI secret | 🟡 |
| **Insider / admin misuse** | One admin = god mode | Step-up + quorum + scoped admin leases + audit | 🟡 |
| **Session hijack after login** | Stolen cookie rides the session | Session bound to a non-exportable key (DPoP / WebAuthn) | 🟡 |
| **Clock-skew abuse** | Slow server honors expired lease | Skew ≤ 30 s; **fail-closed** if clock untrusted | 🟢 |
| **Replay-store outage** | — | **Fail-closed**: reject rather than risk a double-spend | 🟢 |
| **Endpoint malware at mint time** | Full takeover | Limits duration & scope only — _does not_ stop a present attacker | ⚪ |

---

## 🧱 No single point of failure · graceful fallback

**Design principles that remove the single catastrophic point:**

- **Offline verification** — no central server in the request path, so the issuer being down never stops verification of already-minted leases.
- **Segmented issuers** — separate keys for login / admin / workload / broker / recovery; one compromise affects only its domain.
- **Co-signing (k-of-n)** for high-risk issuers — no single signer can mint alone (multi-signature leases; MPC is a later optimization, not a dependency).
- **Durable, atomic replay store** — consumption survives reboots; concurrent double-spend impossible.
- **Signed revocation snapshots** — short TTL + monotonic anti-rollback `seq`; a fast kill switch with no per-request callback.

**Fallback behavior — _"when X fails"_:**

| Component fails | What still works | Fallback behavior |
|---|---|---|
| **Issuer / Vault offline** | Verification of existing leases; live sessions | New mints pause; no new sensitive actions until it returns |
| **One issuer key compromised** | Every other domain | Revoke that key's epoch via the signed feed; co-signers still required |
| **Verifier instance down** | Any other instance (stateless verify) | Horizontal — any node verifies with cached keys |
| **Revocation feed stale** | Low-risk reads | Sensitive actions **blocked** until a fresh snapshot (degrade closed) |
| **Revocation feed unreachable** | Cached snapshot until its `expires_at` | After expiry: **fail-closed** for sensitive; basic availability continues |
| **Replay store unavailable** | — | **Fail-closed**: reject |
| **Clock untrusted (NTP down)** | — | **Fail-closed**: reject rather than guess time |
| **Network partition (data ↔ control plane)** | Offline verification | Mint + revocation refresh pause; verify continues on cached state |
| **Lost user device / Vault** | Account intact | Recovery issuer + quorum (break-glass) re-enrolls a new device |

> This is constraint #4 made concrete: **failure degrades closed for sensitive actions, while basic availability continues.**

---

## 📉 Critical analysis — damage & blast-radius reduction

> **Not a scientific guarantee.** These are engineering estimates for a **Symbolon-first first-party app** (you own the app, API, DB, broker and policy). Real numbers depend on implementation depth and adoption. Measure the delta against a _modern_ baseline (passkeys + short-lived tokens + DPoP), not a password+cookie app.

**The addressable slice of today's breaches:**

- **~80%** of breaches involve a credential or human element.
- Of those, the **majority rely on a reusable / standing / bearer secret** — the part Symbolon structurally removes for systems you control. Realistically **~55–70% of all breaches are _structurally addressable_** by first-party Symbolon.
- The rest — endpoint malware at mint, socially-engineered approval, third-party SaaS you don't own — are **largely out of scope**, and Symbolon should not claim them.

**Where the reduction actually lands — it shrinks _impact and dwell time_ more than _initial intrusion_:**

| Dimension | Today | Symbolon-first | Effect |
|---|---|---|---|
| Credential lifetime | months → until manual rotation | **minutes** | ~100–1000× shorter window |
| Scope of one credential | broad / standing | **one action, one resource** | least-privilege by default |
| Lateral movement | ambient session unlocks everything | **each hop needs a fresh lease** | escalation no longer automatic |
| Secrets exposed by one store breach | many | **≈ zero** (no store) | the "jackpot" disappears |
| Attacker dwell time | weeks–months (until noticed) | **bounded by TTL + revocation epoch** | minutes–hours |
| Single-credential takeover | common | **insufficient on its own** | needs several factors at once |

**Blast radius — what _one_ compromise yields:**

| Attacker gets… | Normal app | Symbolon-first |
|---|---|---|
| A leaked secret / key | Standing access until rotated | An already-expired, scoped ticket |
| App-server shell | Direct path to data & secrets | A shell that still lacks a fresh action lease |
| The credential store | **Everything** | There is no store — nothing to take |
| One issuer key | — | One **segment** only; co-signers still required |

**Honest bottom line:**

- **Initial intrusion likelihood:** modestly reduced (phishing/credential-stuffing classes mostly removed; endpoint/social vectors remain).
- **Breach _impact_ & blast radius:** **reduced sharply** — a single foothold stops auto-escalating into a full compromise.
- **Realistic overall posture** for a well-built Symbolon-first app: materially stronger, **because single-point compromise stops being sufficient** — not because any number approaches 100%. A half-enforced deployment regresses toward the baseline.

> The strongest defensible claim: **Symbolon turns most "one stolen credential → full breach" events into "one stolen credential → a spent, scoped, minutes-old ticket."**

---

## 📊 Status

| Component | Version | Status |
|---|---|---|
| Protocol spec | v0.1 | **Draft.** Breaking changes expected before v1.0. |
| JSON Schema | v1 | Draft, tracks spec v0.1. |
| Test vectors | v0.1 | Draft, placeholder signatures. |
| Reference verifier (Go) | — | Not yet started. |
| Reference vault (iOS / Android) | — | Not yet started. |

<table>
<tr>
<td valign="top">

**✅ In the spec today**
- Human login lease (fresh proof)
- Offline Ed25519 / JCS verification
- One-time JTI · time window
- App & device binding
- §4 algorithm · §6 error codes

</td>
<td valign="top">

**◵ Designed · v1 roadmap**
- Mandatory origin / audience binding
- Cryptographic device + non-exportable session key
- Signed revocation feed (anti-rollback, fail-closed)
- Segmented, co-signed issuers
- Machine / DB / CI auth profile (TPM · SPIFFE · OIDC)
- AWS = Symbolon-gated STS broker

</td>
<td valign="top">

**❌ Not goals**
- A secrets manager / store
- A replacement for AWS IAM
- A full zero-trust mesh on day one
- Protecting SaaS you don't control
- Stopping malware already on the device at mint

</td>
</tr>
</table>

> **Residual risks we don't paper over:** endpoint malware present _at mint time_ can ride that window; the policy engine and any field-level encryption are _separate systems_; a lone invocable signing key stays catastrophic until co-signing ships. The win is structural — **single-point compromise becomes insufficient** — not a guarantee against everything.

---

## 📁 Repository

```
symbolon/
├── spec/
│   ├── symbolon-protocol-v0.1.md       ← normative protocol spec
│   └── symbolon-threat-model-v0.1.md   ← what it defends, what it doesn't
├── schema/
│   └── symbolon-lease-v1.schema.json   ← JSON Schema (Draft 2020-12)
├── test-vectors/
│   └── symbolon-test-vectors-v0.1.json ← signed lease fixtures
├── docs/
│   └── symbolon-v1-strategy.md         ← architecture, MVP & honest scorecard
└── showcase/
    └── index.html                      ← interactive showcase page
```

**Reading order:** [protocol spec](./spec/symbolon-protocol-v0.1.md) → [threat model](./spec/symbolon-threat-model-v0.1.md) → [JSON Schema](./schema/symbolon-lease-v1.schema.json) → [v1 strategy](./docs/symbolon-v1-strategy.md).

---

## 🗺 MVP (Minimal Strong Version)

The first buildable kit — and the only thing to build first:

1. **Login lease** (proof-minted)
2. **DPoP / session-bound non-exportable key**
3. **Sensitive-action lease**
4. **AWS STS broker lease** (verifies lease → `AssumeRole` → temp creds; **no AWS key stored**)
5. **Signed revocation snapshot**
6. **Segmented issuer keys** + multi-signature leases

_Deferred: DB/data mesh, full service mesh, field encryption, MPC, exhaustive provider coverage._ See [`docs/symbolon-v1-strategy.md`](./docs/symbolon-v1-strategy.md).

---

## 📄 License & contributing

License **TBD** before public release (Apache-2.0 for the spec is the leading candidate). This project is **pre-release** — the spec is being stabilised internally first, and external contributions are not yet accepted.

<div align="center">

<br>

**⬡ Symbolon** — _the broken token, made cryptographic._

</div>
