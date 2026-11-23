# Non-Greppable Bug Classes

Seven audit skills for targets where every sink-shaped bug is already filed.

Each one finds defects that have **no searchable token**. A grep for `bash -c`, an unquoted variable, a predictable temp path or a permissive file mode will not surface them, because the vulnerable thing is prose, an absence of ordering, or the composition of two individually correct settings. They are written for crowded bug bounty targets and for code review on mature codebases, where the greppable surface has already been mined.

| Skill | Finds |
|---|---|
| [`agent-instruction-file-injection`](skills/agent-instruction-file-injection/SKILL.md) | Agent instruction files (`SKILL.md`, `AGENTS.md`, `CLAUDE.md`, MCP tool descriptions) treated as vulnerable code — a shell template in a Markdown fence is a sink, and the model is the string-concatenation step |
| [`invariant-asymmetry-audit`](skills/invariant-asymmetry-audit/SKILL.md) | A safety invariant that one branch of a function implements and its sibling omits — ordering bugs in multi-platform code, install flows and privileged state machines |
| [`interrupted-state-audit`](skills/interrupted-state-audit/SKILL.md) | The intermediate states of a privileged state machine — installers, updaters, resets, key rotations — treated as attacker-reachable configurations rather than transients |
| [`dedup-one-fix-test`](skills/dedup-one-fix-test/SKILL.md) | Whether an escalated finding is actually yours to report, by asking whether one fix closes both your finding and the prior art |
| [`verdict-cache-key-completeness`](skills/verdict-cache-key-completeness/SKILL.md) | Caches, memos and "already verified" fast paths whose key omits an input the original verdict depended on — height, upgrade, authorisation data, caller, or the version of the state it read |
| [`failure-path-residue-diff`](skills/failure-path-residue-diff/SKILL.md) | What a failed, cancelled or refunded operation leaves behind — a stuck flag, a second charge, a misrouted refund — found by fault-injecting every exit and diffing all state against the pre-state |
| [`observer-authentication-audit`](skills/observer-authentication-audit/SKILL.md) | Off-chain relayers, validators and verifiers that act on chain facts, and the lookalike event, log or message that parses as real because one authenticator (event identity, discriminator, finality, a second source) is not checked |

## Why these seven

**Agent instruction files.** Products increasingly ship capability as instructions rather than code: skill files, hooks, tool descriptions, playbooks. Those files contain lines like *"Run `osquery --json "SELECT * FROM processes WHERE name = '<procname>'"` for the process the alert names. Quote the name."* The product's own code can be flawless — argv stays discrete, JSON is validated — because the injection happens one hop later, inside a step that exists only as English, where escaping is delegated to a non-deterministic component. No linter greps documentation for shell templates and no taint tracker models an LLM as `sprintf`.

**Invariant asymmetry.** Mature codebases grow variants: x86 against ARM, vendor against vendor, UEFI against BIOS. Each is written at a different time by a different person, and the security-critical ordering established in one is often not carried into the other, because nobody diffs them. A missing *check* is greppable; a missing *invariant* is an absence of ordering, and absence has no token to search for. The highest-signal tell is a comment in the safe branch stating the invariant in general terms while the unsafe sibling quietly does the opposite — and a test suite that asserts it for only one branch, which makes the asymmetry read as deliberate to every later reviewer.

**The one-fix test.** The dedup question is *"would a single fix close both?"*, not *"is my impact novel?"* Novel impact raises severity on a new report; it does not turn a known defect into a new report. This skill exists because getting that backwards costs a validity-ratio hit for zero return, and the worked example in it is a real closed-as-duplicate.

**Interrupted state.** A multi-step privileged operation is a sequence of states, and the ones in the middle are real configurations a machine can sit in indefinitely. The literature on interrupted updates treats power loss as a *reliability* problem — atomicity, rollback, anti-downgrade counters — while the update-security corpus models a hostile network and a hostile repository and assumes the client applies updates atomically. Neither asks what an attacker can do against the machine if it stops at step four of seven. These flows run precisely on handover events, where the person who ends up with the device is the attacker in the threat model and the flow existed to make the handover safe. The tell is a cleanup handler installed on a signal that power loss does not deliver.

**Verdict cache keys.** Validating systems save time by not doing work twice: a transaction already accepted into a pool, a check moved to a later stage "which covers it", an expiry computed from the newest record. Each shortcut is a verdict reused in a context its author did not picture, and it is only safe if its key holds every input the verdict depended on. Zebra's mempool shortcut ignored height-dependent rules and let a miner fork nodes by delivering blocks out of order; Bitcoin Core's removed duplicate-input check left a backstop that tested a weaker property; Stellar's archival read the newest TTL next to an older entry. No line is wrong and there is nothing to grep — the defect is a missing component of a key. The proof is a differential: the same inputs with the shortcut on and off, or through a warm node and a cold one.

**Failure-path residue.** Tests and reviews follow the success path and treat a failed operation as a no-op plus a fee. Nobody diffs the world around a failure, so a flag left set or a charge applied twice surfaces later in an unrelated operation that cannot be traced back. A stuck conversion flag in the Soroban host broke an ordinary event after a caught failure in a callee; Sui charged gas on a cancelled transaction and halted twice, the second time because the interim fix covered one cancellation reason and not two together; a Stellar refund of unused fees on sponsored transactions paid the wrong party. The method is mechanical: force every exit, diff everything, then run a second operation in the same context.

**Observer authentication.** Bridges and relayers increasingly lose money with correct contracts, because the decision to pay is made by an off-chain process that believed it saw an event. That belief rests on a few bytes — an event signature hash, an 8-byte discriminator, a confirmation depth, a second data source — and a decoder that matches by shape accepts a lookalike when one is skipped. Polygon's validator software parsed a signer-change event as a stake update; an Across relayer filled 581 forged deposits from inner instructions without a discriminator check; a single-verifier pathway released hundreds of millions on a message that was never sent. Contract audits pass because the vulnerable code is in a service nobody lists as an entry point. The skill decomposes each belief into authenticators and builds the cheapest object that satisfies the ones that are checked.

## Install

These are [Claude Code](https://claude.com/claude-code) skills (plain Markdown with YAML frontmatter, so they port to any agent harness that reads skill files).

**User-level**, available in every project:

```sh
git clone https://github.com/mihaimacarie98/non-greppable-bug-classes.git
cp -r non-greppable-bug-classes/skills/* ~/.claude/skills/
```

**Project-level**, checked in alongside a repo:

```sh
mkdir -p .claude/skills
cp -r non-greppable-bug-classes/skills/* .claude/skills/
```

Each skill then loads by name. Other harnesses use their own path — `.agents/skills/` for Codex, `.cursor/skills/` for Cursor — and the files are unchanged.

## Scope and intent

For testing systems you own or are authorized to test.

No skill describes an unpatched defect or a target under test. Examples drawn from private work are generalised: the target is removed and only the shape of the mistake remains. Three skills (`verdict-cache-key-completeness`, `failure-path-residue-diff`, `observer-authentication-audit`) instead cite public incidents by name, each already fixed and disclosed by the vendor or analysed in a linked public write-up.

The remediation guidance in each skill is written to be pasted into a report, so these are as useful to someone hardening a codebase as to someone auditing one.

## License

MIT
