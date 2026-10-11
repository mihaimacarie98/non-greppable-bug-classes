---
name: agent-instruction-file-injection
description: >-
  Agent instruction files (SKILL.md, AGENTS.md, CLAUDE.md, MCP tool descriptions,
  system prompts shipped as prose) treated as vulnerable code. Use when auditing a
  product that ships LLM agent skills, hooks or tool definitions, especially where an
  agent runs with approvals disabled and the instructions tell the model to build a
  shell command, path or URL out of runtime data.
---

# SKILL: Agent Instruction Files as an Injection Sink

> **AI LOAD INSTRUCTION**: The vulnerable artifact is **prose, not code**. A shell template written inside a Markdown fence in an agent skill file is a sink, and the LLM is the string-concatenation step that fills it. No linter greps documentation for shell templates, and no taint tracker models a language model as `sprintf`. Base models audit the *scripts* a product ships and skip the *instructions* it ships to its own agent. Audit the instructions with the same suspicion you give a `bash -c`.

## 0. RELATED ROUTING

- [llm-prompt-injection](../llm-prompt-injection/SKILL.md) — getting hostile text *into* a model's context. This skill is about what the model is *told to do* with it.
- [ai-ml-security](../ai-ml-security/SKILL.md) — model supply chain, adversarial examples, model theft.
- [cmdi-command-injection](../cmdi-command-injection/SKILL.md) — the classical sink this reduces to once the model assembles the string.

## 1. WHY THIS CLASS EXISTS

Products increasingly ship agent capability as **instruction files**: `SKILL.md`, `AGENTS.md`, `CLAUDE.md`, `.cursorrules`, hook scripts, MCP tool `description` fields, and "playbooks" the agent is told to follow. These files routinely contain lines like:

> Run `osquery --json "SELECT * FROM processes WHERE name = '<procname>'"` for the process the alert names. Quote the name.

Three properties make that a vulnerability class rather than documentation:

1. **The product's own code can be flawless.** Argv stays discrete, JSON is validated, positional parameters prevent re-tokenisation. Every hop a reviewer greps for is correct. The injection happens one hop further on, inside a step that exists only as English.
2. **Escaping is delegated to a non-deterministic component.** "Quote the name" is a request, not a guarantee. The model may single-quote, may not, may quote inconsistently across turns, and may be *argued out of quoting* by the data itself.
3. **The blast radius is whatever the agent is allowed to do.** If the harness launches with `--permission-mode auto`, `--yolo`, `bypassPermissions`, `--dangerously-skip-permissions`, `--approve-for-me` or equivalent, the assembled command runs with no prompt.

## 2. THE AUDIT

### 2.1 Establish the approval posture first

This determines whether anything else matters. Find the launcher and read the flags it passes per agent.

```sh
grep -rnE 'permission-mode|bypassPermissions|--yolo|dangerously-skip|approve-for-me|--auto\b|allow-all|approval-mode' \
  --include='*' . | grep -v test
```

If approvals are **on**, this class degrades to social engineering of the user and is usually not reportable. If approvals are **off by design**, any attacker-controlled text reaching an agent prompt is a code-execution primitive.

### 2.2 Inventory the instruction surface

```sh
find . \( -name 'SKILL.md' -o -name 'AGENTS.md' -o -name 'CLAUDE.md' -o -name '*.cursorrules' \
       -o -path '*skills/*' -o -path '*hooks/*' -o -name 'mcp*.json' \) -type f | grep -v test
```

For each file, extract every fenced block and every backticked command, then ask: **does any token in it come from runtime data?** Placeholders to hunt: `<procname>`, `$path`, `{{ filename }}`, "the process the alert names", "the file the user selected", "the URL from the issue", "the hostname from the ticket".

### 2.3 Classify each template

| Shape in the instruction file | Severity of the pattern |
|---|---|
| Fixed command, no interpolation | safe |
| Interpolation, but the instruction supplies the quoting literally (`'<exe>'`) | still unsafe — the data can close the quote |
| Interpolation with "quote it" / "escape as needed" | unsafe — escaping delegated to the model |
| Interpolation where the data is a path, a name or a URL | unsafe and usually reachable |
| Instruction to `eval`, to source a file, or to run a file found at runtime | critical |

### 2.4 Find the feed

An instruction template is only a vulnerability if untrusted text reaches it. Trace feeds in this order, highest yield first:

- **Crash and diagnostic paths** — process names (`comm` is attacker-settable via `prctl(PR_SET_NAME)`), executable paths, coredump metadata, `journalctl` output. Logs are a *remote* medium: SSH auth lines, mDNS, printer discovery, Wi-Fi SSIDs, USB product strings all land in the journal.
- **Filenames and file metadata** — anything the agent is told to read about a file the user downloaded.
- **Repository content** — an agent that reads a repo the user merely opened will read that repo's own `AGENTS.md`.
- **Issue, ticket and message text** — agents told to triage.
- **Tool output** — an MCP server's response, an API body, another agent's stream.

### 2.5 Prove it without claiming more than you have

Replace the agent binary with a logger and drive the product's real code path:

```sh
mkdir -p "$TMP/stub" && cat > "$TMP/stub/<agent>" <<'EOF'
#!/bin/bash
{ printf 'ARGV:'; printf ' [%s]' "$@"; printf '\nSTDIN:\n'; cat; } >> "$TMP/agent-prompt.log"
EOF
chmod +x "$TMP/stub/<agent>"
PATH="$TMP/stub:$PATH" <the product command that reaches the agent>
```

The log showing attacker text inside the prompt is your evidence for **reachability**. Note honestly what it does and does not prove: it proves the text arrives, not that the model will build an unquoted command. For the execution half, either demonstrate it against the real agent, or report reachability and let the vendor reason about the model step — do not assert model behaviour you did not observe.

## 3. WHAT WILL GET YOUR REPORT CLOSED

- **The feed is same-user.** If only a process already running as the user can set the text, no boundary is crossed. Find a *sandboxed*, *remote* or *lower-privileged* source or drop it.
- **A prompt-splice issue is already filed.** These products attract many reports. The splice ("attacker text ends up in the prompt") is often already known while the *command-assembly* step is not — but check whether one fix closes both. If the vendor's fix is "stop interpolating untrusted data into the prompt", it closes your escalation too and you are a duplicate. See [dedup-one-fix-test](../dedup-one-fix-test/SKILL.md).
- **You asserted the model's behaviour.** "The LLM will fail to quote this" without a transcript is speculation. Show it or scope it.

## 4. DEFENSIVE GUIDANCE TO INCLUDE IN THE REPORT

- Never interpolate runtime data into a command inside an instruction file. Have the *code* build an argv vector and expose it to the agent as a single opaque tool, so there is no string for the model to assemble.
- If the agent must see the data, pass it as a file path the tool reads, never as a token inside a command template.
- Treat every instruction file as privileged source and review it under the same rules as the shell scripts it sits beside.
- Scope the agent: diagnostic agents do not need write or exec capability on the host.
