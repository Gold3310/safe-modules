# Passkey Module — Hardhat Network Signer Configuration

**Baseline:** `passkey/v0.2.1` at `b6bd0d0ad45b213fe24534cc079d4a0731de2ed8`  
**Research branch:** `security-fix/passkey-hardhat-default-key-2026-11-07`

## Finding PF-01 — Public test mnemonic was assigned to remote networks

**Severity:** High deployment-key risk; requires a misconfigured or accidentally funded remote network.

**Affected files:**
- `modules/passkey/hardhat.config.ts`
- `modules/4337/hardhat.config.ts`
- `modules/allowances/hardhat.config.ts`
- `modules/recovery/hardhat.config.ts`

The original configuration used the well-known Hardhat test mnemonic `candy maple cake sugar pudding cream honey rich smooth crumble sweet treat` whenever neither `PK` nor `MNEMONIC` was set. The resulting signer configuration was spread into both the Sepolia network and the user-defined `CUSTOM_NODE_URL` network.

If `CUSTOM_NODE_URL` points to a live chain and a derived default account receives funds, anyone who knows the public Hardhat mnemonic can derive the same private key and spend those funds. The local Hardhat network does not need this fallback because Hardhat supplies local test accounts itself.

**Remediation on this branch:** Remote network accounts are configured only when an explicit `PK` or `MNEMONIC` is supplied (the Allowances module uses its existing `MNEMONIC` setting). No remote network receives a default mnemonic.

**Operational effect:** Deployments to Sepolia/custom networks without explicit credentials now have no configured signer and fail rather than silently selecting a publicly known account.

**Scope:** This is a deployment configuration issue, not a demonstrated vulnerability in the deployed Passkey Solidity contracts and not a bounty submission.

## Validation status

The code change is committed. No GitHub Actions run was available for this branch at the time this note was written. Verify the Passkey package build and tests before merging or deploying.
