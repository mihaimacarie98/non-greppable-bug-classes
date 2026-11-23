---
name: verdict-cache-key-completeness
description: >-
  Find security bugs in caches, memos, "already verified" shortcuts and fast paths
  inside validating systems by comparing every input the original verdict depended on
  with the key the cache actually uses. Use when auditing a node, verifier, policy
  engine, signature or permission checker, or any validator that skips work it believes
  was done before — especially after a performance change or a feature that reads
  state from more than one snapshot.
---

# SKILL: Verdict Cache Key Completeness — The Shortcut That Forgot What The Verdict Depended On

> **AI LOAD INSTRUCTION**: A cached verdict is a claim of the form *"this input was valid under conditions C"*. The cache key records the input; it rarely records C. When the verdict is later spent under different conditions — another height, another upgrade, another caller, another snapshot — the system accepts something it would have rejected if it had done the work. No line is wrong: the check exists, the cache is correct for the context its author pictured, and the skip is a deliberate optimisation. The defect is a **missing key component**, and absence has no token. Base models confirm the check exists, see that it runs on the first path, and stop. Ask instead: on the second path, *which inputs of the original decision are still the same, and who chose them?*

## 0. RELATED ROUTING

- [invariant-asymmetry-audit](../invariant-asymmetry-audit/SKILL.md) — when the fast path and the slow path are two sibling branches and one omits a check. That skill diffs the branches; this one diffs the *context* of a single verdict across two uses.
- [race-condition](../race-condition/SKILL.md) — when the context change is concurrent (TOCTOU) rather than ordering or height.
- [failure-path-residue-diff](../failure-path-residue-diff/SKILL.md) — when the stale state is left behind by a failed operation rather than cached by a successful one.
- [dedup-one-fix-test](../dedup-one-fix-test/SKILL.md) — run before writing up.

## 1. THE SHAPE, FROM PUBLIC INCIDENTS

| Incident | Cached or skipped verdict | Missing key component | How the gap was spent |
|---|---|---|---|
| Zebra (Zcash), CVE-2026-40880, [GHSA-xvj8-ph7x-65gf](https://github.com/ZcashFoundation/zebra/security/advisories/GHSA-xvj8-ph7x-65gf) | Block verification skipped re-verifying transactions already accepted into the mempool | Height-dependent rules: expiry height, lock time, the network upgrade the transaction is bound to; also authorisation data | A miner submits a transaction expiring at H+1, mines H+1 and H+2 with the transaction in H+2, and delivers H+2 **before** H+1. The vulnerable node commits both and forks itself off. Fix: the optimisation was **removed**; skipping now requires a full integrity match including authorisation data. |
| Bitcoin Core, CVE-2018-17144, [notice](https://bitcoincore.org/en/2018/09/20/notice/) | The duplicate-input check was dropped from block validation for performance, on the assumption that a later stage would catch it | The later stage only asserted that the spent output *existed*, not that it was *unspent* | 0.14: crash (DoS). 0.15–0.16.2: when the output came from an earlier block the duplicate spend went unnoticed, an inflation bug. Fixed in 0.16.3. |
| Stellar Protocol 23 state archival, [post-mortem](https://stellar.org/blog/foundation-news/state-archival-issue-post-mortem) | Eviction checked expiry against the **latest** TTL; a check on the entry data had been removed earlier for performance when it was harmless | The entry was read from a different, older level of the store than the TTL | 478 entries archived with stale contents, undetected for 35 days. The change "appeared surgical" and had one reviewer; the multi-version test did not check restored contents. Fix: load the latest version of the entry with the TTL. |

Three different systems, one shape: a decision made once is reused where one of its inputs has moved, and the reuse was introduced or widened by a performance change.

## 2. THE METHOD

### 2.1 Inventory every place the system skips work it has "already done"

There is no common name, so search by intent, not by token. Candidates:

- memo maps, LRU caches, `seen`/`known`/`verified`/`valid` sets, bloom filters of accepted ids;
- "if already in the pool/mempool/queue, skip verification" branches;
- checks moved from an early stage to a later one "because the later stage covers it";
- reads that combine fields from two snapshots, two levels, two replicas, or two versions (the Stellar case is a cache in all but name);
- precomputed results stored on the object itself (a `checked: true` flag, a cached hash, a stored verdict column).

For each, note the commit that introduced it. Performance commits are the highest-yield entry in the inventory.

### 2.2 Write down the verdict's full input set

For the original, uncached check, list **every** input it reads, including the ambient ones:

| Input class | Typical members |
|---|---|
| Object content | payload bytes, and also the parts excluded from the id (witness or authorisation data, signatures, fee fields) |
| Position | height, epoch, time, sequence number, parent or predecessor |
| Rule set | active upgrade or feature flags, protocol version, configuration |
| Caller context | which account or role asked, which entry point, which tenant |
| State read | balances, existence and spent-ness of referenced objects, the version of each entry read |

Then list what the cache key and the skip condition actually contain. The difference between the two lists is the attack surface. If the key is an identifier that excludes part of the content (a common pattern: an id that excludes signatures or witness data), the authorisation data is already outside the key.

### 2.3 Build the two-context ordering

For each missing component, construct a sequence where the verdict is **warmed in context A and spent in context B**, with B chosen so the full check would fail:

- *Position*: accept an object while it is valid, then deliver a container where it is not (Zebra: out-of-order block delivery; also reorgs, catch-up after downtime, replay from an older snapshot).
- *Caller*: warm through a privileged or lenient entry point, spend through a strict one.
- *State*: warm, then change the referenced state (spend it, delete it, overwrite it with a new version), then spend the verdict.
- *Ordering*: deliver dependencies in a different order than they were produced. Ask what a node that is catching up sees compared with a live node; the cache is usually warm on one and cold on the other, which is how a single-node bug becomes a disagreement between nodes.

Who controls the context matters for severity. In the Zebra case the miner chose the delivery order. Write down which party picks each context component.

### 2.4 Check the later-stage backstop for what it really asserts

When a check was removed because "stage N catches it", read stage N and write its actual predicate. The Bitcoin case is the template: the backstop checked a weaker property (exists) than the removed check (unspent), and a later redesign weakened it further without anyone revisiting the removed check. An `assert` used as the backstop is itself a signal: the author believed the case unreachable.

### 2.5 Test the edges a reviewer will skip

- Multiple versions of one entry, and a test that checks the **contents** restored, not just the count (the Stellar test missed exactly this).
- The cache cold versus warm: run the same sequence twice, once after a restart. A verdict that differs between the two runs is a finding on its own in any system where nodes must agree.
- Eviction: what happens at the cache limit; whether an evicted entry is re-verified or silently treated as valid.

## 3. PROVING IT

- Prefer a differential harness to a narrative: the same sequence of inputs through (a) the system as built and (b) the system with the cache disabled or the skip removed. Any divergence in accept or reject is the evidence. This proof is port-free and needs no network.
- If the system is distributed, show the disagreement between a warm and a cold instance on the same input. That turns "a stale cache" into "two honest nodes disagree", which is the impact that matters.
- State the scope of each proof. A harness that stubs the network proves ordering, not reachability; say who can deliver the out-of-order input and at what cost.

## 4. REPORTING

1. Name the shortcut and the commit or release that introduced it, if it was a performance change.
2. Show the original verdict's input list next to the key, with the missing component highlighted.
3. Give the two-context sequence and who controls each step.
4. Show the differential (cache on versus off, or warm versus cold node).
5. Recommend the remedy in this order of preference.

**Remediation text (paste-ready):**

> The fast path at `<location>` reuses a verdict without re-checking `<missing input>`, which the original check depends on. Either (a) remove the shortcut, as the cost saved is small compared with the consequence of a wrong verdict; or (b) make the cache key include every input the verdict depends on — at minimum `<list>` — and invalidate entries when any of them changes; and (c) add a test that warms the cache in one context and spends it in another, and a test that compares restored or reused contents, not just counts. Where a check was moved to a later stage, assert in that stage the same predicate the removed check enforced, not a weaker one.

Removal is the honest default. Zebra's maintainers removed the optimisation rather than patching the key, and that is usually right when the key cannot be shown complete.
