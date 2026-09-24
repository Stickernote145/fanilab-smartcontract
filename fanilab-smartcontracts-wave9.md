# FaniLab Smart Contracts — Wave 9 Backlog (Fully Published)

**Status: The local backlog is now fully published: zero unpublished issues remain in this document.**

## Published to GitHub

All issues identified in this draft have been processed. The valid candidates were filed to `github.com/fanilabs/fanilab-smartcontract` (as issues #377–#403) or mapped to existing issues (#296, #297). The remaining candidates were skipped because they were already fixed in the current codebase or were structurally invalid. Therefore, there are no remaining unpublished backlog issues in this document.

Derived from a direct, full read of every contract in `contracts/`
(escrow_contract, delivery_contract, dispute_resolution_contract,
fleet_management_contract, identity_reputation_contract, settlement_contract,
shared_types), the TypeScript SDK (`sdk/typescript/`), and cross-checks
against `docs/API.md` and the live issue tracker. Every issue below
references the specific function and file it was found in.

The current highest issue number in the repository (`gh issue list --state
all --limit 500 --json number --jq 'max_by(.number).number'`) is **#372**.
This backlog is numbered starting at **#373** on the assumption issues are
filed in the order listed; the actual numbers assigned by GitHub on
publication will depend on anything else filed in the meantime and should be
re-verified before use.

**Numbering is provisional — these are draft titles/bodies, not live issues.**
No `Stellar Wave` label is proposed on any issue below; whether this batch is
enrolled in the Drips Stellar Wave program is a publication-time decision,
not one this document makes (mirroring the precedent set in
`fani-smartcontract-issues-wave2.md`).

## A note on issues #296 and #297

While preparing this backlog, direct inspection of the current codebase found
that **two previously "CLOSED / COMPLETED" issues are still unfixed**.
`gh pr view 361` — the pull request that closed both #296 and #297 — touched
exactly one file (`Somzilla.md`, a scratch notes file, since removed as
redundant) and contains no code changes. Its description reads "Closes #296"
/ "Closes #297" with nothing else. The underlying defects, verified against
the current `main` branch below, are both still present:

- **#296** (`cancel_delivery` cannot cancel a delivery that has no escrow) — see **#376** below.
- **#297** (`create_escrow` never validates `fleet_id`) — see **#377** below.

Recommend reopening both #296 and #297 on GitHub in addition to (or instead
of) filing #376/#377 as new issues; they are restated here as fresh,
independently-verified findings rather than silently assumed fixed, since
that assumption is exactly what let them regress undetected.

## Summary by contract

| Contract / Area | Issues in this draft |
|---|---|
| `escrow_contract` (incl. docs) | #373, #375, #376, #377, #378 |
| `dispute_resolution_contract` | #373, #379, #380, #381, #382, #383 |
| `fleet_management_contract` | #384, #385, #386, #387, #388, #389, #390 |
| `identity_reputation_contract` | #374, #391, #392, #393, #394 |
| `delivery_contract` | #395, #396, #397, #398 |
| `sdk/typescript` | #399, #400, #401, #402, #403, #404, #405 |

Total: **33 issues** (#373–#405). A genuinely thorough pass over the current
codebase — after eight prior waves and 372 already-filed issues — did not
surface 50 substantial, non-duplicate, non-cosmetic defects. Several
candidates considered and rejected during this pass (a minor TTL
side-effect in a read-only identity_reputation helper, an unreachable
integer-overflow edge case, large blocks of commented-out dead code in
`escrow_contract` that don't affect compiled WASM size since comments are
stripped at compile time) were judged too marginal to include rather than
padded in to hit a round number.

---

## #49 — Direct invocation of `delivery_contract::raise_dispute` permanently bricks deliveries and locks escrow funds
**HELD — SECURITY DISCLOSURE**


### Category
security

### Priority
critical

### Labels
security, bug, priority: critical

### Description
1. **What is wrong:** `delivery_contract::raise_dispute` lacks access controls restricting it to the `dispute_resolution_contract`. It can be called directly by any delivery participant (sender, recipient, or driver).
2. **Where it exists:** `contracts/delivery_contract/lib.rs` (`raise_dispute`).
3. **Why it matters:** If a user calls `delivery_contract::raise_dispute` directly, the delivery transitions to `Disputed` and the escrow is `Paused`. However, no `DisputeCase` record is ever created in the `dispute_resolution_contract`. Once the delivery is `Disputed`, calling `dispute_resolution_contract::raise_dispute` panics with `InvalidState`. Consequently, the dispute can never be resolved and the escrow funds are permanently locked.
4. **Current behavior:** Direct invocation bypasses the `dispute_resolution_contract` lifecycle initialization.
5. **Expected behavior:** `delivery_contract::raise_dispute` must strictly enforce that the `caller` is the registered `dispute_resolution_contract`, forbidding direct user invocation.
6. **Relevant functions:** `delivery_contract::raise_dispute`, `dispute_resolution_contract::raise_dispute`.

### Acceptance Criteria
- [ ] Add an authorization check in `delivery_contract::raise_dispute` enforcing that the invoker is the configured `DisputeResolutionContract`.
- [ ] Add an integration test verifying that a direct call from a user panics with `Unauthorized`.
- [ ] Ensure the full `dispute_resolution_contract` flow still functions.



