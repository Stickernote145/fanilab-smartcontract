# FaniLab Smart Contracts — Wave 9 Backlog (DRAFT — NOT YET PUBLISHED)

**Status: local draft only. Nothing in this document has been filed to GitHub.
Zero issues have been created via `gh issue create`. This file exists purely
for review before publication.**

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

## #373 — Disputes raised through `dispute_resolution_contract` never populate `EscrowRecord.disputed_by`

**Labels:** `bug`

### Problem Statement

`escrow_contract::freeze_funds` (`contracts/escrow_contract/lib.rs:1790-1842`)
transitions an escrow from `Locked`/`Holdback` into `Paused` and sets
`record.disputed_at`, but never sets `record.disputed_by`. Only
`escrow_contract`'s own separate, direct `raise_dispute` entry point
(`lib.rs:1451-1482`) sets `disputed_by = Some(caller)`.

The protocol's actual, documented dispute flow is
`dispute_resolution_contract::raise_dispute`
(`contracts/dispute_resolution_contract/lib.rs:417-473`), which verifies the
caller, transitions the delivery, and then cross-calls `escrow_contract`'s
`freeze_funds` — **not** its `raise_dispute` — passing its own contract
address as `caller` (lib.rs:464-473). That call path never touches
`disputed_by`.

### Why It Matters

For every dispute raised through the intended flow — which is the only flow
documented and the only one with dispute-lifecycle tracking
(`DisputeCase.raised_by`, evidence, resolution) — the escrow record's
`disputed_by` field is silently `None` forever. `dispute_resolution_contract`
itself records the correct value in `DisputeCase.raised_by`, so the
information exists, just not where the escrow-side field promises it.

This is user-visible: `sdk/typescript/src/clients/escrow.client.ts:254`
exposes `disputedBy` on the decoded escrow record, and both `docs/API.md`
and `docs/contract-design/escrow-design.md` document
`EscrowRecord.disputed_by` as part of the escrow's public shape. Any
off-chain consumer reading escrow state directly (dashboards, audit tooling,
the SDK) rather than cross-referencing the dispute contract sees an empty
field for real disputes and a populated one only for the rarely-used direct
`escrow_contract::raise_dispute` path.

### Proposed Solution

Have `dispute_resolution_contract::raise_dispute` pass the real caller
through to `freeze_funds` so it can be recorded, or have `freeze_funds`
accept and store an explicit "raised by" address distinct from its own
authorization check (which must remain scoped to the configured dispute
contract). The simplest fix: add a `raised_by: Address` parameter to
`freeze_funds` and set `record.disputed_by = Some(raised_by)` on the
`Locked`/`Holdback` → `Paused` transition, leaving the already-Paused
no-op branch untouched.

### Acceptance Criteria

- [ ] A dispute raised via `dispute_resolution_contract::raise_dispute` results in `escrow_contract::get_escrow(delivery_id).disputed_by == Some(<the actual party who raised it>)`
- [ ] The direct `escrow_contract::raise_dispute` path is unaffected
- [ ] `freeze_funds`'s already-Paused no-op branch does not overwrite an existing `disputed_by`
- [ ] Regression test covers a dispute raised through the full `dispute_resolution_contract` flow, asserting on the escrow-side `disputed_by`

### Technical Notes

- `escrow_contract/test.rs:4133-4158` (`test_freeze_funds_is_noop_on_already_paused_escrow`) is the only test asserting on `disputed_by` after `freeze_funds`, but it pre-seeds the record via escrow's own direct `raise_dispute` first — it never exercises the dispute-resolution-contract-initiated path.
- `escrow_contract/test.rs:4165` (`test_post_delivery_dispute_end_to_end`) does exercise the real path via the dispute client but never asserts on `disputed_by` at all.
- Changing `freeze_funds`'s signature is a breaking change to a cross-contract call; `dispute_resolution_contract` is the only known caller, so both must be updated together.

### Relevant Files

- `contracts/escrow_contract/lib.rs` — `freeze_funds`
- `contracts/dispute_resolution_contract/lib.rs` — `raise_dispute`
- `sdk/typescript/src/clients/escrow.client.ts` — `disputedBy` decoding

### Testing Requirements

- Integration test: dispute raised via `dispute_resolution_contract::raise_dispute` → escrow's `disputed_by` is populated
- Regression test: direct `escrow_contract::raise_dispute` still sets `disputed_by`
- Regression test: freezing an already-Paused escrow a second time does not clear a previously-set `disputed_by`

### Definition of Done

- [ ] `disputed_by` reliably reflects who raised the dispute regardless of entry point
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

2–4 hours

### Dependencies

None.

---

## #374 — `identity_reputation_contract::award_reputation` uses unchecked addition instead of the saturating arithmetic every sibling function uses

**Labels:** `bug`

### Problem Statement

`increase_reputation` and `decrease_reputation` both route their score
updates through `reputation_up`/`reputation_down`
(`contracts/identity_reputation_contract/lib.rs:41-43`), which use
`saturating_add`/`saturating_sub`. `award_reputation`
(`lib.rs:428-454`) instead computes the new score directly:

```rust
profile.reputation_score = (profile.reputation_score + points).min(MAX_REPUTATION);
```

This is a plain `u32` addition, not `saturating_add`.

### Why It Matters

The workspace's release profile sets `overflow-checks = true`
(`Cargo.toml:15`), so an overflow here panics the transaction instead of
silently wrapping — but it still means `award_reputation` can hard-abort
where every other reputation-mutating function gracefully caps, violating
the codebase's own stated invariant ("saturating math to prevent overflow")
that `PRODUCTION_READINESS.md` lists as an implemented guarantee. The
function is a public entry point reachable by **any** address the admin has
authorized via `set_authorized_contract` (not just
`dispute_resolution_contract`, its sole caller today), so a future
authorized caller — or a misconfigured `points` value — that pushes the sum
past `u32::MAX` aborts instead of capping at `MAX_REPUTATION` as the
function's own doc comment promises ("The resulting score is still capped
at `MAX_REPUTATION`").

### Proposed Solution

Change `award_reputation` to use `reputation_up` (or an equivalent
`saturating_add` + `.min(MAX_REPUTATION)`), matching `increase_reputation`
and keeping all three reputation-mutating functions on one consistent,
overflow-safe code path.

### Acceptance Criteria

- [ ] `award_reputation` cannot panic from arithmetic overflow regardless of the `points` value supplied
- [ ] Behavior for all currently-passing values of `points` is unchanged
- [ ] Regression test calls `award_reputation` with a `points` value close to `u32::MAX` and asserts the score caps at `MAX_REPUTATION` instead of panicking

### Technical Notes

- `reputation_up` already exists and is unit-tested for `increase_reputation`'s use; reuse it rather than adding a second helper.
- The only current caller (`dispute_resolution_contract::resolve_dispute_pay_driver`, `contracts/dispute_resolution_contract/lib.rs:813-822`) passes the fixed constant `DISPUTE_REPUTATION_REWARD`, so this is currently unreachable in production — the fix is about closing the entry point for any future/other authorized caller, not an active exploit.

### Relevant Files

- `contracts/identity_reputation_contract/lib.rs` — `award_reputation`, `reputation_up`

### Testing Requirements

- Unit test: `award_reputation` with `points` near `u32::MAX` caps at `MAX_REPUTATION` without panicking
- Regression test: existing small-`points` behavior unchanged

### Definition of Done

- [ ] `award_reputation` uses saturating arithmetic
- [ ] Test above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

1–2 hours

### Dependencies

None.

---

## #375 — Escrow payout-routing is snapshotted at creation for `create_escrow` but always resolved live at settlement for `create_escrows_batch`

**Labels:** `bug`

### Problem Statement

`create_escrow` (`contracts/escrow_contract/lib.rs:916-1063`), when both a
fleet-management contract is configured and a `fleet_id` is supplied,
queries `get_payout_address` at creation time and stores the result under
`DataKey::EscrowPayoutAddress(delivery_id)` (lib.rs:949-959, 984-992). At
settlement, `payout_driver` (lib.rs:231-303) prefers this snapshot when
present (`payout_was_snapshotted`), and only falls back to a fresh,
settlement-time `get_payout_address` lookup when no snapshot exists.

`create_escrows_batch` (`lib.rs:1069-1240`) accepts a per-entry `fleet_id`
(`Option<u64>` in the `(u64, Address, i128, Option<u64>)` tuple) but **never
computes or stores an `EscrowPayoutAddress` snapshot for any batch entry** —
there is no equivalent of the `create_escrow` snapshot block anywhere in the
batch function.

### Why It Matters

This means two escrows created with an identical `fleet_id` and identical
fleet-membership state resolve payout routing through fundamentally
different mechanisms depending purely on whether they were created via
`create_escrow` or `create_escrows_batch`: single-created escrows lock in
the driver's fleet membership as of escrow creation, while batch-created
escrows always re-resolve membership live at settlement — potentially weeks
later, after the driver's fleet status may have changed. Nothing in the
public API, events, or documentation distinguishes which mode a given
escrow uses; the only way to tell is to check whether
`get_escrow_by_...` — there isn't even a getter for
`EscrowPayoutAddress` — makes this observable at all off-chain.

### Proposed Solution

Either make `create_escrows_batch` snapshot `EscrowPayoutAddress` per entry
exactly like `create_escrow` does (for consistency with the single-creation
path), or deliberately remove snapshotting from `create_escrow` so both
paths resolve live at settlement (for consistency with the batch path,
and to avoid the mutable-fleet-state routing problem tracked separately).
Whichever direction is chosen, document it and make the behavior
observable off-chain (e.g. expose whether a given escrow has a stored
payout-address snapshot).

### Acceptance Criteria

- [ ] `create_escrow` and `create_escrows_batch` resolve fleet payout routing identically for equivalent inputs
- [ ] The chosen behavior (snapshot-at-creation or resolve-at-settlement) is documented
- [ ] Regression test creates one escrow via each path with the same fleet_id/driver and asserts identical settlement routing behavior after a fleet-membership change

### Technical Notes

- This is related to but distinct from the already-known issue of routing being resolved from mutable fleet state at payout time — that issue concerns *whether* live resolution is desirable at all; this issue is that the two creation paths don't even agree on which mode they use.
- `EscrowPayoutAddress` has no public getter; adding one may be a reasonable side effect of the fix.

### Relevant Files

- `contracts/escrow_contract/lib.rs` — `create_escrow`, `create_escrows_batch`, `payout_driver`

### Testing Requirements

- Integration test: escrow created via `create_escrow` with `fleet_id` set, then driver's fleet membership changes before settlement → payout still routes per the original (snapshotted) fleet
- Integration test: escrow created via `create_escrows_batch` with the same setup → document/assert actual (live) routing behavior
- Regression test: both paths behave identically once fixed

### Definition of Done

- [ ] Both creation paths use one consistent payout-routing resolution strategy
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Medium**

### Estimated Effort

4–8 hours

### Dependencies

None; touches the same code area as #377.

---

## #376 — `cancel_delivery` still cannot cancel a delivery that has no escrow (regression of closed issue #296)

**Labels:** `bug`

### Problem Statement

`delivery_contract::cancel_delivery`
(`contracts/delivery_contract/lib.rs:456-499`) unconditionally cross-calls
`escrow_contract::refund_escrow` before updating its own state:

```rust
let _: () = env.invoke_contract(
    &escrow_address,
    &soroban_sdk::Symbol::new(&env, "refund_escrow"),
    soroban_sdk::vec![&env, sender.into_val(&env), u64::from(delivery_id).into_val(&env)],
);
delivery.status = DeliveryStatus::Cancelled;
```

`escrow_contract::refund_escrow` loads the escrow via `load_escrow`
(`contracts/escrow_contract/lib.rs:408-421`), which panics with
`EscrowError::DeliveryNotFound` when no escrow record exists for that
`delivery_id`. The panic propagates and reverts the whole cancellation.
Delivery creation and escrow creation are separate calls on separate
contracts, so a delivery with no escrow is an ordinary, reachable state.

This is the exact defect filed as **issue #296**, which GitHub currently
shows as `CLOSED` / `COMPLETED`. The pull request that closed it (`#361`)
touched only a scratch notes file and made no code changes — the fix never
landed. This issue is filed to restate the defect against current `main`
and flag #296 for reopening.

### Why It Matters

A sender who creates a delivery and never funds an escrow — changed their
mind, the funding transaction failed, or they simply never got to it — has
a delivery record they can never cancel. It remains `Pending` permanently,
occupying a `delivery_id` and appearing in the sender's and recipient's
indexes indefinitely, with no alternative exit path.

### Proposed Solution

Give `escrow_contract` a non-panicking existence check (`has_escrow(delivery_id) -> bool`,
or an `Option`-returning variant of `get_escrow`) so `cancel_delivery` can
skip the refund call when there is nothing to refund. Preserve the existing
ordering guarantee — the escrow call, when it does run, must still happen
before the delivery's local state mutates, so a genuine refund failure
still reverts the whole cancellation.

### Acceptance Criteria

- [ ] A delivery with no escrow can be cancelled
- [ ] A delivery with an escrow still triggers the refund before its state changes
- [ ] A genuine refund failure (e.g. insufficient contract balance) still reverts the whole cancellation
- [ ] Regression test covers cancellation both with and without an escrow
- [ ] Issue #296 reopened or explicitly superseded by this issue with a note explaining why

### Technical Notes

- `MockEscrowContract::refund_escrow` in `delivery_contract/test.rs:33-40` is a no-op stub that never panics regardless of whether the mocked escrow "exists" — the delivery-contract test suite cannot detect this defect on its own; any fix needs a test that exercises the real `escrow_contract`, or an updated mock that mirrors `load_escrow`'s panic behavior.
- No non-panicking escrow-existence check exists anywhere in `escrow_contract` today.

### Relevant Files

- `contracts/delivery_contract/lib.rs` — `cancel_delivery`
- `contracts/escrow_contract/lib.rs` — `refund_escrow`, `load_escrow`, `get_escrow`
- `contracts/delivery_contract/test.rs` — `MockEscrowContract`

### Testing Requirements

- Unit test: cancelling a delivery with no escrow succeeds and sets `Cancelled`
- Regression test: cancelling a delivery with an escrow still refunds the sender
- Regression test: a failing refund still reverts the cancellation

### Definition of Done

- [ ] Cancellation works without an escrow
- [ ] Refund ordering and rollback behavior preserved
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Medium**

### Estimated Effort

4–8 hours

### Dependencies

Restates closed issue #296; recommend reopening #296 rather than tracking both.

---

## #377 — `create_escrow` still never validates `fleet_id` (regression of closed issue #297)

**Labels:** `bug`, `security`

### Problem Statement

`create_escrow` (`contracts/escrow_contract/lib.rs:916-963`) takes
`fleet_id: Option<u64>` from the caller and stores it on the `EscrowRecord`
without validating that the named fleet has any real relationship to the
delivery or the driver:

```rust
pub fn create_escrow(
    env: Env, sender: Address, recipient: Address, driver: Address,
    delivery_id: u64, token: Address, amount: i128, fleet_id: Option<u64>,
) { /* ...fleet_id is stored as-is on the EscrowRecord, and separately used
       to look up a payout address, with no membership check gating
       whether the sender may name this fleet at all... */ }
```

At settlement, this stored value (or a live re-resolution of it, see #375)
decides the payout destination via `fleet_management_contract::get_payout_address`.

This is the exact defect filed as **issue #297**, which GitHub currently
shows as `CLOSED` / `COMPLETED`. As with #296 (see #376), the pull request
that closed it (`#361`) made no code changes. This issue restates the
defect against current `main` and flags #297 for reopening.

### Why It Matters

The sender — the party paying, and the party whose interests are opposite
the driver's on payout — unilaterally selects the fleet whose treasury
receives the driver's earnings. A sender can omit `fleet_id` for a driver
who is an active fleet member (routing the payment to the driver
personally, bypassing the fleet's arrangement), or name a fleet the driver
belongs to but which had nothing to do with this delivery. Neither the
driver nor the fleet consents to or can observe the choice.

### Proposed Solution

Validate the claimed fleet relationship — either at escrow creation (reject
a `fleet_id` the driver isn't an active member of) or resolve the driver's
fleet from the fleet contract rather than trusting the sender's supplied
value, as previously proposed for #297. Whichever direction is chosen
should be coordinated with the fix for #375, since both touch the same
fleet-routing code path.

### Acceptance Criteria

- [ ] A sender cannot route a driver's payout to a fleet the driver is not active in
- [ ] A sender cannot bypass a driver's active fleet arrangement by omitting `fleet_id`
- [ ] Routing for a driver with no fleet membership is unchanged
- [ ] Routing for a driver in exactly one fleet is unchanged
- [ ] Regression test covers a sender naming a fleet the driver does not belong to
- [ ] Issue #297 reopened or explicitly superseded by this issue with a note explaining why

### Technical Notes

- `fleet_management_contract::get_payout_address` already returns the driver's own address for `Pending`, `Removed`, and `None` membership statuses, so an invalid claim currently degrades to a direct payout rather than failing outright — that is the existing safety net, but it means the misrouting only affects drivers who *are* legitimately active in some fleet, just not necessarily the one named.
- Coordinate with #375 (payout-routing snapshot inconsistency) — both touch `create_escrow`'s fleet-routing logic.

### Relevant Files

- `contracts/escrow_contract/lib.rs` — `create_escrow`, `payout_driver`
- `contracts/fleet_management_contract/lib.rs` — `get_payout_address`

### Testing Requirements

- Integration test: sender names a fleet the driver is not a member of → payout does not reach that treasury
- Integration test: sender omits `fleet_id` for an active fleet driver → behavior matches the agreed policy
- Regression test: driver with no fleet is paid directly
- Regression test: driver in one fleet routes to that treasury

### Definition of Done

- [ ] Fleet routing determined by the driver's actual membership rather than the sender's unchecked claim
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**High**

### Estimated Effort

1–2 days

### Dependencies

Restates closed issue #297; recommend reopening #297 rather than tracking both. Related to #375.

---

## #378 — `docs/API.md` omits the entire holdback-expiry escape hatch (`set_holdback_window`, `get_holdback_window`, `release_expired_holdback`)

**Labels:** `documentation`

### Problem Statement

`docs/API.md` documents `mark_holdback_escrow` and `release_holdback_escrow`
(confirmed present in the doc), but grepping the full document for
`set_holdback_window`, `get_holdback_window`, and `release_expired_holdback`
returns zero matches for all three. These are real, implemented, public
functions (`contracts/escrow_contract/lib.rs:1759-1779`, `1733-1756`) that
together form the permissionless escape hatch preventing a passive
recipient from stranding a driver's funds in `Holdback` indefinitely.

### Why It Matters

This is a security-relevant mechanism — it exists specifically so a
driver's funds cannot be trapped by an unresponsive recipient — and it is
entirely invisible to anyone reading the documented API surface. An
integrator or auditor relying on `docs/API.md` would not know the window is
configurable, what it defaults to, or that a permissionless release path
exists at all.

### Proposed Solution

Add documentation entries for all three functions in the Escrow Contract
section of `docs/API.md`, alongside the existing `mark_holdback_escrow` /
`release_holdback_escrow` entries, including the default window
(`DEFAULT_HOLDBACK_WINDOW_SECONDS`, 3 days) and minimum bound
(`MIN_HOLDBACK_WINDOW_SECONDS`, 1 day).

### Acceptance Criteria

- [ ] `set_holdback_window`, `get_holdback_window`, and `release_expired_holdback` all have entries in `docs/API.md`
- [ ] The documented behavior matches the actual code (default/min values, permissionless-caller note)

### Technical Notes

- No code changes required — documentation only.

### Relevant Files

- `docs/API.md`
- `contracts/escrow_contract/lib.rs` — `set_holdback_window`, `get_holdback_window`, `release_expired_holdback`

### Testing Requirements

N/A — documentation change, no test coverage applicable.

### Definition of Done

- [ ] All three functions documented
- [ ] Reviewed for accuracy against current code

### Complexity

**Low**

### Estimated Effort

1–2 hours

### Dependencies

None.

---

## #379 — `dispute_resolution_contract` never extends the TTL of any of its instance-storage entries

**Labels:** `bug`, `security`

### Problem Statement

Every persistent-storage write in `contracts/dispute_resolution_contract/lib.rs`
(e.g. the `Dispute(DeliveryId)` key) calls `.extend_ttl(...)` after writing.
No equivalent exists for **instance** storage. Grepping the whole file for
`instance().set` finds writes to `DataKey::Admin`, `AdminList`,
`DeliveryContract`, `EscrowContract`, `IdentityReputationContract`,
`DisputeTimeLimit`, `DisputeResolutionLimit`, and `DisputeReputationPenalty`
(e.g. lines 185-205, 214-229, 274-278, 323-325, 355-357, 389-391, 407-409) —
none is followed by an instance-TTL extension, and there is no
`extend_instance_ttl`-style helper anywhere in the file at all.

### Why It Matters

`escrow_contract` had this exact class of bug — instance storage archiving
after a quiet period, silently bricking admin-gated functions once
`is_admin` starts reading `None` — and hardened it with a dedicated
`extend_instance_ttl` helper called after every instance write (see
`escrow_contract/lib.rs:86-102`). `dispute_resolution_contract` has no such
helper at all, meaning its entire admin roster (`AdminList`), its wiring to
every peer contract, and its dispute-window configuration are all at risk
of archival with no code path that ever refreshes their TTL. This is
broader than the escrow_contract case: it's not a few setters that were
missed, it's the complete absence of the pattern. If the instance entry
archives, `is_admin` fails closed for every admin, and the contract's
governance becomes unrecoverable without a low-level storage restore.

### Proposed Solution

Add an `extend_instance_ttl` helper mirroring `escrow_contract`'s, and call
it after every instance-storage write in this file (`init`, `add_admin`,
`remove_admin`, `set_delivery_contract`, `set_escrow_contract`,
`set_identity_reputation_contract`, `update_dispute_time_limit`,
`set_dispute_resolution_limit`, `set_dispute_reputation_penalty`).

### Acceptance Criteria

- [ ] Every instance-storage write in `dispute_resolution_contract` is followed by a TTL extension
- [ ] A regression test simulates ledger advancement past the TTL threshold and confirms the admin/peer-contract wiring survives

### Technical Notes

- Mirror `escrow_contract::extend_instance_ttl` (`lib.rs:86-102`) exactly for consistency across the codebase.
- This is the same class of defect as #386 (fleet_management_contract) and #395 (delivery_contract) below — each contract needs its own fix since none currently share a common instance-TTL helper.

### Relevant Files

- `contracts/dispute_resolution_contract/lib.rs` — all instance-storage writers

### Testing Requirements

- Regression test: advance the ledger past `ttl::LEDGER_TTL_THRESHOLD` after `init` with no other activity, then confirm `is_admin` still recognizes the real admin

### Definition of Done

- [ ] `extend_instance_ttl` helper added and called from every instance writer
- [ ] Test above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

2–4 hours

### Dependencies

Same pattern as #386, #395 — consider fixing together for consistency, but independently shippable.

---

## #380 — `force_resolve_dispute`'s automatic 50/50 split skips the reputation penalty the admin-driven split path applies for the same outcome

**Labels:** `bug`

### Problem Statement

`resolve_dispute_split_funds` (`contracts/dispute_resolution_contract/lib.rs:663-746`)
applies `DISPUTE_REPUTATION_SPLIT_PENALTY` to the driver when an admin
resolves a dispute with a split outcome (`EscrowStatus::Split`).
`force_resolve_dispute` (`lib.rs:839-919`) — the permissionless timeout path
that also produces a 50/50 `Split` outcome when no admin has ruled within
the resolution window — applies no reputation penalty at all.

### Why It Matters

Two disputes that resolve to an identical funds outcome (`Split`) produce
different, inconsistent reputation consequences for the driver purely based
on which code path resolved them — an admin ruling versus a timeout. No
test in the suite (`test_force_resolve_dispute_by_party_after_window_elapsed`
and siblings) asserts on reputation at all after a force-resolve, which
confirms this is an untested gap rather than a deliberate, documented
design choice.

### Proposed Solution

Apply the same `DISPUTE_REPUTATION_SPLIT_PENALTY` (or an explicitly
different, documented value) in `force_resolve_dispute`'s split branch,
matching `resolve_dispute_split_funds`'s behavior for the same outcome.

### Acceptance Criteria

- [ ] A dispute resolved via `force_resolve_dispute`'s split branch applies the same reputation consequence as `resolve_dispute_split_funds`
- [ ] Regression test asserts driver reputation after a force-resolved split matches an admin-resolved split under equivalent starting conditions

### Technical Notes

- If a difference in penalty is actually intended (e.g. force-resolution shouldn't penalize a driver for an admin's inaction), that should be a documented, deliberate choice — not a silent omission — and this issue's proposed solution should be adjusted accordingly during review.

### Relevant Files

- `contracts/dispute_resolution_contract/lib.rs` — `resolve_dispute_split_funds`, `force_resolve_dispute`

### Testing Requirements

- Regression test: force-resolved split applies the same reputation penalty as an admin-resolved split
- Unit test: penalty value used in both code paths comes from the same source (`DISPUTE_REPUTATION_SPLIT_PENALTY` or equivalent)

### Definition of Done

- [ ] Reputation consequence for a `Split` outcome is consistent regardless of resolution path
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

2–4 hours

### Dependencies

None.

---

## #381 — `set_dispute_resolution_limit` and `set_identity_reputation_contract` are security-relevant admin setters that emit no event

**Labels:** `bug`, `enhancement`

### Problem Statement

`set_dispute_resolution_limit` (`contracts/dispute_resolution_contract/lib.rs:378-392`)
and `set_identity_reputation_contract` (`lib.rs:314-326`) both change
protocol-relevant configuration — the force-resolution timeout window and
the reputation contract every dispute ruling routes through — but neither
calls `env.events().publish(...)`. Sibling setters in the same file,
`update_dispute_time_limit` and `set_dispute_reputation_penalty`, both emit
a descriptive event on every change.

### Why It Matters

An off-chain monitor watching for changes to the dispute-resolution window
or to which reputation contract disputes are wired into — both plausible
targets for a compromised or malicious admin key — would see nothing for
these two setters while catching every other comparable configuration
change in the same contract. This is an inconsistency the contract itself
otherwise avoids.

### Proposed Solution

Add an event to each function, following the existing pattern (e.g.
`(caller, old_value, new_value)` for `set_dispute_resolution_limit`, and
`(caller, new_identity_reputation_contract)` for
`set_identity_reputation_contract`, matching sibling peer-contract setters
elsewhere in the codebase).

### Acceptance Criteria

- [ ] `set_dispute_resolution_limit` emits an event with the old and new limit
- [ ] `set_identity_reputation_contract` emits an event with the new address
- [ ] Regression tests assert both events are published

### Technical Notes

- Match the topic-naming convention already used for `update_dispute_time_limit`'s `dispute_time_limit_updated` event.

### Relevant Files

- `contracts/dispute_resolution_contract/lib.rs` — `set_dispute_resolution_limit`, `set_identity_reputation_contract`

### Testing Requirements

- Regression test: `set_dispute_resolution_limit` publishes the expected event
- Regression test: `set_identity_reputation_contract` publishes the expected event

### Definition of Done

- [ ] Both setters emit events
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

1–2 hours

### Dependencies

None.

---

## #382 — No unauthorized-caller test exists for `update_dispute_time_limit` or `set_dispute_resolution_limit`

**Labels:** `test`

### Problem Statement

Both `update_dispute_time_limit` and `set_dispute_resolution_limit`
(`contracts/dispute_resolution_contract/lib.rs:378-414`) gate on
`is_admin`, but neither has a dedicated `test_unauthorized_..._fails`-style
test in `test.rs`. Every other admin setter in the same file
(`set_dispute_reputation_penalty`, `add_admin`) has one.

### Why It Matters

These two functions gate the dispute-resolution timeout window — a
parameter that directly controls how long a dispute stays contestable
before automatic resolution kicks in. An authorization regression here
(e.g. from a future refactor) would not be caught by the existing suite.

### Proposed Solution

Add `test_unauthorized_update_dispute_time_limit_fails` and
`test_unauthorized_set_dispute_resolution_limit_fails`, matching the
existing pattern used for `set_dispute_reputation_penalty`.

### Acceptance Criteria

- [ ] Both functions have a dedicated unauthorized-caller test
- [ ] Tests pass against current code (confirming the existing `is_admin` gate already works — this is a coverage gap, not a known-broken auth check)

### Technical Notes

- This is a coverage gap, not a suspected authorization bug — both functions do call `is_admin` correctly today.

### Relevant Files

- `contracts/dispute_resolution_contract/test.rs`

### Testing Requirements

- `test_unauthorized_update_dispute_time_limit_fails`
- `test_unauthorized_set_dispute_resolution_limit_fails`

### Definition of Done

- [ ] Both tests added and passing

### Complexity

**Low**

### Estimated Effort

1–2 hours

### Dependencies

None.

---

## #383 — `raise_dispute`'s `Delivered` branch uses unchecked addition instead of `saturating_add`

**Labels:** `bug`

### Problem Statement

`raise_dispute`'s `Delivered` branch
(`contracts/dispute_resolution_contract/lib.rs:437-443`) computes the
dispute deadline as `delivered_at + dispute_limit` using plain `+`. Two
functions later, `force_resolve_dispute`'s equivalent deadline check
(`lib.rs:872`) uses `saturating_add` for the identical kind of
timestamp-plus-window computation — the established convention elsewhere in
this file and across the codebase.

### Why It Matters

With `overflow-checks = true` in the release profile, an overflow here
would panic rather than saturate. This is practically unreachable — it
would require `delivered_at` to already be within `dispute_limit` of
`u64::MAX` — but it's a real, verifiable inconsistency against the one
convention this codebase otherwise applies uniformly to every
timestamp-plus-duration deadline calculation, in the same file, two
functions apart.

### Proposed Solution

Change `delivered_at + dispute_limit` to
`delivered_at.saturating_add(dispute_limit)`, matching
`force_resolve_dispute`.

### Acceptance Criteria

- [ ] The `Delivered` branch's deadline computation uses `saturating_add`
- [ ] Existing dispute-window behavior is unchanged for all realistic timestamp values

### Technical Notes

- Low severity, included for completeness and consistency rather than as an active exploit — flagged explicitly as such.

### Relevant Files

- `contracts/dispute_resolution_contract/lib.rs` — `raise_dispute`

### Testing Requirements

- Regression test: existing dispute-window behavior unchanged after the arithmetic change

### Definition of Done

- [ ] `saturating_add` used consistently for this deadline check
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

30 minutes

### Dependencies

None.

---

## #384 — `fleet_management_contract::init` never calls `admin.require_auth()`, letting anyone install themselves as admin

**Labels:** `bug`, `security`

### Problem Statement

`fleet_management_contract::init` (`contracts/fleet_management_contract/lib.rs:148-156`)
writes the supplied `admin` address to storage without ever calling
`admin.require_auth()`. Every other contract's `init` — escrow, delivery,
dispute_resolution, identity_reputation, settlement — calls
`admin.require_auth()` before writing `StorageKey::Admin`; this is the sole
exception.

### Why It Matters

Before the legitimate deployer calls `init` (or via a front-run against an
unsent `init` transaction), any caller can invoke
`init(env, attacker_address)` and become the sole, unrestricted admin of
the entire fleet-management contract — gaining control of
`admin_reassign_fleet_owner` and `admin_force_update_treasury`, the two
emergency-override functions meant to protect fleets from exactly this kind
of takeover.

### Proposed Solution

Add `admin.require_auth()` to `init`, matching every other contract's
initializer, before the `AlreadyInitialized` check or immediately after it.

### Acceptance Criteria

- [ ] `init` requires authorization from the `admin` address before writing it to storage
- [ ] Regression test confirms `init` fails when the caller does not authorize as `admin`
- [ ] Existing legitimate-init tests still pass

### Technical Notes

- This is the same category of defect the other five contracts already correctly guard against — the fix is to bring this contract in line with the established pattern, not to design something new.

### Relevant Files

- `contracts/fleet_management_contract/lib.rs` — `init`

### Testing Requirements

- Regression test: `init` without `admin.require_auth()` mock-authorization fails
- Regression test: legitimate `init` (with authorization) still succeeds

### Definition of Done

- [ ] `init` requires admin authorization
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

1–2 hours

### Dependencies

None. Highest priority in this batch — recommend fixing immediately regardless of publication order.

---

## #385 — Any fleet configured with `signature_threshold > 1` is permanently locked out of its own treasury and roster functions

**Labels:** `bug`, `security`

### Problem Statement

`require_signer_threshold` (`contracts/fleet_management_contract/lib.rs:48-62`)
counts authorized signers for a call and `break`s after the first match, so
`authorized_signer_count` for any single invocation is always 0 or 1. The
comparison `authorized_signer_count < profile.signature_threshold`
therefore always evaluates true — and the call always fails — for any fleet
with `threshold >= 2`. `configure_signers` accepts thresholds up to
`signers.len()` with no upper guard against this.

### Why It Matters

Multi-signature is a named, implemented feature for fleet treasury
management, but it is structurally non-functional: **no** call can ever
satisfy a threshold of 2 or more in a single transaction, since the
function that checks the threshold only ever sees one signer's
authorization per invocation. The existing test
`test_signer_threshold_is_enforced_for_fleet_actions`
(`contracts/fleet_management_contract/test.rs:1007-1039`) already proves
this without the test author apparently recognizing it as a bug: after
setting `threshold = 2`, it asserts that `update_fleet_treasury`,
`add_driver_to_fleet`, `cancel_invite`, and `remove_driver_from_fleet`
**all fail even when called by the fleet owner**. No test anywhere shows a
threshold-2+ fleet successfully completing a signer-gated action, because
under the current implementation it cannot happen. Any fleet owner who
configures multi-sig for genuine security instead permanently locks their
own fleet out of treasury changes and roster management, recoverable only
via an admin emergency-override path.

### Proposed Solution

Soroban contract calls don't natively support multi-party co-signing within
a single transaction the way this code assumes; implementing real
multi-sig requires either an off-chain co-signature aggregation scheme
(collect N signatures, submit as one call with N `require_auth` checks
against the N distinct signers) or a propose/approve/execute pattern
(one signer proposes an action, subsequent signers approve it, the Nth
approval executes it) — the second is more idiomatic for Soroban and
mirrors the existing `PendingSettlementContract`/`propose_admin`-style
timelock patterns already used elsewhere in this codebase. Either way, the
current single-call, single-signer-max design needs to be replaced, not
patched.

### Acceptance Criteria

- [ ] A fleet with `signature_threshold >= 2` can successfully complete a signer-gated action given the required number of distinct signer authorizations
- [ ] `test_signer_threshold_is_enforced_for_fleet_actions` is updated to prove the *positive* case (threshold met → success), not just the negative case
- [ ] Existing threshold-1 (no-op) behavior is unchanged

### Technical Notes

- This is a design-level gap, not a one-line fix — flagged as `High` complexity accordingly.
- `configure_signers` should also gain an upper-bound check (`threshold <= signers.len()`) as part of this work if not already enforced elsewhere.

### Relevant Files

- `contracts/fleet_management_contract/lib.rs` — `require_signer_threshold`, `configure_signers`, and every function gated by it (`update_fleet_treasury`, `add_driver_to_fleet`, `cancel_invite`, `remove_driver_from_fleet`)
- `contracts/fleet_management_contract/test.rs` — `test_signer_threshold_is_enforced_for_fleet_actions`

### Testing Requirements

- Integration test: a threshold-2 fleet's treasury update succeeds given two valid signer authorizations submitted per the new mechanism
- Regression test: a threshold-2 fleet's treasury update still fails given only one signer authorization
- Regression test: threshold-1 (effectively single-signer) fleets are unaffected

### Definition of Done

- [ ] Multi-signature fleets can actually complete signer-gated actions
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**High**

### Estimated Effort

2–3 days

### Dependencies

None, but should be prioritized alongside #384 — both are fleet_management_contract authorization defects with real fund/governance impact.

---

## #386 — `fleet_management_contract` instance storage (`Admin`, `IdentityContract`, `EscrowContract`) is never TTL-extended

**Labels:** `bug`, `security`

### Problem Statement

Every persistent-storage write in `contracts/fleet_management_contract/lib.rs`
(`Fleet`, `DriverFleet`, `FleetRoster`, `PendingTreasury`) is followed by
`.extend_ttl(...)`. Instance storage (`Admin`, `IdentityContract`,
`EscrowContract`, written e.g. at `lib.rs:148-179`) is never touched by any
TTL-extension call — there is no `extend_instance_ttl`-equivalent helper in
this file.

### Why It Matters

If the instance entry archives after a quiet period, `is_admin` reads
`None` and returns `false` for every caller, including the real admin —
silently and fail-closed bricking every admin-gated function in the
contract: `deactivate_fleet`, `admin_reassign_fleet_owner`,
`admin_force_update_treasury`, `set_identity_contract`,
`set_escrow_contract`. This is the same class of defect already fixed for
`escrow_contract` (issues #25/#299) and flagged again for
`dispute_resolution_contract` (#379) and `delivery_contract` (#395) in this
batch.

### Proposed Solution

Add an `extend_instance_ttl` helper mirroring `escrow_contract`'s, called
after every instance-storage write (`init`, `set_identity_contract`,
`set_escrow_contract`, and any other admin-writing function).

### Acceptance Criteria

- [ ] Every instance-storage write in `fleet_management_contract` is followed by a TTL extension
- [ ] Regression test simulates ledger advancement past the TTL threshold and confirms admin-gated functions still work

### Technical Notes

- Mirror `escrow_contract::extend_instance_ttl` (`lib.rs:86-102`) for consistency.

### Relevant Files

- `contracts/fleet_management_contract/lib.rs` — `init`, `set_identity_contract`, `set_escrow_contract`

### Testing Requirements

- Regression test: advance the ledger past `ttl::LEDGER_TTL_THRESHOLD` after `init` with no other activity, then confirm `is_admin` still recognizes the real admin

### Definition of Done

- [ ] `extend_instance_ttl` helper added and called from every instance writer
- [ ] Test above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

2–4 hours

### Dependencies

Same pattern as #379, #395.

---

## #387 — `accept_fleet_invite`'s roster-full guard reports the wrong, misleading error

**Labels:** `bug`

### Problem Statement

`accept_fleet_invite` (`contracts/fleet_management_contract/lib.rs:674-677`)
panics with `FleetError::FleetNotFound` when
`profile.total_active_drivers >= MAX_ROSTER_SIZE`, even though the fleet
plainly exists and was just looked up successfully.

### Why It Matters

`FleetNotFound` tells a caller (or off-chain error handler) that the fleet
ID was wrong, when the actual problem is that the roster is full. This is
the same category of defect already fixed elsewhere in the codebase for
`create_escrows_batch`'s size cap, which was given its own dedicated
`EscrowError::BatchTooLarge` rather than reusing an unrelated error
variant — the same fix was never applied here.

### Proposed Solution

Add a dedicated `FleetError::RosterFull` (or equivalent) variant and use it
in place of `FleetNotFound` for this guard.

### Acceptance Criteria

- [ ] A roster-full rejection returns a dedicated, correctly-named error
- [ ] `FleetNotFound` is reserved for actual missing-fleet cases
- [ ] Regression test asserts the new error variant on a full roster

### Technical Notes

- Follows the precedent set by `EscrowError::BatchTooLarge`.

### Relevant Files

- `contracts/fleet_management_contract/lib.rs` — `accept_fleet_invite`, `FleetError`

### Testing Requirements

- Regression test: accepting an invite into a full roster returns the new dedicated error, not `FleetNotFound`

### Definition of Done

- [ ] Dedicated error variant added and used
- [ ] Test above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

1–2 hours

### Dependencies

None.

---

## #388 — `remove_driver_from_fleet`'s roster compaction is O(n) per removal and can exceed transaction resource limits for large fleets

**Labels:** `bug`, `performance`

### Problem Statement

`remove_driver_from_fleet` (`contracts/fleet_management_contract/lib.rs:767-805`)
compacts the roster by rewriting every entry after the removed driver's
position, up to `total_active_drivers`, which can be as large as
`MAX_ROSTER_SIZE`.

### Why It Matters

Removing a driver near the front of a large, long-lived fleet's roster
requires rewriting every subsequent entry in a single transaction. A fleet
that grows toward `MAX_ROSTER_SIZE` could reach a point where removing an
early-joined driver requires more storage writes/instructions than a
single Soroban transaction permits — making that driver practically
unremovable through normal means, a real operational dead end for large
fleets.

### Proposed Solution

Switch to an index structure that doesn't require full compaction on
removal — e.g. tombstoning the removed slot (leave a marker, skip it on
iteration) rather than shifting every subsequent entry, or a
doubly-linked/paged structure similar to the bounded-page pattern already
used for escrow indexes (`INDEX_PAGE` in `escrow_contract/lib.rs`).

### Acceptance Criteria

- [ ] Removing any single driver from a roster of realistic maximum size completes within normal transaction resource limits
- [ ] Roster read/iteration behavior (`get_fleet_roster` and friends) is unaffected from the caller's perspective

### Technical Notes

- `MAX_ROSTER_SIZE` should be checked against realistic Soroban transaction instruction/storage-write limits to quantify at what roster size this actually becomes unreachable, as part of scoping the fix.

### Relevant Files

- `contracts/fleet_management_contract/lib.rs` — `remove_driver_from_fleet`, roster storage layout

### Testing Requirements

- Test: removing a driver from a near-maximum-size roster completes successfully
- Regression test: roster contents and ordering (where order matters to callers) are correct after removal under the new structure

### Definition of Done

- [ ] Roster removal no longer requires full O(n) compaction
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**High**

### Estimated Effort

1–2 days

### Dependencies

None.

---

## #389 — `remove_driver_from_fleet`'s roster-compaction loop uses a raw `.unwrap()` instead of a typed contract error

**Labels:** `bug`, `refactor`

### Problem Statement

Inside the roster-compaction loop, `remove_driver_from_fleet`
(`contracts/fleet_management_contract/lib.rs:789-793`) calls
`env.storage().persistent().get(&next_key).unwrap()` — a raw `.unwrap()` —
rather than the `panic_with_error!`-with-typed-`FleetError` pattern used by
every other fallible storage read in this codebase.

### Why It Matters

If the roster is ever inconsistent (e.g. from a future refactor bug
introducing a gap), this produces an opaque, untyped WASM trap instead of a
diagnosable typed error, breaking the established convention that makes
every other failure mode in this codebase identifiable off-chain by error
code.

### Proposed Solution

Replace the raw `.unwrap()` with
`.unwrap_or_else(|| panic_with_error!(&env, FleetError::...))` using an
appropriately named variant (new or existing), matching the rest of the
file.

### Acceptance Criteria

- [ ] The roster-compaction loop uses a typed error instead of a raw `.unwrap()`
- [ ] Normal (non-corrupted) roster compaction behavior is unchanged

### Technical Notes

- Low-risk, mechanical fix — bundling with #388's larger roster-structure rework may be more efficient than fixing in isolation, but is independently shippable first if #388 is deferred.

### Relevant Files

- `contracts/fleet_management_contract/lib.rs` — `remove_driver_from_fleet`

### Testing Requirements

- Regression test: normal roster compaction still succeeds and produces identical results to before the change

### Definition of Done

- [ ] Typed error used in place of the raw `.unwrap()`
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

1 hour

### Dependencies

Consider bundling with #388.

---

## #390 — No way to reactivate a fleet once `deactivate_fleet` sets it inactive

**Labels:** `enhancement`, `feature`

### Problem Statement

`deactivate_fleet` (`contracts/fleet_management_contract/lib.rs:269-307`)
sets a fleet inactive; no corresponding `reactivate_fleet` / `activate_fleet`
function exists anywhere in the file.

### Why It Matters

The driver-lifecycle equivalent in `identity_reputation_contract`
(`suspend_driver` / `reinstate_driver`) is explicitly reversible. Fleet
deactivation is not — an accidental deactivation, or one issued during an
incident that later resolves, permanently strands the fleet with no
recovery path short of an admin-mediated migration to a brand-new fleet
record.

### Proposed Solution

Add a `reactivate_fleet` function, gated by the same authorization as
`deactivate_fleet` (owner and/or admin), that reverses the inactive flag
and emits a corresponding event, mirroring the
`suspend_driver`/`reinstate_driver` pattern.

### Acceptance Criteria

- [ ] A deactivated fleet can be reactivated by an authorized caller
- [ ] Reactivation emits an event
- [ ] All fleet operations that were blocked while inactive resume normally after reactivation

### Technical Notes

- Follow the `suspend_driver`/`reinstate_driver` precedent in `identity_reputation_contract` for authorization and event-naming conventions.

### Relevant Files

- `contracts/fleet_management_contract/lib.rs` — `deactivate_fleet`, new `reactivate_fleet`

### Testing Requirements

- Test: reactivating a deactivated fleet succeeds and restores normal operation
- Test: unauthorized caller cannot reactivate
- Test: reactivating an already-active fleet is a defined no-op or rejected consistently

### Definition of Done

- [ ] `reactivate_fleet` implemented and tested
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

2–4 hours

### Dependencies

None.

---

## #391 — `identity_reputation_contract::register_user` has no authorization check

**Labels:** `bug`, `security`

### Problem Statement

`register_user` (`contracts/identity_reputation_contract/lib.rs:251-277`)
never calls `user.require_auth()` anywhere in its body. `register_driver`,
by contrast, calls `driver.require_auth()` (`lib.rs:222`) before writing a
`DriverProfile`.

### Why It Matters

Any caller can create a `UserProfile` — including a caller-chosen
`registered_at` timestamp — for any address, including one that never
signed anything. This lets an unrelated party fabricate or backdate
registration records attributed to addresses they don't control.

### Proposed Solution

Add `user.require_auth()` to `register_user`, matching `register_driver`'s
pattern.

### Acceptance Criteria

- [ ] `register_user` requires authorization from the `user` address
- [ ] Regression test confirms `register_user` fails when the caller does not authorize as `user`
- [ ] Existing legitimate-registration tests still pass

### Technical Notes

- `test.rs:48-67` currently only tests idempotency (re-registration behavior), not unauthorized-caller rejection — this gap in coverage is why the missing check went unnoticed.

### Relevant Files

- `contracts/identity_reputation_contract/lib.rs` — `register_user`

### Testing Requirements

- Regression test: `register_user` without `user.require_auth()` mock-authorization fails
- Regression test: legitimate registration (with authorization) still succeeds

### Definition of Done

- [ ] `register_user` requires user authorization
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

1–2 hours

### Dependencies

None.

---

## #392 — `award_reputation` omits the `require_escrow_not_paused` guard its own doc comment claims to mirror

**Labels:** `bug`

### Problem Statement

`award_reputation`'s doc comment
(`contracts/identity_reputation_contract/lib.rs:420-427`) states it is
"mirroring `decrease_reputation`". `decrease_reputation` (`lib.rs:392`) and
`increase_reputation` (`lib.rs:347`) both call
`require_escrow_not_paused(&env)` as their first check. `award_reputation`
(`lib.rs:428-432`) does not call it at all.

### Why It Matters

The function's own documentation asserts parity with a sibling that
enforces the pause guard, but the implementation doesn't. It happens to be
safe today only because its sole current caller
(`dispute_resolution_contract::resolve_dispute_pay_driver`) independently
checks pause state before invoking it — but any other authorized contract
calling `award_reputation` directly bypasses the pause guard entirely,
undermining the protocol-wide pause as an emergency control.

### Proposed Solution

Add `require_escrow_not_paused(&env)` as the first check in
`award_reputation`, matching its siblings and its own documented intent.

### Acceptance Criteria

- [ ] `award_reputation` rejects calls while the escrow contract is paused
- [ ] Existing (unpaused) behavior is unchanged
- [ ] Regression test confirms the pause guard is enforced independently of the caller

### Technical Notes

- Low-risk, mechanical fix — one line, matching an existing pattern already used twice in the same file.

### Relevant Files

- `contracts/identity_reputation_contract/lib.rs` — `award_reputation`

### Testing Requirements

- Regression test: `award_reputation` fails while paused, called directly (not through `dispute_resolution_contract`)

### Definition of Done

- [ ] Pause guard added
- [ ] Test above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

30 minutes

### Dependencies

None.

---

## #393 — `set_reputation_config` has no upper-bound validation, unlike the analogous penalty config in `dispute_resolution_contract`

**Labels:** `bug`, `security`

### Problem Statement

`set_reputation_config` (`contracts/identity_reputation_contract/lib.rs:143-151`)
accepts any `u32` for `base_points`, `heavy_cargo_points`, and
`fragile_points` with no upper-bound check.
`dispute_resolution_contract::set_dispute_reputation_penalty` guards its
own single configurable value with `MAX_DISPUTE_REPUTATION_PENALTY`, with
an explicit comment explaining why: "a single mistyped configuration value
... turns every subsequent adverse ruling into a permanent reset." The same
reasoning applies here and is unguarded.

### Why It Matters

An admin (or a mistyped/fat-fingered value) can set `base_points` to an
arbitrarily large number, instantly maxing every driver's reputation score
on their very next delivery — exactly the failure mode
`MAX_DISPUTE_REPUTATION_PENALTY`'s own comment was written to prevent,
just left unguarded on this sibling contract. `test.rs:399-427` only
exercises small, valid configuration values, confirming this boundary is
untested as well as unguarded.

### Proposed Solution

Add an upper-bound constant (e.g. `MAX_REPUTATION_CONFIG_POINTS`, scoped
relative to `MAX_REPUTATION`) and validate all three fields against it in
`set_reputation_config`, mirroring the `MAX_DISPUTE_REPUTATION_PENALTY`
pattern.

### Acceptance Criteria

- [ ] `set_reputation_config` rejects configuration values above a documented, sane maximum
- [ ] Existing valid configuration values are unaffected
- [ ] Regression test asserts rejection of an oversized value for each of the three fields

### Technical Notes

- Follow the `MAX_DISPUTE_REPUTATION_PENALTY` precedent in `dispute_resolution_contract/lib.rs:13-42` for both the bound's derivation and its documentation comment.

### Relevant Files

- `contracts/identity_reputation_contract/lib.rs` — `set_reputation_config`

### Testing Requirements

- Regression test: oversized `base_points` is rejected
- Regression test: oversized `heavy_cargo_points` is rejected
- Regression test: oversized `fragile_points` is rejected
- Regression test: existing valid values still succeed

### Definition of Done

- [ ] Upper bound added and enforced
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

2–3 hours

### Dependencies

None.

---

## #394 — `identity_reputation_contract::is_authorized_contract` extends persistent-storage TTL as a side effect of a read-only check

**Labels:** `performance`, `refactor`

### Problem Statement

`is_authorized_contract` (`contracts/identity_reputation_contract/lib.rs:198-210`)
extends the TTL of its underlying storage entry every time it's called,
despite being an authorization *check* — called on every single invocation
of `increase_reputation`, `decrease_reputation`, and `award_reputation`.

### Why It Matters

A read-only authorization gate performing a storage write (the TTL
extension) on every call adds unnecessary instruction/resource cost to the
hottest paths in this contract, and blurs the "reads don't write" invariant
the rest of the codebase generally follows for simple getters. This is a
minor efficiency/clarity issue rather than a correctness bug — included for
completeness at low priority.

### Proposed Solution

Separate the TTL-extension responsibility from the authorization check:
either extend TTL only when the underlying `AuthorizedContract` entry is
actually written (via `set_authorized_contract`), or make the extension
explicit and intentional at call sites that already know they're in a
state-mutating flow, rather than folding it into every read.

### Acceptance Criteria

- [ ] `is_authorized_contract` no longer performs a storage write as a side effect of a read
- [ ] TTL for the `AuthorizedContract` entry is still extended appropriately when the entry is actually used/written

### Technical Notes

- Low priority — flagged for completeness, not confident it independently clears the bar for urgent action, but it is a real, verifiable pattern deviation worth tracking.

### Relevant Files

- `contracts/identity_reputation_contract/lib.rs` — `is_authorized_contract`

### Testing Requirements

- Regression test: authorization behavior (accept/reject) is unchanged after removing the side-effecting TTL extension

### Definition of Done

- [ ] Read-only check no longer writes storage
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

1–2 hours

### Dependencies

None.

---

## #395 — `delivery_contract` instance storage (`Admin`, `EscrowContract`, `IdentityReputationContract`) is never TTL-extended

**Labels:** `bug`, `security`

### Problem Statement

`init` (`contracts/delivery_contract/lib.rs:168-190`) and
`set_identity_reputation_contract` (`lib.rs:192-205`) both write to
instance storage (`StorageKey::Admin`, `DataKey::EscrowContract`,
`DataKey::IdentityReputationContract`) with no TTL-extension call anywhere
in the file — no `extend_instance_ttl`-equivalent helper exists here,
unlike `escrow_contract`.

### Why It Matters

If the instance entry archives, every function reading
`DataKey::EscrowContract` — `require_escrow_not_paused`, `cancel_delivery`,
`confirm_delivery`, `raise_dispute`, `mark_in_transit`,
`get_combined_state` — starts panicking with `NotInitialized`, bricking the
contract's core delivery lifecycle until an admin somehow re-initializes
storage. Same class of defect as #379 (dispute_resolution_contract) and
#386 (fleet_management_contract) in this batch.

### Proposed Solution

Add an `extend_instance_ttl` helper mirroring `escrow_contract`'s, called
after every instance-storage write (`init`, `set_identity_reputation_contract`,
and any other admin-writing function in this file).

### Acceptance Criteria

- [ ] Every instance-storage write in `delivery_contract` is followed by a TTL extension
- [ ] Regression test simulates ledger advancement past the TTL threshold and confirms core delivery functions still work

### Technical Notes

- Mirror `escrow_contract::extend_instance_ttl` (`lib.rs:86-102`) for consistency across the codebase.

### Relevant Files

- `contracts/delivery_contract/lib.rs` — `init`, `set_identity_reputation_contract`

### Testing Requirements

- Regression test: advance the ledger past `ttl::LEDGER_TTL_THRESHOLD` after `init` with no other activity, then confirm `create_delivery`/`cancel_delivery` still work

### Definition of Done

- [ ] `extend_instance_ttl` helper added and called from every instance writer
- [ ] Test above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

2–4 hours

### Dependencies

Same pattern as #379, #386.

---

## #396 — `create_deliveries_batch` never validates `sender != recipient`, unlike its single-item counterpart

**Labels:** `bug`

### Problem Statement

`create_delivery` (`contracts/delivery_contract/lib.rs:224-235`) explicitly
panics with `DeliveryError::InvalidParties` when `sender == recipient`.
`create_deliveries_batch` (`lib.rs:319-343`) has no equivalent check
anywhere in its body.

### Why It Matters

A sender can batch-create deliveries naming themselves as the recipient,
bypassing a validation their own single-item counterpart enforces in the
exact same contract — an inconsistency between two code paths that should
behave identically for this check.

### Proposed Solution

Add the same `sender != recipient` check to `create_deliveries_batch`,
matching `create_delivery`.

### Acceptance Criteria

- [ ] `create_deliveries_batch` rejects a batch where `sender == recipient`, matching `create_delivery`
- [ ] Regression test confirms rejection

### Technical Notes

- Mechanical parity fix — no design decision required, just applying an existing check consistently.

### Relevant Files

- `contracts/delivery_contract/lib.rs` — `create_delivery`, `create_deliveries_batch`

### Testing Requirements

- Regression test: `create_deliveries_batch` with `sender == recipient` fails with `InvalidParties`

### Definition of Done

- [ ] Check added and enforced
- [ ] Test above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Low**

### Estimated Effort

30 minutes

### Dependencies

None.

---

## #397 — `get_combined_state` panics for any delivery whose escrow hasn't been created yet — an ordinary, common state

**Labels:** `bug`

### Problem Statement

`get_combined_state` (`contracts/delivery_contract/lib.rs:787-808`) is
documented as a general-purpose combined delivery-and-escrow state view,
but unconditionally cross-calls `escrow_contract::get_escrow`, which panics
with `DeliveryNotFound` (`escrow_contract/lib.rs:1781-1787`) when no escrow
exists for that delivery yet.

### Why It Matters

Delivery creation and escrow creation are two separate calls that can
happen in either order — documented repeatedly elsewhere in this codebase.
Any `Pending` delivery with no escrow yet, an entirely ordinary and common
state, makes this "combined state" view function hard-revert instead of
returning something useful (e.g. a flag indicating no escrow exists yet).
A view function that panics on a common, expected state is a poor building
block for any dashboard or monitoring tool built on top of it.

### Proposed Solution

Change `get_combined_state` to tolerate a missing escrow — either by
returning an `Option`/optional-escrow field in its result type, or by
adding a non-panicking existence check to `escrow_contract` (as proposed in
#376) and using it here.

### Acceptance Criteria

- [ ] `get_combined_state` returns a usable result for a delivery with no escrow yet, instead of panicking
- [ ] Behavior for a delivery with an escrow is unchanged
- [ ] Regression test covers both cases

### Technical Notes

- Depends on or can share the non-panicking escrow-existence check proposed in #376.

### Relevant Files

- `contracts/delivery_contract/lib.rs` — `get_combined_state`
- `contracts/escrow_contract/lib.rs` — `get_escrow`

### Testing Requirements

- Unit test: `get_combined_state` on a delivery with no escrow returns a usable result, not a panic
- Regression test: `get_combined_state` on a delivery with an escrow is unchanged

### Definition of Done

- [ ] Function no longer panics on the common no-escrow-yet state
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Medium**

### Estimated Effort

3–5 hours

### Dependencies

Benefits from the same fix proposed in #376.

---

## #398 — `assign_driver` lets any unaffiliated address unilaterally self-assign to someone else's `Pending` delivery

**Labels:** `security`, `enhancement`

### Problem Statement

`assign_driver` (`contracts/delivery_contract/lib.rs:513-555`) authorizes
the caller with `is_caller_admin || caller == driver` — i.e. anyone can
assign themselves as the driver for any `Pending` delivery, with no
invitation, allowlist, or sender-approval step. Once assigned,
`validate_transition` (`lib.rs:124-142`) has no `Active → Active` entry and
no unassign/reassign function exists.

### Why It Matters

Any address can claim driver status on an arbitrary pending delivery it
has no relationship to. Once claimed, if that address never calls
`mark_in_transit`, the sender's only recourse is to cancel the delivery
(`Active → Cancelled` is allowed) and recreate it from scratch — funds and
all, if an escrow was already created. This is a real griefing/denial-of-
service vector: a malicious or careless actor can squat on deliveries
faster than legitimate drivers can claim them, with no cost to the
attacker and real cost (re-creating the delivery, potentially re-funding
escrow) to the sender.

### Proposed Solution

Require some form of sender consent before a self-assignment finalizes —
e.g. an invitation/acceptance flow similar to
`fleet_management_contract`'s driver-invite pattern, or requiring the
delivery's sender to co-authorize the assignment. Alternatively (lower
effort, less complete), add a dedicated `unassign_driver` /
`reassign_driver` function so a squatted delivery has a cheaper recovery
path than full cancellation and recreation.

### Acceptance Criteria

- [ ] A driver cannot become permanently assigned to a delivery without some form of sender consent, or the sender has a low-cost way to undo an unwanted assignment
- [ ] Legitimate driver self-assignment (with consent, however implemented) still works
- [ ] Regression test covers the previously-unrestricted self-assignment path is no longer exploitable as-is

### Technical Notes

- This is a design-level question (consent model vs. cheap-undo model) — flagged as `Medium`/`High` complexity depending on which direction is chosen during review.

### Relevant Files

- `contracts/delivery_contract/lib.rs` — `assign_driver`, `validate_transition`

### Testing Requirements

- Test: an unaffiliated address can no longer unilaterally lock in driver assignment without consent (or: sender can cheaply undo an unwanted assignment)
- Regression test: legitimate assignment flow still works end-to-end

### Definition of Done

- [ ] Griefing vector closed or mitigated with a low-cost recovery path
- [ ] Tests above added and passing
- [ ] Formatting, clippy, and full suite clean

### Complexity

**Medium**

### Estimated Effort

1 day

### Dependencies

None.

---

## #399 — SDK's `createEscrowsBatch` encodes batch entries as a `Map`, not the `Vec<tuple>` the contract expects, and omits `fleet_id` entirely

**Labels:** `bug`, `security`

### Problem Statement

`sdk/typescript/src/clients/escrow.client.ts:70-75` builds each batch entry
via `map([['delivery_id', ...], ['driver', ...], ['amount', ...]])` — an
`ScVal::Map`. The contract's `create_escrows_batch`
(`contracts/escrow_contract/lib.rs:1069-1075`) expects
`soroban_sdk::Vec<(u64, Address, i128, Option<u64>)>` — an `ScVal::Vec` of
4-element tuples, including `fleet_id`, which the SDK encoding never
includes at all.

### Why It Matters

This is a wire-format mismatch on a fund-moving function: a call built with
the current SDK encoding will either fail XDR decoding on-chain (safe but
broken) or, if the mismatch happens to decode into something structurally
valid but semantically wrong, silently drop fleet routing for every
batch-created escrow. Either way, the SDK's batch-creation path does not
work correctly against the current contract at all.

### Proposed Solution

Rewrite the batch-entry encoding to produce a `Vec` of 4-element tuples
`(delivery_id: u64, driver: Address, amount: i128, fleet_id: Option<u64>)`
matching the contract's actual parameter type, including a way for SDK
callers to supply `fleet_id` per entry.

### Acceptance Criteria

- [ ] `createEscrowsBatch`'s wire encoding matches `create_escrows_batch`'s actual parameter type exactly
- [ ] SDK callers can supply a per-entry `fleet_id`
- [ ] Integration test round-trips a batch call through the SDK against a real (or simulated) contract call and confirms all fields, including `fleet_id`, arrive correctly

### Technical Notes

- This should be caught by an XDR round-trip / contract-simulation test, not just a type-level check, since the defect is specifically in the wire encoding.

### Relevant Files

- `sdk/typescript/src/clients/escrow.client.ts` — batch entry encoding
- `contracts/escrow_contract/lib.rs` — `create_escrows_batch`

### Testing Requirements

- Integration test: SDK-built `createEscrowsBatch` call, simulated/executed against the actual contract, succeeds and all fields (including `fleet_id`) are correctly received
- Regression test: existing non-batch `createEscrow` calls are unaffected

### Definition of Done

- [ ] Batch encoding matches the contract's expected type
- [ ] Tests above added and passing
- [ ] SDK build/typecheck clean

### Complexity

**Medium**

### Estimated Effort

4–6 hours

### Dependencies

None. High priority — this is a broken fund-moving SDK path, not a documentation gap.

---

## #400 — SDK's `decodeEscrow` drops `delivery_id` and `holdback_started_at` from the decoded `EscrowRecord`

**Labels:** `bug`

### Problem Statement

`sdk/typescript/src/clients/escrow.client.ts:248-258` and
`sdk/typescript/src/types/common.types.ts:51-64` omit `delivery_id` and
`holdback_started_at` from the decoded/typed `EscrowRecord`, even though
both fields have existed on the Rust `EscrowRecord`
(`contracts/shared_types/lib.rs:673-693`) for several prior waves.

### Why It Matters

Callers reading `getEscrow()` through the SDK silently lose both fields —
`delivery_id` (the record's own primary identifier) and
`holdback_started_at` (needed to compute whether `release_expired_holdback`
is currently callable, see #378). Any SDK consumer trying to build
holdback-expiry tooling on top of this client cannot do so without falling
back to raw contract calls.

### Proposed Solution

Add both fields to the `EscrowRecord` TypeScript type and to
`decodeEscrow`'s field mapping.

### Acceptance Criteria

- [ ] `decodeEscrow` returns `delivery_id` and `holdback_started_at`
- [ ] `EscrowRecord` type in `common.types.ts` declares both fields with correct optionality (`holdback_started_at` is `Option<u64>` in Rust)
- [ ] Regression test decodes a real escrow record and asserts both fields are present and correctly typed

### Technical Notes

- Straightforward mechanical fix once identified — the underlying contract data already includes both fields.

### Relevant Files

- `sdk/typescript/src/clients/escrow.client.ts` — `decodeEscrow`
- `sdk/typescript/src/types/common.types.ts` — `EscrowRecord`

### Testing Requirements

- Regression test: `decodeEscrow` output includes `delivery_id` and `holdback_started_at` with correct types/values

### Definition of Done

- [ ] Both fields decoded and typed correctly
- [ ] Test above added and passing
- [ ] SDK build/typecheck clean

### Complexity

**Low**

### Estimated Effort

1–2 hours

### Dependencies

None.

---

## #401 — `IdentityReputationClient` has no bindings for `suspend_driver`, `reinstate_driver`, or `is_driver_suspended`

**Labels:** `bug`

### Problem Statement

`sdk/typescript/src/clients/identity_reputation.client.ts` has no methods
corresponding to `suspend_driver`, `reinstate_driver`, or
`is_driver_suspended` — all three are implemented
(`contracts/identity_reputation_contract/lib.rs:488-564`) and documented
(`docs/API.md:1904-1974`).

### Why It Matters

The entire driver-suspension lifecycle — a core admin moderation tool — is
unreachable from the SDK. Any application built on this SDK cannot suspend
or reinstate a driver, or even check suspension status, without dropping
down to raw contract calls, defeating the purpose of having a typed client
for this contract.

### Proposed Solution

Add `suspendDriver`, `reinstateDriver`, and `isDriverSuspended` methods to
`IdentityReputationClient`, matching the existing method conventions in the
same file (parameter naming, auth handling, return decoding).

### Acceptance Criteria

- [ ] All three methods added with signatures matching the contract
- [ ] Each method's wire encoding verified against the actual contract call (not just type-level)
- [ ] Regression test exercises all three through a simulated/executed contract call

### Technical Notes

- Follow the same encoding conventions already established for other methods in this client to avoid introducing a new instance of #399's class of defect.

### Relevant Files

- `sdk/typescript/src/clients/identity_reputation.client.ts`

### Testing Requirements

- Integration test: `suspendDriver` → `isDriverSuspended` returns `true` → `reinstateDriver` → `isDriverSuspended` returns `false`

### Definition of Done

- [ ] All three methods implemented and tested
- [ ] SDK build/typecheck clean

### Complexity

**Low**

### Estimated Effort

2–4 hours

### Dependencies

None.

---

## #402 — `DeliveryClient.getDeliveriesByDriver` always throws, and pagination endpoints are entirely unexposed

**Labels:** `bug`, `enhancement`

### Problem Statement

`sdk/typescript/src/clients/delivery.client.ts:160-162` unconditionally
throws `'DeliveryContract does not expose get_deliveries_by_driver'`. The
contract does expose a generic paginated accessor,
`get_deliveries_page(owner, kind, offset, limit)`
(`contracts/delivery_contract/lib.rs:851`), which could serve this exact
query with the appropriate `kind` value — the same pattern
`escrow_contract`'s `get_escrows_page` already supports for its own
per-role queries.

### Why It Matters

A method that always throws is worse than a missing method — it implies
the capability doesn't exist on the contract at all, when in fact the
underlying paginated accessor is right there and already used for other
per-role delivery queries.

### Proposed Solution

Implement `getDeliveriesByDriver` in terms of `get_deliveries_page` with
the correct `kind` value, and expose `get_deliveries_page`/`get_escrows_page`
generically in their respective clients so future per-role query needs
don't require another hardcoded, single-purpose method.

### Acceptance Criteria

- [ ] `getDeliveriesByDriver` returns real data instead of throwing
- [ ] `get_deliveries_page`/`get_escrows_page` are exposed as general pagination methods on their clients
- [ ] Regression test confirms `getDeliveriesByDriver` returns the expected delivery IDs for a driver with known deliveries

### Technical Notes

- Check the exact `kind` value convention used elsewhere in the file/contract for "by driver" queries before implementing, to avoid an off-by-one on the role-kind enum.

### Relevant Files

- `sdk/typescript/src/clients/delivery.client.ts` — `getDeliveriesByDriver`
- `contracts/delivery_contract/lib.rs` — `get_deliveries_page`

### Testing Requirements

- Integration test: `getDeliveriesByDriver` returns correct delivery IDs for a driver with known deliveries, verified against a real contract call

### Definition of Done

- [ ] `getDeliveriesByDriver` implemented and working
- [ ] Generic pagination exposed on relevant clients
- [ ] Test above added and passing
- [ ] SDK build/typecheck clean

### Complexity

**Low**

### Estimated Effort

2–3 hours

### Dependencies

None.

---

## #403 — `DeliveryClient.createDelivery` hardcodes fake cargo data regardless of caller input

**Labels:** `bug`

### Problem Statement

`sdk/typescript/src/clients/delivery.client.ts:52-56` hardcodes the cargo
descriptor sent on-chain as 1 gram, `General` category, not fragile,
regardless of what the caller actually specifies. The SDK's own
`DeliveryMetadata` type (`sdk/typescript/src/types/delivery.types.ts:14-20`)
doesn't even have fields to specify real cargo data — it exposes
`pickupLocation`/`dropoffLocation`/`items`/`notes`/`estimatedDistance`,
where `items` and `notes` are themselves unused by `createDelivery`.

### Why It Matters

Every single delivery created through the SDK reports fake, uniform cargo
data on-chain — weight, category, and fragility, all of which feed directly
into `identity_reputation_contract::increase_reputation`'s point
calculation (heavy-cargo and fragile-cargo bonuses never trigger for any
SDK-originated delivery) and into any downstream logic that inspects cargo
attributes. This silently breaks a real, on-chain-consequential feature for
every application built on this SDK.

### Proposed Solution

Add real cargo fields (`weightGrams`, `category`, `fragile`) to the SDK's
delivery-creation input type, and wire them through to the actual
`CargoDescriptor` sent on-chain in `createDelivery`.

### Acceptance Criteria

- [ ] `createDelivery` accepts and transmits real cargo weight, category, and fragility
- [ ] Existing callers that don't specify cargo data get a documented, sensible default (not silently wrong data presented as real)
- [ ] Regression test confirms cargo data round-trips correctly from SDK call to on-chain record

### Technical Notes

- This is very likely why `increase_reputation`'s heavy-cargo/fragile bonuses have never been observed triggering for SDK-originated traffic, if that's been a point of confusion in the past.

### Relevant Files

- `sdk/typescript/src/clients/delivery.client.ts` — `createDelivery`
- `sdk/typescript/src/types/delivery.types.ts` — `DeliveryMetadata`

### Testing Requirements

- Integration test: `createDelivery` with specific cargo weight/category/fragility results in a matching on-chain `CargoDescriptor`

### Definition of Done

- [ ] Real cargo data flows through from SDK caller to on-chain record
- [ ] Test above added and passing
- [ ] SDK build/typecheck clean

### Complexity

**Low**

### Estimated Effort

2–3 hours

### Dependencies

None.

---

## #404 — `EscrowClient` omits roughly 20 contract functions covering peer-contract wiring and fund-accounting reads

**Labels:** `enhancement`

### Problem Statement

`EscrowClient` has no bindings for: `set_delivery_contract` /
`get_delivery_contract`, `set_dispute_resolution_contract` /
`get_dispute_resolution_contract`, `set_identity_reputation_contract` /
`get_identity_contract`, `propose_admin` / `accept_admin`,
`clear_settlement_contract`, `confirm_settlement_contract` /
`get_pending_settlement_contract`, `clear_fleet_management_contract`,
`release_expired_holdback`, `set_holdback_window` / `get_holdback_window`,
`set_volume_tiers` / `get_volume_tiers`, `get_sender_volume`,
`get_total_locked`, `get_untracked_balance`, `sweep_untracked_balance`,
`get_protocol_version`, `update_slippage_tolerance` /
`get_slippage_tolerance`. Meanwhile `EscrowTypes.FreezeFundsParams` and
`SetDisputeResolutionContractParams` are declared in the types file but
never referenced by any client method — dead types signaling incomplete
wiring.

### Why It Matters

Roughly a third of the escrow contract's public surface, including the
peer-contract wiring an operator needs to deploy and configure the
protocol at all, and the holdback-expiry/volume-tier features covered
elsewhere in this batch (#378, #375), is unreachable through the SDK.
Anything building tooling on top of this client has to fall back to raw
contract calls for basic operational tasks.

### Proposed Solution

Add client methods for all listed functions, following existing
conventions in the file. Remove or wire up the two currently-dead
parameter types.

### Acceptance Criteria

- [ ] All listed functions have corresponding client methods
- [ ] `FreezeFundsParams` and `SetDisputeResolutionContractParams` are either used or removed
- [ ] Each new method verified against a real/simulated contract call, not just type-level

### Technical Notes

- Large in scope — consider splitting into smaller PRs by functional group (peer-contract wiring vs. fund-accounting reads vs. holdback/volume-tier surface) rather than one large change.

### Relevant Files

- `sdk/typescript/src/clients/escrow.client.ts`
- `sdk/typescript/src/types/escrow.types.ts`

### Testing Requirements

- Integration test per new method group, verifying wire-level correctness against the actual contract

### Definition of Done

- [ ] All listed functions exposed
- [ ] Dead types resolved
- [ ] Tests added and passing
- [ ] SDK build/typecheck clean

### Complexity

**High**

### Estimated Effort

2–3 days

### Dependencies

None, but touches the same client as #399/#400 — consider sequencing after those land.

---

## #405 — `type-parity.test.ts` leaves `DisputeStatus`, `DriverTier`, and `DriverFleetStatus` completely unverified

**Labels:** `test`

### Problem Statement

`sdk/typescript/src/__tests__/type-parity.test.ts` only verifies
`EscrowStatus` and `DeliveryStatus` against `shared_types`. `DisputeStatus`,
`DriverTier`, and `DriverFleetStatus` — all declared in
`sdk/typescript/src/types/common.types.ts` — have zero equivalent
verification. The file also verifies no method-signature or parameter
parity at all, only these two enums' variant sets.

### Why It Matters

A test file named `type-parity.test.ts` implies broader coverage than it
actually delivers — a drift in any of the three unverified enums (a new
`DisputeStatus` variant added on the Rust side, for instance) would go
completely undetected by the one test suite whose name suggests it exists
specifically to catch this.

### Proposed Solution

Extend the existing parity checks to cover `DisputeStatus`, `DriverTier`,
and `DriverFleetStatus`, following the same pattern already used for
`EscrowStatus`/`DeliveryStatus`.

### Acceptance Criteria

- [ ] All five enums (`EscrowStatus`, `DeliveryStatus`, `DisputeStatus`, `DriverTier`, `DriverFleetStatus`) are verified for variant parity between the SDK types and `shared_types`
- [ ] Test fails if a variant is added to one side without the other

### Technical Notes

- Scope intentionally limited to enum variant parity, matching the existing test's actual scope — broader method/parameter-signature parity is a separate, larger effort not addressed by this issue.

### Relevant Files

- `sdk/typescript/src/__tests__/type-parity.test.ts`
- `sdk/typescript/src/types/common.types.ts`
- `contracts/shared_types/lib.rs`

### Testing Requirements

- The extended test itself is the deliverable; confirm it fails when a variant is deliberately desynced locally, then passes once corrected

### Definition of Done

- [ ] All five enums covered
- [ ] Test suite passes against current, in-sync definitions
- [ ] SDK build/typecheck clean

### Complexity

**Low**

### Estimated Effort

1–2 hours

### Dependencies

None.
