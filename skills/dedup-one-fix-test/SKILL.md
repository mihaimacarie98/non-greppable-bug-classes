---
name: dedup-one-fix-test
description: >-
  Decide whether a finding is yours to report when the underlying defect is already
  public. Use before submitting any report on a target with existing issues, pull
  requests, advisories or disclosed reports — especially when you have escalated a
  known low-impact bug to a higher impact and are tempted to file on novelty.
---

# SKILL: The One-Fix Test — Dedup for Escalated Findings

> **AI LOAD INSTRUCTION**: The dedup question is **"would a single fix close both?"** — not **"is my impact novel?"** Novel impact raises severity on a *new* report; it does not turn a known defect into a new report. Base models (and hunters) reason from impact novelty, file the report, and get closed as a duplicate for zero return. Run this test against the *prior art*, not just against your own findings.

## 0. RELATED ROUTING

- [triage-validation](../triage-validation/SKILL.md) — the 7-Question Gate, run *before* this.
- [report-writing](../report-writing/SKILL.md) — how to frame prior art once you decide to file.
- [dupcheck](../dupcheck/SKILL.md) — the mechanical search step this skill interprets.

## 1. THE TEST

For every piece of prior art you find:

1. Write out, in one sentence, **the fix that prior art proposes or implies**.
2. Ask whether that same fix also closes your finding.
3. If yes → you are a duplicate. Do not file on impact novelty.
4. If no → you have a genuinely separate root cause. File, and say so explicitly.

That is the whole test. It is cheap, and it is the step that gets skipped.

### Worked example (a real closed-as-duplicate)

- **Prior art:** an open community PR reports that a name containing `/` or `&` breaks a `sed -i` rewrite, leaving a config "untouched (or corrupted)". Framed purely as a cosmetic bug. Proposed fix: escape `\`, `&` and `/` before the replacement.
- **The escalation:** the same construct reaches `/bin/sh`, because GNU sed's `e` flag executes the substituted pattern space and a `/` in the data can supply the flag. Reproduced end to end: arbitrary command execution as the user.
- **One-fix test:** escaping `/` is *precisely* what removes `e`-flag reachability. One patch, three characters, closes both.
- **Outcome:** closed as duplicate. The technical work was sound and the impact genuinely unreported; it still earned nothing.

## 2. SEARCH DISCIPLINE

Search **both** sides. Issues-only is the classic miss.

```sh
gh search issues --repo <owner>/<repo> "<terms>" --limit 20
gh search prs    --repo <owner>/<repo> "<terms>" --limit 20
```

Use at least four term sets per candidate, covering:

- the **component** (filename, binary, module)
- the **mechanism** (the function, the construct, the protocol)
- the **impact** (what you claim happens)
- the **error string** a user would paste when they hit it accidentally — this finds reports from people who tripped over it without understanding it

Then widen beyond the tracker:

```sh
gh api repos/<owner>/<repo>/security-advisories --jq '.[] | "\(.ghsa_id) \(.summary)"'
```

Plus CVE aggregators, the project's security page, and a web search for the component name with "vulnerability" or the exact error text. **Open every plausible hit and read the comments**, not just the title — the review thread is where the impact is usually discussed or dismissed.

## 3. CLASSIFY THE OVERLAP

| Relationship | Verdict |
|---|---|
| Same asset, same primitive, same impact already public | duplicate — drop |
| Same asset, same primitive, your impact strictly stronger, **one fix closes both** | duplicate — comment on the existing thread instead |
| Same asset, same primitive, **different fix required** | file, and explain why the existing fix is insufficient |
| Same asset, different root cause reaching the same impact | file as separate |
| Behaviour documented and accepted in-repo (a comment or doc naming the risk) | not a vulnerability — propose hardening, do not file |

## 4. WHEN YOU ARE A DUPLICATE BUT THE ESCALATION MATTERS

A duplicate close does not mean the information is worthless — it means it has no *report* value. The escalation still changes how the vendor prioritises the fix, and delivering it costs nothing:

- **Comment on the existing report or PR** with the escalation, framed as prioritisation input, not as a severity argument.
- Mind the disclosure terms first. If the *defect* is public but the *security impact* is not, posting the impact on a public thread is a disclosure. Most programs forbid public disclosure without consent, and that obligation typically survives both a duplicate close and a patch. Use the private channel (the closed report, or the security contact) unless you have consent.
- Keep it short, accept the close without arguing severity, and give the maintainer the two or three things they would otherwise miss by taking the fix from the prior art alone.

## 5. THE CHECKLIST

Before you click submit:

- [ ] Searched issues **and** PRs, four term sets minimum, closed and merged included
- [ ] Checked advisories, CVE aggregators and the project's security page
- [ ] Opened and read every plausible hit **including comments**
- [ ] Wrote out the fix each hit implies
- [ ] Applied the one-fix test to each
- [ ] Confirmed the behaviour is not documented-and-accepted in-repo
- [ ] If duplicate: switched to a comment on the existing thread, through the channel the disclosure policy allows
