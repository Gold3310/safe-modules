# Safe Passkey Module v0.2.1 — Initial Security Audit Worklog

**Status:** Preliminary static review; dynamic verification not yet executed  
**Research branch:** `security-audit/passkey-v0.2.1-2026-11-07`  
**Audited repository:** `safe-fndn/safe-modules` (maintained in this fork)  
**Baseline tag:** `passkey/v0.2.1`  
**Baseline commit:** `b6bd0d0ad45b213fe24534cc079d4a0731de2ed8`  
**Scope correction:** Safe Smart Account v1.5.0 is a separate product/version. The Passkey Module release in the Safe bug bounty scope is v0.2.1.

## 1. Rules of engagement

- Read-only review of the Safe upstream repository.
- Write work product only to the researcher's fork and this isolated branch.
- No changes to Safe upstream, production deployments, live wallets, real user funds, or production transactions.
- Do not publish suspected vulnerabilities or submit a bounty report until a finding is reproduced and its impact is established.
- Treat all observations below as hypotheses or implementation notes unless explicitly marked as confirmed.

## 2. Source files inspected

- `modules/passkey/contracts/libraries/WebAuthn.sol`
- `modules/passkey/contracts/libraries/P256.sol`
- `modules/passkey/contracts/SafeWebAuthnSignerFactory.sol`
- `modules/passkey/contracts/SafeWebAuthnSignerProxy.sol`
- `modules/passkey/contracts/SafeWebAuthnSignerSingleton.sol`
- `modules/passkey/contracts/4337/SafeWebAuthnSharedSigner.sol`
- `modules/passkey/contracts/base/SignatureValidator.sol`
- `modules/passkey/contracts/verifiers/FCLP256Verifier.sol`
- Relevant factory, P256, WebAuthn shim, Safe user-story, and shared-signer test files.
- `modules/passkey/package.json`, `hardhat.config.ts`, and `docs/v0.2.1/audit.md`.

## 3. Preliminary observations — not vulnerability findings

### OBS-01 — Short authenticator data reverts

**Location:** `modules/passkey/contracts/libraries/WebAuthn.sol`, `checkAuthenticatorFlags(bytes, AuthenticatorFlags)`.

The function reads `authenticatorData[32]` without an explicit length guard. A byte array shorter than 33 bytes will revert on the calldata bounds check.

**Current assessment:** No bounty impact established. The caller supplies the signature; malformed signature data is expected to fail validation. This is not currently classified as denial of service or a security vulnerability.

**Follow-up:** Add isolated tests for lengths 0, 1, 31, 32, and 33. Determine whether a revert changes any externally observable Safe execution/security property compared with a normal invalid-signature result. Do not claim impact solely from the revert.

### OBS-02 — No contract-level fixed origin/RP-ID allowlist

**Location:** `modules/passkey/contracts/libraries/WebAuthn.sol`, `encodeClientDataJson` and `encodeSigningMessage`.

The library reconstructs client data using the supplied additional fields and binds the resulting client-data hash into the P-256 signed message. The contract does not compare the origin or RP ID against one fixed expected value.

**Current assessment:** Not a confirmed vulnerability. This module is a generic on-chain signature verifier and does not define one universal relying-party origin. The origin is part of the signed WebAuthn client data; tampering with it without a new valid credential signature invalidates verification. Browser/authenticator RP-ID and origin policy, including subdomain and related-origin configuration, must be evaluated in the actual product integration.

**Follow-up:** Establish the intended RP-ID/origin policy for each supported integration. Test whether an attacker can obtain an accepted assertion from an origin that the integration explicitly considers unauthorized, without compromising the credential or relying-party configuration. Do not report the absence of a hard-coded origin check in isolation.

### OBS-03 — WebAuthn path uses malleability-permitting P-256 verification

**Location:** `modules/passkey/contracts/libraries/WebAuthn.sol`, `verifySignature(...)`; `modules/passkey/contracts/libraries/P256.sol`.

The WebAuthn path calls `verifySignatureAllowMalleability`, rather than the low-S enforcing `verifySignature` overload.

**Current assessment:** Design question, not a finding. WebAuthn ES256 verification semantics and Safe transaction replay/nonce handling must be considered before assigning impact. No unauthorized execution or replay has been demonstrated.

**Follow-up:** Test high-S and low-S variants against the same Safe transaction hash and nonce. Determine whether any downstream component relies on signature-byte uniqueness or whether the Safe nonce fully prevents replay.


### OBS-04 — Formal-verification source artifacts are not included

**Location:** `modules/passkey/docs/v0.2.1/audit-report-certora.pdf`.

The tagged source tree includes a Certora audit PDF but does not include the claimed `certora/munged` source, harness, or specification paths. The earlier worklog statement asserting a production-versus-munged parser discrepancy was unsupported and is withdrawn.

**Current assessment:** No model/source divergence has been established. The Certora report's proof assumptions and covered properties cannot be independently reconstructed from the tagged source tree alone.

**Follow-up:** Review the PDF report and obtain the corresponding Certora specifications from the audit provider if deeper proof-coverage validation is required. Do not treat the withdrawn discrepancy as a vulnerability or reportable finding.

## 4. Preliminary control review

| Control | Static observation | Status |
|---|---|---|
| Challenge binding | Challenge is reconstructed into clientDataJSON and hashed into the signed message | No bypass demonstrated |
| Authenticator flags | Required user-verification flag is checked | No bypass demonstrated |
| P-256 verifier return data | Exact 32-byte return and exact value 1 are required | No bypass demonstrated |
| Factory deterministic address | CREATE2 init code includes singleton, public key and verifier configuration | No bypass demonstrated |
| Proxy configuration | Singleton, public-key coordinates and verifier configuration are immutable | No bypass demonstrated |
| Shared signer configuration | Configuration is stored in Safe storage and configure is delegatecall-gated | No bypass demonstrated |
| Signature length | ABI signature length is bounded to constrain excess padding | Needs adversarial boundary tests |

## 5. Dynamic test plan

Run from the repository root after installing the pinned workspace dependencies:

```bash
pnpm install
pnpm --filter @safe-global/safe-passkey test
pnpm --filter @safe-global/safe-passkey run test:4337
```

If workspace filtering is unsupported by the repository's pinned pnpm version, run the package scripts from `modules/passkey` using the repository's documented commands.

Targeted test matrix:

1. Authenticator-data lengths 0, 1, 31, 32, 33, and standard WebAuthn length.
2. Missing UP and UV flags; present UP and UV flags.
3. Correct challenge, wrong challenge, malformed challenge fields, and altered origin field.
4. Valid P-256 signature; invalid signature; high-S and low-S variants.
5. Precompile success, precompile unavailable, fallback success, fallback revert, malformed return data, and invalid return value.
6. CREATE2 signer address consistency across all constructor parameters.
7. Direct CALL versus Safe DELEGATECALL to shared-signer configuration.
8. Shared-signer configuration isolation across two Safe storage contexts.
9. ERC-1271 validation and Safe transaction execution with valid, invalid, malformed, and replayed signatures.
10. ERC-4337 validation paths and storage-access restrictions.

All tests must run against local Hardhat/Anvil fixtures. Do not use production RPC signing or submit live transactions.

## 6. Current result

**Confirmed vulnerabilities:** None at this stage.  
**Critical/high/medium/low findings:** None established.  
**Dynamic reproduction:** Not yet run in this environment.  
**Bounty submission:** Not ready.

The previous static observations are not to be presented as bounty findings without demonstrated security impact.

## 7. Audit provenance

- Exact source baseline: `b6bd0d0ad45b213fe24534cc079d4a0731de2ed8`.
- The user's fork main branch was verified to contain this baseline in its history and to be 40 commits ahead of it.
- The upstream Passkey v0.2.1 audit history records the Hats Finance competition and Certora review. The Nethermind report is also documented in the maintained v0.2.1 audit history.
- The v0.2.1-1 tag points to `dfd3b05966e727dbb7a2fdeef52e4b230f63304e`; PR #486 describes verification tooling/compiler-input fixes, not a Passkey Solidity logic patch.

## 8. Next gate

Before any remediation commit or bounty submission:
1. Obtain and run the exact source/dependency tree in an isolated environment.
2. Execute the existing test suite.
3. Add minimal regression tests for each plausible issue.
4. Reproduce a concrete unauthorized security impact.
5. Verify the impact against the current supported release and bounty scope.
6. Only then prepare a narrowly scoped patch and private disclosure report.


## 9. Verified release comparison

Compared the following production Solidity files at `passkey/v0.2.1` and `passkey/v0.2.1-1`. Their Git blob SHAs are identical across both tags:

| File | Blob SHA |
|---|---|
| `contracts/libraries/WebAuthn.sol` | `c0efe5f99a149c663f8267f21a92c56263ac599c` |
| `contracts/libraries/P256.sol` | `882cb1e6b600b654a3f177a06dd6c28a87503a8d` |
| `contracts/SafeWebAuthnSignerFactory.sol` | `90363359840989d01c1a2689f859cf81b197435e` |
| `contracts/4337/SafeWebAuthnSharedSigner.sol` | `f73b6eb037f1ef9c5c1f98c348f515b04c216749` |

This verifies that these four core Solidity files did not change between the two tags. PR #486's diff contains verification/build tooling changes rather than changes to these production contracts. The release label alone must not be interpreted as a Solidity logic fix.

