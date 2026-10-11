# Non-Greppable Bug Classes

Four audit skills for targets where every sink-shaped bug is already filed.

Each one finds defects that have **no searchable token**. A grep for `bash -c`, an unquoted variable, a predictable temp path or a permissive file mode will not surface them, because the vulnerable thing is prose, an absence of ordering, or the composition of two individually correct settings. They are written for crowded bug bounty targets and for code review on mature codebases, where the greppable surface has already been mined.

| Skill | Finds |
|---|---|
| [`agent-instruction-file-injection`](skills/agent-instruction-file-injection/SKILL.md) | Agent instruction files (`SKILL.md`, `AGENTS.md`, `CLAUDE.md`, MCP tool descriptions) treated as vulnerable code — a shell template in a Markdown fence is a sink, and the model is the string-concatenation step |
| [`invariant-asymmetry-audit`](skills/invariant-asymmetry-audit/SKILL.md) | A safety invariant that one branch of a function implements and its sibling omits — ordering bugs in multi-platform code, install flows and privileged state machines |
| [`interrupted-state-audit`](skills/interrupted-state-audit/SKILL.md) | The intermediate states of a privileged state machine — installers, updaters, resets, key rotations — treated as attacker-reachable configurations rather than transients |
| [`dedup-one-fix-test`](skills/dedup-one-fix-test/SKILL.md) | Whether an escalated finding is actually yours to report, by asking whether one fix closes both your finding and the prior art |

## Why these four

**Agent instruction files.** Products increasingly ship capability as instructions rather than code: skill files, hooks, tool descriptions, playbooks. Those files contain lines like *"Run `osquery --json "SELECT * FROM processes WHERE name = '<procname>'"` for the process the alert names. Quote the name."* The product's own code can be flawless — argv stays discrete, JSON is validated — because the injection happens one hop later, inside a step that exists only as English, where escaping is delegated to a non-deterministic component. No linter greps documentation for shell templates and no taint tracker models an LLM as `sprintf`.

**Invariant asymmetry.** Mature codebases grow variants: x86 against ARM, vendor against vendor, UEFI against BIOS. Each is written at a different time by a different person, and the security-critical ordering established in one is often not carried into the other, because nobody diffs them. A missing *check* is greppable; a missing *invariant* is an absence of ordering, and absence has no token to search for. The highest-signal tell is a comment in the safe branch stating the invariant in general terms while the unsafe sibling quietly does the opposite — and a test suite that asserts it for only one branch, which makes the asymmetry read as deliberate to every later reviewer.

**The one-fix test.** The dedup question is *"would a single fix close both?"*, not *"is my impact novel?"* Novel impact raises severity on a new report; it does not turn a known defect into a new report. This skill exists because getting that backwards costs a validity-ratio hit for zero return, and the worked example in it is a real closed-as-duplicate.

**Interrupted state.** A multi-step privileged operation is a sequence of states, and the ones in the middle are real configurations a machine can sit in indefinitely. The literature on interrupted updates treats power loss as a *reliability* problem — atomicity, rollback, anti-downgrade counters — while the update-security corpus models a hostile network and a hostile repository and assumes the client applies updates atomically. Neither asks what an attacker can do against the machine if it stops at step four of seven. These flows run precisely on handover events, where the person who ends up with the device is the attacker in the threat model and the flow existed to make the handover safe. The tell is a cleanup handler installed on a signal that power loss does not deliver.

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

Every example in these skills is generalised. None names a target, a vendor, or an unpatched defect. Where a real case is used to illustrate a technique, the target is removed and only the shape of the mistake remains.

The remediation guidance in each skill is written to be pasted into a report, so these are as useful to someone hardening a codebase as to someone auditing one.

## License

MIT
