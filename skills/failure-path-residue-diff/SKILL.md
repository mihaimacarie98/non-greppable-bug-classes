---
name: failure-path-residue-diff
description: >-
  Find security bugs in what a failed, cancelled, reverted or refunded operation leaves
  behind — stuck flags, double charges, misrouted refunds, half-restored caches — by
  fault-injecting every fallible step and diffing all state against the pre-state. Use
  when auditing transaction engines, VMs, settlement and fee logic, rollback or cancel
  paths, nested calls with try/catch, or any operation whose success path is well tested
  and whose failure path is not.
---

# SKILL: Failure-Path Residue Diff — What A Failed Operation Leaves Behind

> **AI LOAD INSTRUCTION**: Reviewers and tests follow the success path. A failed operation is assumed to be a no-op apart from the fee it is meant to charge, so nobody diffs the world before and after it. The residue is a flag left set, a charge applied twice, a refund paid to the wrong party, a cache entry not restored — and it surfaces **later, in an unrelated operation**, so the crash or the wrong balance never points back at its cause. There is no token for "the thing a failure forgot to undo". Treat every failure exit as an operation in its own right, with a post-state that must equal the pre-state plus exactly the documented charges.

## 0. RELATED ROUTING

- [interrupted-state-audit](../interrupted-state-audit/SKILL.md) — when the operation is *interrupted from outside* (signal, power loss) mid-way. This skill covers failures the code itself takes: error returns, cancellations, reverts, caught exceptions.
- [verdict-cache-key-completeness](../verdict-cache-key-completeness/SKILL.md) — when stale state comes from a successful verdict reused in the wrong context.
- [invariant-asymmetry-audit](../invariant-asymmetry-audit/SKILL.md) — when the success branch and the failure branch are siblings and only one maintains an invariant.

## 1. THE SHAPE, FROM PUBLIC INCIDENTS

| Incident | Failure path | Residue | Where it surfaced |
|---|---|---|---|
| Soroban host, [GHSA-pm4j-7r4q-ccg8](https://github.com/stellar/rs-soroban-env/security/advisories/GHSA-pm4j-7r4q-ccg8) (Low; fixed in `soroban-env-host` 26.0.0) | A value-to-storage-key conversion fails on a prohibited type | A "conversion in progress" flag stays set until the next storage access | A caller that *caught* the inner failure (`try_call`) then emits an ordinary event, which is wrongly rejected. The residue crossed a contract boundary that was supposed to contain the failure. |
| Sui mainnet halts, May 2026, [post-mortem](https://www.sui.io/blog/sui-mainnet-halts-resolved-after-major-upgrade) | The scheduler cancels a transaction that would overdraw an address balance | Gas smashing still ran on the cancelled transaction and spent the same funds | Settlement applied a negative delta to a zero balance; the network halted (the post says no user funds were at risk). The interim fix skipped the charge for one cancellation reason; a transaction with **two** reasons (congestion masking the balance error) bypassed it and halted the network again the next morning. |
| Stellar Core 20.1.0, [fee-bump disclosure](https://stellar.org/blog/developers/fee-bump-bug-disclosure) | Refund of the unused refundable fee of a fee-bumped transaction | The refund went to the sponsored account instead of the sponsor that paid | A sponsored user could craft transactions to maximise refunds owed to someone else. Disclosed before the protocol upgrade vote. |

Common features: each operation's success path was correct; each residue was in a field with an unremarkable name; each was found or exploited through a *second* operation.

## 2. THE METHOD

### 2.1 List every failure exit of the operation

Walk the operation and number every place it can stop short of success:

- explicit error returns and `throw`/`panic`/`revert`;
- cancellation decided by a scheduler, a pre-check, a quota, a congestion rule;
- caught exceptions in nested calls (`try_call`, `catch`, error-tolerant loops) — the outer operation continues, so residue from the inner one is **live** for the rest of it;
- timeouts and limit hits (gas, size, depth, iteration count);
- partial success: some sub-steps applied, some not.

Record for each exit which shared state had already been touched: flags, guards, counters, caches, temporary balances, reservation tables, nonces, storage-size accounting, event buffers.

### 2.2 Define the expected post-state for each exit

For each exit, write the post-state the design promises — usually "pre-state, plus fee F charged to payer P, plus nothing else". Be precise about F and P. Fee conservation is the most common place for residue: a charge applied on the failure path that the success path applies elsewhere, a refund that names the wrong party, a charge computed from inputs that the cancellation should have released.

### 2.3 Fault-inject every exit and diff everything

Build a harness that runs the operation with each exit forced in turn, then snapshots **all** global state — not just balances — and diffs it against pre-state plus the expected charge:

```text
for exit in exits:
    s0 = snapshot(world)            # flags, caches, balances, counters, sizes, queues
    force(exit); run(op)
    s1 = snapshot(world)
    report(diff(s1, expected(s0, exit)))   # anything non-empty is residue
```

Then run a **second, unrelated operation** in the same context (same transaction, same block, same host instance, same process) and compare it with the same operation run from a clean context. The Soroban residue is invisible to the first diff if the snapshot omits host-internal flags; the second operation exposes it.

### 2.4 Combine failure reasons

The Sui repeat halt is the lesson: a fix that handles failure reason R1 must be tested with R1 and R2 both present, with R2 the one that is reported. For each exit, ask whether another condition can mask it, change its reported reason, or arrive first. Enumerate pairs; they are few and cheap to test.

### 2.5 Check who can force each exit

Residue only matters if someone can drive the failure on purpose. For each exit, note the cheapest party who can trigger it: any user (malformed argument, overdraw, prohibited type), a contract author (a callee that fails on demand), or only an operator. A failure any caller can force, leaving residue that a later caller observes, is the pattern that becomes a halt, a wrong payout or a denial of service for other users.

## 3. PROVING IT

- The deliverable is a state diff: the failing operation, the field that differs from the expected post-state, and the later operation whose outcome changes because of it.
- For consensus or replicated systems, also show that residue makes an outcome depend on something nodes do not share (process lifetime, cache warmth, ordering within a batch). That is how residue becomes disagreement.
- For accounting residue, show conservation explicitly: sum of debits, sum of credits, and the amount that went missing or appeared, over the failing sequence.
- Keep the harness port-free where possible: unit-level fault injection against the real engine is stronger evidence than a narrative and needs no network.

## 4. REPORTING

1. The operation and the failure exit, with the trigger and who can force it.
2. The residue, as a diff against the documented post-state.
3. The later operation it changes, and the consequence (halt, wrong balance, rejected valid operation).
4. Any interim or partial fix and the combination of failure reasons it misses.

**Remediation text (paste-ready):**

> On the failure path at `<location>`, `<state>` is not restored to its pre-operation value (or `<charge>` is applied to `<party>` instead of `<party>`). Restore all temporary state on every exit, preferably with a scope guard or a transactional snapshot that is discarded on failure rather than per-exit cleanup code. Apply charges on the failure path from the same accounting routine as the success path, and assert conservation (debits equal credits plus fees) at the end of every operation, including cancelled ones. Add tests that force each failure exit, including combinations of failure reasons, and that run a second operation in the same context afterwards.
