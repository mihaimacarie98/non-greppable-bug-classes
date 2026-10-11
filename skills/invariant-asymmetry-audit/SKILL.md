---
name: invariant-asymmetry-audit
description: >-
  Find security bugs by diffing sibling code paths for a safety invariant that one
  branch implements and another omits. Use when auditing multi-platform or
  multi-variant code, privileged state machines, install and reset flows, or any
  function with per-architecture or per-vendor branches — especially where the test
  suite asserts the invariant for only one branch.
---

# SKILL: Invariant Asymmetry Audit — Diffing Sibling Paths for a Missing Safety Property

> **AI LOAD INSTRUCTION**: Look for a safety property that **one branch of the same function implements and its sibling does not**. The code is individually correct everywhere, so no grep, linter or taint tracker fires. The highest-signal tell is a **comment in the safe branch stating the invariant in general terms** while the unsafe branch quietly does the opposite — and a test suite that asserts the invariant only for the safe branch, which makes the asymmetry read as deliberate to every subsequent reviewer. Base models audit each branch in isolation and never compare them.

## 0. RELATED ROUTING

- [race-condition](../race-condition/SKILL.md) — when the missing property is atomicity under concurrency rather than ordering under failure.
- [business-logic-vulnerabilities](../business-logic-vulnerabilities/SKILL.md) — state-machine reasoning for application workflows.
- [config-composition-audit](../config-composition-audit/SKILL.md) — the configuration-level analogue of this technique.

## 1. THE SHAPE

Mature codebases grow variants: x86 versus ARM, vendor A versus vendor B, UEFI versus BIOS, cloud versus on-prem, legacy versus modern. Each variant is usually written at a different time by a different person. The security-critical ordering established in one is often **not carried into the other**, because nobody diffs them — they are read as two independent implementations of the same feature.

This is distinct from a missing check. A missing check is greppable. A missing *invariant* is an absence of ordering, and absence has no token to search for.

Canonical examples of the invariant that goes missing:

| Invariant | Safe ordering | Unsafe sibling |
|---|---|---|
| Crash safety / power-loss safety | switch the live target **before** enrolling the new credential | enroll first, switch later — an interruption leaves both live |
| Fail-closed | write the restrictive state first, relax after success | relax first, restrict after success |
| Secret lifetime | stage the secret, consume it, erase before publishing | publish the artifact containing the secret, erase later |
| Commit-then-advertise | make the change, then record it as done | record as done, then make the change |
| Revoke-before-grant | retire the old credential before the new one is usable | grant then retire, with no barrier between |

## 2. THE METHOD

### 2.1 Enumerate the branch points

```sh
grep -rnE 'case .*(x86|aarch64|arm|apple|amd|intel|uefi|bios|secure.?boot)|if .*platform|hw-(x86|aarch64|apple)' \
  --include='*.sh' --include='*' . | grep -v test
```

Also: functions whose name ends in a variant, duplicated blocks with one differing call, and anything dispatched through a platform helper.

### 2.2 Normalise the two branches side by side

Extract each branch to a file and reduce both to an **ordered list of security-relevant operations only**: credential created, credential destroyed, state file written, artifact published, target switched, trap installed, lock taken. Discard logging and cosmetics. Then diff the *order*, not the text.

```sh
sed -n '<start>,<end>p' file > /tmp/a   # branch A
sed -n '<start>,<end>p' file > /tmp/b   # branch B
diff <(grep -nE 'luksAddKey|luksKillSlot|install -|mv -|rm -|trap |flock|systemctl|printf.*>' /tmp/a) \
     <(grep -nE 'luksAddKey|luksKillSlot|install -|mv -|rm -|trap |flock|systemctl|printf.*>' /tmp/b)
```

### 2.3 Read the comments in the safe branch as a specification

This is the single highest-yield step. When a branch carries a comment like *"a power loss before the switch leaves the old target asking for its password, never unlocked unattended"*, you have been handed the invariant in the author's own words. Treat it as a requirement and check the sibling against it. If the sibling violates it, the author has already conceded the severity for you — quote the comment in your report.

### 2.4 Check the test suite for the tell

```sh
grep -rn '<invariant keyword>' test/ | sort
```

If tests assert the strong ordering for one branch and only a weaker property for the other, that asymmetry is why the bug survived review: every reader after the author assumed the difference was intentional. Say this explicitly in the report — it answers "how did this get missed?" before the triager asks.

### 2.5 Find the interruption that makes it bite

An ordering bug needs an interruption to become an impact. Enumerate what can interrupt and what each one skips:

- `SIGINT` / `SIGTERM` — EXIT traps **do** run, so a compensating cleanup works.
- `SIGKILL`, power loss, battery pull, kernel panic, thermal shutdown — traps do **not** run. This is the case that matters, and it is the one authors forget because their testing uses Ctrl-C.
- OOM kill, container stop, forced reboot during an update.
- A failure return between the two operations with no rollback.

Prove the window exists by instrumenting the real code: insert a `kill -9 $$` at the candidate point in a **copy** of the script, run it in a disposable container, and inspect the resulting state.

## 3. PROVING IT SAFELY

Use a disposable container with the real tooling rather than reasoning about it:

```sh
docker run --rm -it archlinux:latest bash -lc '
  pacman -Sy --noconfirm cryptsetup >/dev/null
  # build a loop-backed LUKS container, run the extracted branch,
  # kill at the window, then inspect which keyslots remain enrolled
'
```

The deliverable is a state inspection after the interruption: which credential is still valid, which artifact still contains the secret, what an attacker who now has the device can do. A keyslot listing or a hexdump of the published artifact is far stronger evidence than a narrative.

## 4. REPORTING

Lead with the asymmetry, not the mechanism — it is what makes the finding credible and non-duplicate:

1. Quote the safe branch's comment as the stated invariant.
2. Show the sibling's ordering violating it, with exact line numbers for both.
3. Name the interruption class that defeats the compensating cleanup (usually: traps do not run on SIGKILL or power loss).
4. Show the post-interruption state you observed.
5. Note the test-coverage asymmetry as the reason it survived review.

Expect two pushbacks and pre-empt both: *"the cleanup handles it"* — answer with the signal that skips the trap; *"this is an unlikely interruption"* — answer with the realistic trigger (a laptop running out of battery mid-reset is not exotic).
