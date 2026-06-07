# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Symbolon is an **authentication protocol specification** — not running software. There
is no build, lint, or test toolchain. The repository is a set of normative documents,
a JSON Schema, and test-vector fixtures. Reference implementations (a Go verifier, iOS/
Android vaults) are planned but **not yet started** (see README "Status" table).

Core idea: login authority lives in an external **Vault** (a hardware-backed device),
not in the application. The Vault mints short-lived, single-use, signed **Login Leases**;
the application (**Verifier**) checks a lease offline against the Vault's published
Ed25519 public key before opening a session. Roles: **Subject** (user), **Vault**
(issuer), **Verifier** (app), **user-agent** (treated as a hostile carrier).

## File map and authority order

The spec is the single source of truth; everything else tracks it. When changing
protocol semantics, update in this order and keep them consistent:

1. `spec/symbolon-protocol-v0.1.md` — **normative spec.** Section numbers (§3 lease
   object, §4 verification algorithm, §6 error codes, §7 discovery, §8 transports) are
   referenced by every other file. This is the document to change first.
2. `spec/symbolon-threat-model-v0.1.md` — what is/isn't defended. Section 7 lists
   **invariants I-1…I-6** that future changes MUST NOT regress; any change that breaks
   one is a v2 change, not a v1.x change.
3. `schema/symbolon-lease-v1.schema.json` — JSON Schema (Draft 2020-12) for lease shape.
   Mirrors spec §3.1. `additionalProperties: false` plus the `required` array must match
   the spec's field table exactly.
4. `test-vectors/symbolon-test-vectors-v0.1.json` — fixtures keyed to spec sections via
   each vector's `exercises` field and to error codes via `expect_error`.

A "conformance" doc and a `reference/README.md` are referenced by the README but **do
not exist yet** — do not assume they are present.

## Invariants when editing

- **Cross-references are load-bearing.** Adding/removing/renumbering a spec section, a
  lease field, or an error code means updating the schema's `required`/`properties`, the
  test vectors' `exercises`/`expect_error`, and the threat-model references in lockstep.
- **Error codes are append-only.** Per spec §6, numeric codes (1001, 1100, 1200, …)
  MUST NOT be reused across versions. Add new codes; never repurpose old ones.
- **Verification step order is normative.** Spec §4 defines steps 1–14 in a required
  order (version → type → vault lookup → signature → app_id → time → replay → geo →
  device → challenge → consume → session). Appendix A pseudocode and the test-vector
  `exercises` references must agree with this order.
- **Fail-closed is mandatory.** Unknown vault, unknown sigalg, untrusted clock, and
  replay-store-unavailable all reject (invariant I-5). Don't introduce permissive
  fallbacks.
- **Canonicalization.** The signed form is JCS (RFC 8785) of the lease with
  `vault_signature` removed; signature is Ed25519, multibase base58btc (`z` prefix).
  Re-serialization must not change signature validity (invariant I-6).

## Test vectors

Signature/key fields are **placeholders** (`PLACEHOLDER_*`). Vectors V-002, V-008, V-009
exercise real signatures and cannot run end-to-end until the Go reference verifier ships
real Ed25519 fixtures. The other vectors (parse, time, replay, binding, geo, clock) are
exercisable without live crypto. Vectors carry `level: "MUST"` or `"SHOULD"` — only
`MUST` vectors are required for conformance (spec §11.3).

## Versioning convention

Wire version is a single integer (`v: 1`). The `v0.1` filename suffix is a draft
revision of the v1 spec; at freeze, files become `…-v1.0.md` with no further v1 wire
breaks. Keep the schema `$id`/`const` (`v == 1`) and the `type` discriminator
(`"symbolon.lease"`) aligned with the spec across renames.
