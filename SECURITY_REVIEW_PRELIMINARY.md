# Preliminary Security Review — Wallet V5

Repository reviewed: `junayedsakib04-byte/wallet-contract-v5`
Review scope: source-level, manual first-pass review of `contracts/wallet_v5.fc`, `Specification.md`, package scripts, and CI configuration.
Status: **Preliminary review only — not a formal audit and not a deployment approval.**

## Summary

The repository contains a TON Wallet V5 implementation with signature-based requests, extension-authenticated requests, and pre-composed outgoing actions. The README documents known operational and design limitations. No production-safety conclusion can be made from this source-only pass.

## Confirmed design constraints documented by the project

1. Wallet IDs must differ between networks because TON does not natively prevent replaying messages from testnet to mainnet.
2. `valid_until` is a 32-bit value and the project documents that it will stop working after 2106.
3. The project documents a known edge case involving invalid `action_send_msg` content with send mode `+2`.
4. The project notes that message and contract-data parsing do not consistently call `end_parse()`.

## Code paths that require focused adversarial testing

### 1. Pre-composed C5 action validation

`verify_c5_actions` accepts only `action_send_msg` entries, checks the encoded send mode and references, and caps the number of actions at 255. This is a high-impact trust boundary because accepted actions execute with wallet authority. Tests should cover malformed linked lists, exotic cells, truncated cells, unexpected references, all send-mode bit combinations, and message bodies with invalid TL-B structure.

### 2. Extension authentication and lifecycle

`recv_internal` authenticates an extension using the sender workchain and address hash, then checks membership in the stored extension dictionary. `process_actions` permits extension add/remove operations and allows extensions to change signature mode. Tests should cover unauthorized senders, wrong workchains, self-removal, removing the final extension while signatures are disabled, duplicate add/remove attempts, and sequences that combine extension changes with outgoing actions.

### 3. Signature and replay protection

`process_signed_request` checks the signature, signature-enabled state, seqno, wallet ID, and expiry before incrementing seqno. External requests commit seqno before processing actions. Tests should verify boundary values, expired requests, replay attempts, invalid signatures on external versus internal delivery, and action-phase failures after the external commit.

### 4. Persistent storage parsing

The contract reads a fixed layout containing signature mode, seqno, wallet ID, public key, and an extension dictionary. The documented absence of consistent end-of-slice checks means malformed or trailing storage/message data deserves explicit tests. Do not change parsing behavior without compatibility analysis.

## Recommended next steps

- Run the complete existing test suite and build in a reproducible environment.
- Add property-based/fuzz tests for C5 action lists and malformed cells.
- Add adversarial extension lifecycle and signature-mode transition tests.
- Compare this fork's contract and tests against the upstream TON implementation at the exact intended base revision.
- Have an independent TON smart-contract auditor review the code and resulting test evidence before deployment or handling real assets.

## Important limitation

This report records review scope and test priorities. It does **not** claim that a vulnerability was exploited, that all vulnerabilities were found, or that the contract is secure. No contract logic was modified by this report.
