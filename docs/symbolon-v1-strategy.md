# Symbolon v1 — Strategy, Architecture & Honest Scorecard

**Status:** Strategy draft (2026-06-05). Drives, but does not yet modify, the spec,
schema, test vectors, or showcase. When approved, it dictates the v0.1 → v1.0 changes.
**Audience:** the protocol author + future implementers, before any code is written.

---

## 1. The thesis (and the one claim worth making)

> **Symbolon does not make compromise impossible. It makes single-point compromise
> insufficient.**

Symbolon v1 is **a local-verification authorization protocol** that binds *identity
proof, device/session key, origin, action, resource, time, and issuer epoch* into a
short-lived signed **lease**. It is the **authorization gate**, not the credential store.

It is **not**:
- a secrets manager (it does not store downstream secrets),
- a replacement for AWS IAM (AWS will not verify a lease),
- a full zero-trust mesh on day one.

For **first-party** systems (your auth, session, API, sensitive actions, DB gate) Symbolon
sits directly in the path and gives proof-bound, multi-layer authorization. For
**third-party** systems (AWS, SaaS) it sits *in front of* HashiCorp Vault / AWS STS / a
credential broker, so issuing a credential requires a valid, short-lived, action-scoped
lease — it never replaces the third party's native credential.

---

## 2. Hard constraints — the design contract

You cannot have zero complexity, zero latency, zero single-point-of-failure, and zero
disaster scenario *at the same time*. Security trades between them. The realistic,
achievable goal:

1. **No single ordinary compromise grants sensitive access.**
2. **No long-lived secret is enough by itself.**
3. **No central service is required in the hot verification path.**
4. **Failure degrades closed for sensitive actions, but basic app availability continues.**

Every design choice below serves these four and nothing more ambitious.

---

## 3. Two planes, one cardinal rule

```
CONTROL PLANE  (may be online; not in the request hot path)
  Symbolon issuers / Vaults      — mint + sign leases after proof
  Policy config (signed)         — what is allowed
  Revocation feed (signed)       — kill switch
  Audit log (hash-chained)       — forensics
  Enrollment                     — pin issuer keys, register devices/workloads

DATA PLANE  (local verification only)
  App verifier · Session verifier · API verifier
  AWS-broker verifier · DB / data verifier
```

> **Cardinal rule: minting may depend on the Vaults. Verification must not.**

Login and action issuance may involve an issuer over the network. Every *protected
service* verifies leases **locally** against cached public keys, a policy snapshot, and a
revocation snapshot. This keeps the issuer out of the hot path (preserves the offline-
verify invariant) and bounds blast radius and latency.

---

## 4. The v1 lease (what becomes signed)

The v0.1 lease gains the fields that close the security gaps. **All are inside the signed
body.** This is a **wire-breaking** change set, so it MUST land *before the v1.0 freeze*
(after freeze it would be a v2 change per the threat-model invariant rule).

New / changed signed fields:

| Field | Purpose | Closes |
|---|---|---|
| `issued_at` | revocation floor + freshness; fixes the §10.3 bug (it referenced a field that didn't exist) | Gap 3 |
| `aud` | audience: the service the lease is *for* | Gap 2 |
| `origin` | the web origin the lease is bound to | Gap 2 |
| `resource` | the specific object (`customer:123`, `arn:…:bucket-x/*`) | Gap 2 |
| `action` | the single intent (`export_customer`, `aws:s3:GetObject`) | Gap 2 |
| `proof_level` | level of assurance of the proof (`user_verified`, `tpm_attested`, …) | Gap 2 |
| `key_epoch` | issuer key generation; old epochs are rejected | Gap 3 |
| `required_signatures[]` + a **signature array** | multi-issuer co-signing (replaces the single `vault_signature`) | Gap 4 |

Repo invariants this change set MUST respect (from `CLAUDE.md`):
- **Error codes are append-only.** Add new ones, never renumber:
  `origin_mismatch`, `audience_mismatch`, `resource_mismatch`, `action_mismatch`,
  `proof_level_insufficient`, `revocation_stale`, `key_epoch_retired`,
  `signature_threshold_unmet`, `challenge_pending_expired`.
- **§4 step order stays normative** and Appendix A pseudocode, the schema
  (`required` / `additionalProperties`), and each test vector's `exercises` move in
  lockstep.
- **Schema tightening:** require `not_after > not_before`, cap lease lifetime (normative
  verifier policy), constrain `allowed_actions`, and define extension criticality
  (unknown *critical* extension ⇒ reject; unknown non-critical ⇒ ignore).

---

## 5. The six gaps and their fixes

### Gap 1 — Ephemeral / session key location
A browser-JS key is stealable by XSS/malware, which defeats the entire session-binding
story. **v1 rule: the session key MUST be non-exportable when the platform supports it.**
Preference order:

1. **Best** — WebAuthn / passkey credential
2. **Good** — platform secure element (Secure Enclave / TPM / StrongBox / Keychain)
3. **Acceptable** — WebCrypto non-exportable key
4. **Weak (demo-only)** — plain JS in-memory key

Implementation notes (load-bearing):
- WebAuthn keys are **ES256 (P-256)**, not Ed25519, and WebAuthn signs its own assertion
  structure (`clientDataJSON` + `authenticatorData`), not a raw nonce. The possession
  proof MUST accept the WebAuthn assertion format and the spec's `ecdsa-p256` agility.
  **The session-binding key and the issuer signing key are deliberately different
  algorithms.**
- Flow: `login lease → client creates non-exportable key → session bound to that key`.
  A stolen cookie alone then opens nothing.

### Gap 2 — Origin / audience binding (mandatory)
Phishing-relay (AiTM) resistance requires binding, and binding must be **mandatory** for
browser/login flows. The verifier rejects on wrong `aud`, `origin`, `resource`, `action`,
`proof_level`, time, or device/session key.

Sharpening: **origin binding is only as strong as the source of the origin value.**
Comparing the lease to an HTTP header the relay controls is weak. Bind against the
**WebAuthn `clientDataJSON` origin** (set by the browser, not the page) — that is what
actually defeats a real-time proxy.

### Gap 3 — Revocation without a hot-path dependency
Verifiers do **not** call the issuer per request. The issuer publishes **signed revocation
snapshots**:

```
snapshot = sign({
  seq:            <monotonic counter>,     // anti-rollback
  key_epoch:      <current>,
  revoked_key_ids:[...],
  min_issued_at:  <timestamp>,
  disabled_subjects/devices: [...],
  expires_at:     now + 30–60s
})
```

Verifier behaviour (this is how constraint #4 is honoured):
- **Fresh snapshot** → verify normally.
- **Stale snapshot** → allow low-risk reads briefly; **block sensitive actions**.
- **Key revoked / epoch retired / `issued_at < min_issued_at`** → reject.

Sharpening: each snapshot carries a **monotonic `seq`**; verifiers **never accept a lower
`seq` than already seen**, or an attacker serves yesterday's validly-signed snapshot to
suppress a revocation.

### Gap 4 — No single point of failure
Do not use one global signing key. **Segment issuers:** user-login, admin-action,
machine/workload, AWS-broker, recovery. High-risk issuers require **co-signing**.

For the MVP, **do not build MPC.** Use **multiple independent signatures on the lease** —
the verifier checks *k of n* signatures against *n* pinned keys:

```
"required_signatures": ["user_vault", "org_hsm"]   // 2-of-3 example
```

This is simple (N Ed25519 checks), and no single signer compromise is a total disaster.
Threshold/MPC is a later optimization, not a v1 dependency.

### Gap 5 — Enrollment (explicit, never TOFU)
- **App verifier:** an admin installs trusted issuer public keys via **signed config**;
  the verifier **pins** key ids + issuer ids. *Do not silently trust a discovery URL as
  authority* (this replaces the §7 TOFU risk).
- **User device:** enroll via passkey/WebAuthn; the Vault key is linked to the account;
  the app stores only the public key / issuer id.
- **Workload:** enroll via cloud identity / TPM / CI OIDC; Symbolon stores the attested
  workload identity.

### Gap 6 — Latency
Correctly designed, normal verification adds **no network latency**:

```
hot path:  request carries lease + session proof
           → verify signature(s) locally
           → check local revocation snapshot
           → check local policy snapshot
           → allow / deny
```

Latency occurs **only at boundary mint events** — login, sensitive action, AWS credential
request, admin step-up. Those are not per-request, so it is acceptable.

---

## 6. Replay: the explicit state machine

The v0.1 algorithm was ambiguous (step 9 "atomic check-and-insert" vs step 13 "mark
consumed"). v1 makes it explicit:

```
unseen → reserve(jti)   [atomic; concurrent reservers cannot both win]
       → challenge       [while reserved; per-jti rate-limited; short reservation TTL]
       → commit(jti)      [durable; gates session creation]
on challenge failure or timeout → release reservation
```

Resolution of the release-vs-consume tradeoff: **release on failure**, but bound total
exposure with a short reservation TTL, a per-`jti` challenge rate-limit, and the lease's
own `not_after`. Final `commit` — not the initial reserve — is what authorizes the
session.

---

## 7. Minimal Strong Version (the only thing to build first)

1. **Login lease** (proof-minted)
2. **DPoP / session-bound non-exportable key** (Gap 1)
3. **Sensitive-action lease** (`action` + `resource` + `proof_level`)
4. **AWS STS broker lease** (broker verifies lease → `AssumeRole` → temp creds; **no AWS
   key stored**)
5. **Signed revocation snapshot** (Gap 3)
6. **Segmented issuer keys** + multi-signature leases (Gap 4)

**Explicitly deferred** (high effort, partly orthogonal, diminishing compound return):
DB/data mesh, full service mesh, field-level encryption, MPC/threshold signing, exhaustive
auth-provider coverage.

POCs that demonstrate the property:
- **Custom app protection** — login, session binding, sensitive-action lease.
- **Server-compromise demo** — attacker gets a shell but cannot export data without a
  fresh action lease.
- **AWS access demo** — app must obtain a lease; broker verifies it, then mints temp STS
  creds. No AWS key in `.env`.

---

## 8. The compound-security property

| Attacker has | Still missing |
|---|---|
| App-server shell | Valid Symbolon workload/action lease |
| Stolen Vault/broker token | Symbolon lease + policy approval |
| Stolen AWS temp credential | Very short TTL, narrow IAM scope |
| Stolen session cookie | Non-exportable DPoP/session private key |
| DB dump | Data-gate / decryption lease (if built) |
| One issuer key | Limited to that one domain (segmented) |
| Old lease | Blocked by TTL / JTI / `key_epoch` |
| CI runner access | CI OIDC + policy + short TTL |

The design test for **any** future layer: *does it force the attacker to independently
hold one more thing?* If yes, it may be worth its complexity. If no, it is cost without
compound benefit — cut it.

---

## 9. Honest scorecard

These are **not** scientific guarantees. "Protection %" conflates three different effects
(likelihood reduction, blast-radius reduction, attacker-effort increase) into one number,
which invites overclaiming. Read the table as *direction + the corrected ceiling*, and
measure the delta against a **modern** baseline (passkeys + short-lived tokens + DPoP),
not a password+cookie app — against a strong baseline the gains are real but concentrated
in **action binding, offline verification, and blast-radius/TTL reduction**.

| Area | Normal app | Symbolon-first | Honest note |
|---|---|---|---|
| Password phishing | weak | very strong | No reusable secret; origin-bound lease |
| Stolen session cookie | weak | strong **iff** key is non-exportable | Else no better |
| API-key leak | weak | very strong | Short-lived scoped leases replace static keys |
| Server compromise | weak | medium–strong **iff** action/data gates built | Much of this win is the data-gate, not the lease |
| DB dump | weak–medium | strong **iff** field encryption + data gate | Credit belongs partly to encryption, not Symbolon |
| Insider/admin misuse | medium | strong | Step-up + quorum + audit + scoped leases |
| CI/CD secret theft | weak | strong | No long-lived CI secret; proof-bound release lease |
| AWS credential theft | medium | strong | Symbolon gates STS; AWS keeps native IAM |
| Endpoint malware (at mint time) | weak | **only modest** | Hard class; present-at-mint malware rides the window — do not overclaim |
| Single issuer-key compromise | very weak | **low until** multi-sig + working revocation | The 50–85% figure is earned only *after* Gap 3 + Gap 4 ship |

Realistic overall: a well-built Symbolon-first app is materially stronger because
**single-point compromise stops being sufficient** — not because any one number approaches
100%. A badly-implemented one (enforced at only some boundaries) regresses toward the
baseline.

---

## 10. Residual risks & open questions (do not pretend these are solved)

- **Endpoint malware present at mint** can approve the proof and use the freshly-issued
  key for that window. Symbolon limits *duration and scope*, not this.
- **The policy engine is a separate system** (OPA/Cedar-class). Symbolon carries intent;
  it does not evaluate org policy.
- **Field encryption / data gate is largely orthogonal** to the lease protocol — keep it
  in the architecture, but do not attribute its wins to Symbolon.
- **UX vs security:** fresh proof per sensitive action risks biometric fatigue →
  reflexive approval. Design the step-up cadence deliberately.
- **Threshold/MPC** is deferred; until it ships, segmentation + multi-sig is the
  single-point mitigation, not elimination.

---

## 11. What must be locked before any code

1. **Ephemeral/session key = non-exportable** (decides every cookie/malware claim).
2. **Mandatory origin/audience binding, bound to a trustworthy origin source**
   (decides every phishing-proxy claim).
3. **A working offline revocation + freshness model with anti-rollback**
   (decides every "contained compromise" claim).
4. **Scope discipline** — ship the six Minimal-Strong items; defer the mesh.

When these four are settled, the next artifact is an implementation plan for the Minimal
Strong Version (POC 1 first), and the corresponding v0.1 → v1.0 spec/schema/test-vector
changes — produced in lockstep per the repo's cross-reference rules.
