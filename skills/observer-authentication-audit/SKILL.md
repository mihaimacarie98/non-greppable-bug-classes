---
name: observer-authentication-audit
description: >-
  Find security bugs in off-chain actors that act on chain facts — relayers, bridge
  validators, oracle and DVN signers, indexers, keepers, solvers — by writing down each
  belief they form, the channel it arrives on, and what authenticates its origin, then
  crafting a lookalike that parses as the real thing but is not. Use when the contracts
  are audited and clean but money moves because an off-chain process believed an event,
  log, instruction or message happened.
---

# SKILL: Observer Authentication Audit — Lookalike Facts For Off-Chain Actors

> **AI LOAD INSTRUCTION**: In bridges and relayer systems the contracts can be correct and still lose money, because the decision to pay is made by an off-chain process that *observed* something. That process authenticates the observation with a handful of bytes — an emitter address, an event signature hash, an 8-byte discriminator, a confirmation depth, a second data source. Drop one and anything that parses becomes a fact. Contract audits and scanners pass because the missing check is in a service nobody lists as an entry point, and the decoder that skips it is a library call that "works". Base models read the decoder, see fields extracted correctly, and move on. Ask instead: **what else would this decoder accept, and who can produce it?**

## 0. RELATED ROUTING

- [invariant-asymmetry-audit](../invariant-asymmetry-audit/SKILL.md) — when two event handlers in the same service check different subsets of these properties.
- [verdict-cache-key-completeness](../verdict-cache-key-completeness/SKILL.md) — when the observer caches "already seen" facts keyed on an id that omits the source.
- [dedup-one-fix-test](../dedup-one-fix-test/SKILL.md) — and read the program scope: several programs pay for off-chain components only when they are named in scope (see section 4).

## 1. THE SHAPE, FROM PUBLIC INCIDENTS

| Incident | Belief | What authenticated it | What was missing | Lookalike |
|---|---|---|---|---|
| Polygon Heimdall log confusion, [Asymmetric Research](https://asymmetric.com/polygon-log-confusion/) (reported through the program's bug bounty, patched) | "This log is a stake update" | Emitter address and log index | **topic0** (event signature): the ABI unpack accepted any event with the same number of indexed arguments. A uint256 nonce was also compared after silent truncation to 64 bits | A signer-change event from the same contract, with a ground address whose low bytes match the expected nonce — inflate stake, then approve fake deposits |
| Across Solana relayer, July 2026, [Cryptonomist](https://en.cryptonomist.ch/2026/07/24/across-solana-relayer-attack/) | "This inner instruction is a deposit event" | The program the instruction targeted | The **8-byte Anchor event discriminator** | 1,627 forged deposits, 581 filled, about USD 4.5M gross of relayer capital lost (under USD 4M net). No contract was exploited |
| KelpDAO / LayerZero, April 2026, [Hypernative](https://www.hypernative.io/blog/the-kelpdao-observation-layer-exploit-291m-released-on-a-message-that-never-existed) (vendor analysis) | "A send event exists on the source chain" | A configuration that required a single verifier network (two of its three signers signed), 42 L2 blocks of confirmation | **A second, independently operated verifier**, existence of the source transaction, finality depth, supply reconciliation | A message that was never emitted; about USD 291M released from escrow, roughly 2,300 times the source-chain supply |

Each is a different missing property — event identity, event discriminator, observation independence — on the same question: *how does this process know the fact is real?*

## 2. THE METHOD

### 2.1 Inventory beliefs, not endpoints

List every decision the off-chain actor takes that moves value or signs something (fill an order, release funds, sign an attestation, mint, post a dispute, liquidate). For each, write the belief it rests on as one sentence: *"Contract X on chain Y emitted event E with fields F, in a block that will not be reverted."*

### 2.2 Decompose each belief into authenticators

For each belief, record what the code actually checks:

| Property | Question | Typical check |
|---|---|---|
| Source | Is it from the right contract or program? | emitter address, program id, and for proxies the address that is actually stored |
| Identity | Is it this kind of event and not a sibling? | topic0 / event signature hash, Anchor discriminator, message type tag |
| Carrier | Is it a real event, or something that merely looks like one in the same container? | top-level log vs inner instruction vs CPI vs trace entry; whether a helper or arbitrary call can place bytes there |
| Integrity | Do decoded fields mean what the consumer thinks? | full-width integer comparison, canonical encoding, no silent truncation or defaults on missing fields |
| Finality | Can it still disappear? | confirmation depth against the chain's real finality, handling of removed or reorged logs |
| Independence | Would a lying data source be caught? | second RPC or node, a second verifier from another operator, reconciliation against source supply |

Each blank cell is a candidate. Library calls that "decode a log against an ABI" deserve particular suspicion: several decode by shape and never compare the signature.

### 2.3 Craft the lookalike

For each missing property, construct the cheapest object that passes every check the code does make:

- *Identity missing*: emit a sibling event from the same contract with the same indexed-argument count (Heimdall).
- *Discriminator or carrier missing*: get bytes into the observed container from a path the attacker controls — an arbitrary call, a read-only helper, an inner instruction (Across).
- *Integrity*: choose field values that collide after truncation or after default-filling.
- *Finality*: target the window between the configured depth and real finality.
- *Independence*: identify the single data source and ask what it takes to make it lie (compromised or misbehaving RPC, pipeline injection); this is often out of scope as an attack, but the missing second source is a reportable design gap when a program says so.

Then trace the lookalike through to the value-moving action and note the first check that would stop it. If none does, the finding is complete in design; prove it next.

### 2.4 Check the sibling handlers

When the service handles several event types, the checks are usually written per handler. Diff them. The handler written last or for a newer chain is the likeliest to be missing a property its siblings check.

## 3. PROVING IT SAFELY

- Never forge events against a live system. Use a local chain or devnet with the real contracts deployed, run the real off-chain service against it, emit the lookalike, and capture the service's log or the transaction it builds. The action it *attempts* is the evidence.
- A decoder-level unit test is a good first proof: feed the real decoding function a sibling event or a forged carrier and show it returns a populated "real" object.
- State clearly whose funds are at risk: user funds held by contracts, or the operator's own capital. This decides payability in most programs.

## 4. REPORTING

1. The belief, in one sentence, and the value-moving action it gates.
2. The authenticator table for that belief, with the missing property marked.
3. The lookalike and how an unprivileged attacker produces it.
4. The trace from lookalike to action, with the service log or built transaction.
5. Whose funds are exposed.

Check scope before writing. Public precedent is mixed: the Heimdall issue went through a bug bounty program that covered the validator software; the Across loss was relayer capital handled as an incident; many programs cover off-chain components only when they are named in scope. Quote the program's scope line that covers the component.

**Remediation text (paste-ready):**

> The off-chain component at `<location>` treats `<object>` as `<event>` after checking only `<checks present>`. Authenticate every observed fact on all of: emitter or program id, event identity (signature hash or discriminator), carrier type (reject inner or nested instructions unless they are the documented event path), full-width field comparison with no truncation or defaulting, and confirmation at the chain's actual finality. For value above a set threshold, require a second independent observation (a separate node provider or a verifier run by a different operator) and reconcile released value against source-side supply before acting.
