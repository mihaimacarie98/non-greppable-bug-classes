---
name: interrupted-state-audit
description: >-
  Find security bugs in the intermediate states of privileged state machines — installers,
  updaters, resets, migrations, key rotations, onboarding flows. Use when auditing any
  multi-step privileged operation that stages a credential, publishes an artifact, or
  switches an active target, and ask what an interruption leaves behind.
---

# SKILL: Interrupted State Audit — The Intermediate State As An Attacker-Reachable Configuration

> **AI LOAD INSTRUCTION**: A multi-step privileged operation is not one state and then another state. It is a *sequence* of states, and the ones in the middle are real configurations a machine can be left sitting in indefinitely. Audit them as deployment states, not as transients. Base models read such a flow start-to-end, confirm it reaches a correct final state, and never ask what the machine looks like if it stops at step 4 of 7 — which is the only question that matters here.

## 0. RELATED ROUTING

- [invariant-asymmetry-audit](../invariant-asymmetry-audit/SKILL.md) — when two sibling branches implement the *same* flow and only one gets the ordering right. That skill diffs branches; this one walks a single branch's interruption points.
- [race-condition](../race-condition/SKILL.md) — when the second actor is concurrent rather than an interruption.
- [dedup-one-fix-test](../dedup-one-fix-test/SKILL.md) — run before writing any of this up.

## 1. WHY THIS CLASS IS UNDER-HUNTED

There is a large literature on interrupted updates, and essentially all of it treats power loss as a **reliability** problem. The A/B-update and crash-consistency corpus is about atomicity, rollback and anti-downgrade counters: did the device come back up, did it come back up on a version no older than before. The update-security corpus (TUF and successors) models a hostile *network* and a hostile *repository*, and assumes the client applies updates atomically.

Neither asks the security question. Nobody models the intermediate states of an update, reset or provisioning flow as **configurations an attacker can reach and then act against**. Likewise, the secrets-in-artifacts literature — Dahlmanns et al., *Secrets Revealed in Container Images* (ACM AsiaCCS 2023), which found secrets in 8.5% of 337k Docker Hub images — established the class for distributed container images, and has no counterpart for boot artifacts, install media, or device-handover state.

So there is a real gap: a flow can be correct at every endpoint, reliability-safe under interruption, and still spend a window of seconds to minutes in a state that is a serious vulnerability if the machine stops there. That window has no CVE class, no linter, and no token to grep for.

The commercial consequence: these flows run on *handover* events — a device sold, returned, RMA'd, re-provisioned, handed to a new employee. The person who ends up with the machine is exactly the attacker in the threat model, and the flow's whole purpose was to make the handover safe.

## 2. THE METHOD

### 2.1 Reduce the flow to an ordered list of security-relevant operations

Discard logging, progress output and cosmetics. Keep only operations that change what an attacker can do:

- credential created / enrolled / published / destroyed
- secret written to disk, and *which* filesystem (encrypted? removable? unauthenticated?)
- artifact published to a location readable without authorization
- the active target switched (which root boots, which config is live, which key is canonical)
- authentication requirement added or removed
- cleanup handler installed or torn down

Number them. You now have N states, and N-1 gaps.

### 2.2 For each gap, write the configuration down as if it were a shipped product

This is the discipline that does the work. For gap *i*, answer concretely:

| Question | Why it matters |
|---|---|
| What credentials are valid here? | Enrolled-but-unpublished is usually harmless; enrolled-and-published is usually the bug |
| Where is each secret readable from, and by whom with what authorization? | A key on an unauthenticated partition is a different finding from one on an encrypted volume |
| Which target is active — the old one or the new one? | Publishing the new target's credentials while the *old* target is still active is the classic defect |
| What authentication does a person at the machine face? | If a prompt that normally gates access is gone here, that is the impact |
| Whose data is still present? | An interrupted wipe that has published credentials but not yet moved data is the worst case |

If the honest answer to the last four is "the new system's key is live, readable without authorization, and the old owner's data is still mounted behind it," you have a finding, and you should be able to state it in one sentence.

### 2.3 Enumerate what can actually interrupt, and what each one skips

Most authors test with Ctrl-C, which runs their cleanup handler, so their mental model is that cleanup always runs. It does not:

- `SIGINT` / `SIGTERM` — EXIT/`trap` handlers **do** run. A compensating cleanup works here. This is the case the author tested.
- **`SIGKILL`, power loss, battery exhaustion, held power button, kernel panic, thermal shutdown, OOM kill** — handlers do **not** run. This is the case that matters and the one that is almost never tested.
- Container stop, forced reboot during the operation, hypervisor reset.
- A plain error return between two operations with no rollback path.
- Hardware removal: the disk, the dock, the USB target pulled mid-operation.

Then ask the question that converts an exotic-sounding interruption into an ordinary one: **what does the product do to prevent an interruption during this window?** Look specifically for a suspend/idle inhibitor. If a sibling flow in the same codebase takes the trouble to inhibit sleep and this one does not, closing a laptop lid is now a supported way to reach your intermediate state, and you have pre-empted the only serious objection to the finding. That absence is also greppable, which makes it cheap to check.

### 2.4 Check whether anything repairs it later

A flow often installs self-healing machinery — a first-boot service, a pending marker, a verification unit. Establish precisely **which copy of the system** that machinery was installed into. If the repair was staged into the new target but the interruption left the *old* target active, the repair is sitting in a filesystem that never boots, and nothing runs it.

Then bound the duration honestly. Something unrelated may clear the state incidentally — a later package upgrade that rebuilds the same artifact, a scheduled job, a reconciliation loop. Say "persists until X happens, which on a machine being handed on is plausibly never" rather than "persists permanently." A reviewer who finds one path that clears it will discard the whole finding if you claimed permanence.

## 3. PROVING IT

Two proofs, kept separate. Do not let one bleed into the other — a reviewer who catches you presenting a stub's output as a measurement will distrust everything else.

**Proof of ordering** — that the state is reachable. Extract the real function bodies from the real script, source them, stub the destructive externals, and replace the operation at your candidate interruption point with `kill -KILL $$`. Then inspect what is on disk. This proves control flow and resulting state. It proves nothing about cryptography, and you must say so.

```sh
# in a disposable container, against a copy, never the host
awk '/^target_function\(\)/,/^}/' ./real-script > fns.sh
. ./fns.sh
verify_step() { kill -KILL $$; }   # power loss at a real instruction boundary
target_function "$fixture"; echo "exit $?"   # 137 = SIGKILL
```

**Proof of consequence** — that the state is harmful. Use real tooling, no stubs: build the real artifact with the real builder, put it on the real filesystem type, and read it back the way someone with the device would, using no credential. For a boot artifact that means carving the section out and unpacking it:

```sh
objcopy -O binary --only-section=.initrd artifact.efi initrd.raw
# unpack with the tool that understands the concatenated-archive layout rather
# than guessing at the member boundary -- a byte-offset search for a compression
# magic will hit a false positive inside the uncompressed early member
```

Two traps worth naming, because both produce confident false negatives:

- **Searching a compressed artifact for a plaintext secret finds nothing, and that is not evidence of absence.** Decompress first, then look.
- **A privileged loop mount is often unavailable in a container.** Read the filesystem image with userspace tools (`mtools` for FAT, `debugfs` for ext) instead, which is also closer to what an attacker with the raw disk does.

## 4. REPORTING

Lead with the ordering, because that is what makes it a defect rather than a misconfiguration:

1. The sequence, with exact line numbers, showing the publish happening before the switch.
2. The interruption class that defeats the cleanup — name the signal and say that handlers do not run for it.
3. The intermediate configuration, written out as a state.
4. The observed post-interruption evidence, with each proof's scope stated.
5. What the product itself promises, quoted from its own documentation or code comments. If the flow's purpose is safe handover and the defect defeats exactly that, say so plainly — this is usually the strongest sentence available.
6. Whether anything repairs it, and the honest duration.

**Score it conservatively and argue impact in prose.** These findings usually compute to Medium: physical access is typically required, which caps the vector hard, and an interruption precondition caps it again. Reviewers downgrade inflated vectors and upgrade well-argued impact paragraphs, so keep the metrics defensible and spend the effort on the paragraph explaining that the one control the design depends on is removed by the tool meant to protect it.

Pre-empt the two certain objections explicitly:

- *"The cleanup handles it."* — Name the signal that skips the handler, and show the handler has exactly one caller reachable only from that handler.
- *"That interruption is too unlikely."* — Give the ordinary trigger, and cite the missing inhibitor alongside the sibling flow that has one.
