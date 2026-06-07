# Symbolon Threat Model — v0.1 (DRAFT)

**Companion to:** `symbolon-protocol-v0.1.md`
**Status:** Draft. Tracks spec v0.1.

This document enumerates the attacks Symbolon defends against, the
attacks it does NOT defend against, and the residual risks an
implementer must address at adjacent layers (TLS, session, browser,
host).

It exists so that adopters do not have to reverse-engineer the threat
model from the spec text and so that future spec changes can be
checked against an explicit list of properties they MUST NOT regress.

---

## 1. Assets

| Asset | Held by | Compromise impact |
|---|---|---|
| Vault long-lived signing key | Vault hardware key store | Catastrophic — attacker mints arbitrary leases. Mitigated by hardware-backed storage (§5.1 of spec). |
| Subject's local user proof (biometric / PIN / platform passkey) | Subject's body / Vault device | High — combined with device theft, allows lease minting. |
| Ephemeral lease keypair | Subject's user-agent for one lease window | Low — single-use, expires in minutes. |
| Verifier's trusted-vault config | Verifier service | High — swapping vault pubkey lets attacker forge leases. Protected as normal config-as-code. |
| Consumed-JTI store | Verifier service | Medium — store loss enables replay until original `not_after` passes. |

## 2. Adversaries

| Adversary | Capabilities assumed |
|---|---|
| **A1 — Network attacker** | Passive + active on the network. TLS holds. |
| **A2 — Phisher** | Can stand up a lookalike Verifier and lure the user. |
| **A3 — Malware on Subject user-agent** | Can read/write process memory of the user-agent process, but cannot access the Vault device's secure enclave. |
| **A4 — Thief (device theft)** | Has physical possession of either the user-agent device or the Vault device. Does NOT have the user's biometric/PIN. |
| **A5 — Coerced-user attacker** | Has biometric/PIN by force or deception. |
| **A6 — Verifier breach** | Has read/write on the Verifier's database. |
| **A7 — Insider at Vault vendor** | Can ship malicious Vault updates. |
| **A8 — Nation-state with cryptanalysis** | Ed25519 collisions / preimage attacks. |

## 3. Defended attacks

For each attack, the mitigation reference is the spec section.

| Attack | Defended? | Mechanism |
|---|---|---|
| Stolen bearer token replay | YES | One-time JTI consumed atomically (§4 step 9, §4.1). |
| Bearer token used past expiry | YES | `not_after` enforced server-side (§4 step 8). |
| Bearer token used from different device | YES | `device_id` binding (§4 step 11). |
| Bearer token used against a different app | YES | `app_id` binding (§4 step 7). |
| Lease tampering (changing subject, actions, expiry) | YES | Ed25519 over JCS canonical form covers all signed fields (§3.4). |
| Lease replay across Verifier reboot | YES | Consumed-JTI store is durable; loss is detectable and recoverable (§4.1). |
| Lease minted without user consent | YES (assuming Vault conformant) | Vault §5.2 requires fresh local user proof per lease; cached proofs MUST NOT be reused. |
| Persistent passive credential theft (password DB dump) | YES | No long-lived bearer credential exists in the Verifier. |
| Phishing user into entering a credential | PARTIAL | No password to phish. Phisher can still relay an active lease — see §4 below. |
| Verifier breach exposes session material | YES | Verifier holds only consumed-JTI records and short-lived session tokens. |
| Clock skew abuse (slow verifier honors expired lease) | YES | Skew capped at 30s; verifier MUST fail closed if NTP unreachable (§4.2, §9.3). |

## 4. Partially defended attacks

| Attack | Status | Notes |
|---|---|---|
| Real-time phishing relay (attacker proxies a live lease) | PARTIAL | `device_id` binding + `geo_constraint` (§3.3) raise the bar. A determined relay from the victim's own device/IP can still succeed. Mitigation requires a Vault that pins the Verifier's identity (e.g. URL/origin attestation) — recommended but not normative in v0.1. |
| Malware on user-agent during active session | NOT in scope of lease layer | Use DPoP token binding, transaction signing for high-risk actions (Layer 7 / Layer 8 of the Fortress plan). |
| Coerced user | PARTIAL | Panic-lock (§10.3) lets the user revoke after the fact. Duress PIN (a second PIN that silently locks the Vault) is an implementation extension, not mandated. |

## 5. Out-of-scope attacks

These MUST be addressed at adjacent layers; Symbolon does not claim
defence.

| Attack | Adjacent layer |
|---|---|
| TLS downgrade / cert mis-issuance | TLS, CT logs, HSTS, key pinning. |
| Session hijack after a valid login | DPoP (RFC 9449), mTLS, secure cookie attributes, browser HttpOnly. |
| Cross-site request forgery on Verifier | CSRF tokens, SameSite cookies — Verifier-side. |
| XSS exfiltrating session token from browser | CSP, Trusted Types, output encoding — Verifier-side. |
| Stolen ephemeral private key from a compromised user-agent | Limited blast radius (one lease, minutes). Use a hardware-backed user-agent key store where possible. |
| Compromise of the Vault vendor's update pipeline (A7) | Reproducible builds, signed firmware, separate update key from signing key. |
| Cryptographic break of Ed25519 (A8) | Crypto agility — `extensions.sigalg` reserved for PQC (§9.2). |

## 6. Required adjacent-layer properties

A Verifier deployment that uses Symbolon and claims security MUST also
satisfy these properties. Symbolon's guarantees are void without them.

1. **TLS 1.3** in front of all Verifier endpoints; HSTS preloaded.
2. **Trusted clock source** (NTP with authenticated peer or NTS).
3. **Atomic, durable consumed-JTI store** (§4.1).
4. **Session-token binding** — once a lease is consumed, the session
   cookie/token issued by the Verifier MUST be bound to the ephemeral
   public key (via DPoP) or to a client TLS cert. Otherwise a stolen
   session token defeats the whole protocol.
5. **Per-action transaction signing for destructive operations** —
   `allowed_actions` is advisory until the Verifier rechecks at the
   action boundary (spec §10.1).
6. **Audit log of consumed leases** — for forensic reconstruction.

## 7. Invariants the spec MUST preserve across versions

Future spec changes MUST NOT regress any of the following:

| ID | Invariant |
|---|---|
| I-1 | Verifier never needs to call a central server to verify a lease (offline-first). |
| I-2 | No long-lived bearer credential ever traverses the wire to the Verifier. |
| I-3 | Lease consumption is atomic and durable. |
| I-4 | Vault cannot mint without a fresh local user proof. |
| I-5 | Verifier fails closed on unknown signature algorithm, unknown vault, clock untrusted, or replay-store unavailable. |
| I-6 | Lease object is canonical (JCS) — re-serialization MUST NOT alter signature validity. |

If any future change would break one of these, it is a v2 change, not
a v1.x change.

## 8. Comparison to neighbouring designs

| Concern | Password + 2FA | WebAuthn / passkeys | OAuth2 access token | Symbolon lease |
|---|---|---|---|---|
| Long-lived secret at server | YES (hash) | NO | YES (refresh) | NO |
| Phish-resistant | NO | YES | NO | YES (with origin pinning) |
| One-time use | NO | NO (passkey reusable) | NO | YES |
| Time-boxed | session only | session only | minutes-to-days | minutes |
| Action-bound | NO | NO | scopes (coarse) | YES (fine) |
| Device-bound | NO | per-credential | NO (bearer) | YES |
| Offline verifiable | YES | YES | usually NO (introspection) | YES |
| Independent of identity provider | n/a | NO (relies on RP) | NO | YES (Vault is separable) |

## 9. Known weaknesses being tracked for v1.0

| ID | Issue | Plan |
|---|---|---|
| W-1 | Subject ID linkability across Verifiers | Pairwise subject IDs (spec §13 open question 5). |
| W-2 | No mandatory Verifier-origin pinning in the lease | Add optional `verifier_origin` field bound by signature. |
| W-3 | No recovery story for total Vault loss | Add appendix on quorum recovery (open question 4). |
| W-4 | Software-only vaults are allowed but only labeled, not blocked | Add a Verifier policy field `min_key_backing` for v1.0. |
| W-5 | Coercion via duress PIN is implementation-defined | Consider normative duress-PIN guidance in v1.1. |
