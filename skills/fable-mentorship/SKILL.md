---
name: fable-mentorship
description: >-
Mandatory process discipline for non-trivial coding and agentic tasks. Load this at the start of ANY task that involves editing files in a repo, debugging a failure, running commands with side effects, or work expected to take more than one step — even if the user doesn't ask for "process" and even if the task looks easy. Triggers include: implement, fix, refactor, migrate, set up, debug, investigate, "make it work", multi-file changes, anything touching git/bash. Do not load for pure Q&A or single-line answers.
---

# fable-mentorship

## How to use this skill

Read this file in full before your first edit. These rules are mandatory — when a rule conflicts with speed or with your confidence, the rule wins. Your confidence is not evidence; command output is evidence.

You will forget these rules as your context grows. Compensate on disk: when you create `.agent/plan.md`, copy the **LOOP card** from §3 verbatim to its top. At every checkpoint, re-read `plan.md` (which carries the card). Re-read this entire file only after a plan event (§4).

Maintain in `.agent/`: `plan.md` (numbered steps + check commands), `assumptions.md` (numbered, with probe commands), `beliefs.md` (facts, each citing the command + output that proved it), `worklog.md` (append-only, one line per action).

**Hard rule: no source edit is legal before `.agent/plan.md` exists.**

## 0. Setup — once, before anything else

1. `mkdir -p .agent` at the repo root. If in a git repo: `echo '.agent/' >> .git/info/exclude` so `.agent/` never appears in status or diffs. (Not `.gitignore` — that edits a tracked file.)
2. If git is available: `git checkout -b agent/<short-task-name>`. Local commits on this branch are safe and required at checkpoints. `git push` stays deny-listed.
3. Determine check commands from files (`package.json` scripts, `Makefile`, `pyproject.toml`, CI config) — never from assumption. Identify TEST (the full suite) and your **step-check fallback chain**: unit test > build/typecheck > lint > run the script > syntax check (`python -m py_compile`, `node --check`) > a grep or file-existence assertion. Some rung is always executable.
4. Run TEST once now. Record in `beliefs.md`: its duration, and every failure verbatim. This is the **baseline**. Everywhere below, **GREEN means: no failures beyond the baseline.** Pre-existing failures go in the handoff as a report; fix them only if the user asks or the task requires touching that code.

## 1. Task intake

**Two bins for every open question:** desired *behavior* → only the user can answer; *codebase/environment* → only the repo can answer, never ask the user what a file read would tell you. If unsure which bin: try the repo first; if the repo can't answer, it's a user question.

**Ambiguity procedure:** write plausible interpretations into `assumptions.md`. An interpretation only counts if it differs in **observable behavior or in which files get touched** — naming and wording differences don't count. If ≥2 still survive after reading the repo → ask: one batched multiple-choice message with a marked default. **One clarification round at intake, maximum.** Mid-task: at most one targeted question per new blocker, never a fresh round.

**Always-ask list** (touching these → confirm before acting): auth/authorization changes, data deletion, payments, outbound sends (email, webhooks, writes to external APIs), schema migrations, deploys.

**Context gathering, in order:** (1) grep for the symbol/route/error string — don't browse hoping. (2) Read the definition plus direct callers and callees; stop there unless a specific question forces more. (3) Find one existing example of the same kind of thing in this repo and copy its conventions — local consistency beats general best practice. (4) Write the exact TEST command into `plan.md`.

**Read-before-edit, defined:** editing an existing file requires having read, this session, the regions you'll change plus enough surrounding context to know how they're used. For files over ~500 lines or machine-generated files (lockfiles, bundles, codegen output), grep + ranged reads of the relevant regions count as reading — never read such files end-to-end. Creating a **new** file requires only confirming the path doesn't already exist (`ls`).

**Readiness test** — you may plan only when you can write: (a) files you'll touch, (b) 1–2 sentences per file on the change, (c) a check command per change. One "read operation" = one file read or one search. If unmet after ~15 read operations, the task is underspecified: stop and ask.

**Bugs: reproduce first.** No source edit until a failing command and its output are in `worklog.md`. Bound: after 3 distinct reproduction attempts, if it can't be reproduced in this environment (needs prod data, an external service, other hardware), stop and ask using the §5 escalation template. Do not fix blind; do not keep trying.

## 2. Planning

Write `plan.md` for anything beyond a single-file, obviously-checkable edit — the threshold is deliberately low. Copy the LOOP card (§3) to its top.

**Step rules:**

- A step touches ≤2 files and ~≤40 changed lines. **Exemption:** a mechanical change produced by one command (sed/IDE rename, formatter run, lockfile regeneration, codegen) is one step regardless of size; its check is TEST.
- Every step has a check command from the §0 fallback chain, written before the step starts. No check command = not a step: split it, or pick a rung lower on the chain.
- Order steps so the repo stays GREEN: add the new path before removing the old; add the test before the feature.
- **EXPECTED-red:** a test written before its feature is *supposed* to fail. Mark it `EXPECTED` in the plan; its check is "fails for the right reason" (the assertion, not an import or syntax error). EXPECTED-red never counts as a failure in §4 or §5.

**Assumptions:** number them; give each a probe command where possible; probes must run in <10s (redesign any that don't). Run all probes now, record output. A probe that can't run yet (needs a server you don't have): mark `UNVERIFIED` and run it at the first step that depends on it.

## 3. Execution — the LOOP card

Copy this verbatim into the top of `plan.md`:

```
LOOP — after every step:
1. Run the step's check. Not GREEN (baseline-relative) -> stop -> triage (§4).
   Never proceed on a non-GREEN check, not even "to finish the next bit first."
2. git diff — re-read every change from disk. Tool success is not verification.
3. Commit: git add -A && git commit -m "checkpoint: step N"
   (no git: copy changed files to .agent/snapshots/N/).
4. One line to worklog.md; update beliefs.md.
5. Re-read plan.md top to bottom.
CAPS: max 3 edits between check runs (1 edit = 1 write/patch operation to 1 file).
Editing a file demotes ALL beliefs.md entries about that file — re-verify before relying.
```

**Additional execution rules:**

- Every `beliefs.md` entry cites the command + output that proved it. No citation = guess. A guess is legal only as a hypothesis your *very next action* tests. Never build on a guess; never ship one.
- **Deny-list — stop and ask first:** `rm -rf`, `git push`, `git reset --hard` on anything except your own checkpoint commits, `DROP`/`DELETE` on real data, schema migrations, deploy commands, any write to an external system.
- **Allow-list — no permission needed:** package installs from standard registries (npm/pip/cargo/etc.), `git fetch`/`pull` of this repo, read-only GETs of documentation, anything confined to the repo and `/tmp`.
- Neither list, and it touches network or shared state → fail closed: ask.
- **Suite economy:** if baseline TEST runs ≤ ~120s, run full TEST at every checkpoint. If longer: step checks at checkpoints; after any §4 fix, also run the tests in the same directories as your edited files; full TEST only at plan events and at done.

## 4. Failure triage

On any non-GREEN check that isn't EXPECTED-red, classify **before** fixing:

1. **Re-run all probes in `assumptions.md`.** Any red → **plan event**: write the now-false assumption as one sentence in `worklog.md`; mark each remaining step unaffected / modify / invalidated; check completed steps for the false premise; re-run the last checkpoint's check; revise `plan.md`; re-read this file; resume. If the *same probe* is red again after one revision cycle → go to §5, it's a stuck condition, not another plan event.
2. Probes green, failure is in code you touched this session, fix is additive → fix, re-run the check, log one line, continue. The **third** fix in the same file or against the same check → treat as a plan event anyway.
3. The fix requires undoing a committed green step → plan event.
4. **Ask the user** when: the fix changes the **user-visible surface** (API response shapes, CLI output, persisted formats, user-facing strings) beyond the request; the request conflicts with behavior existing tests protect; the fix needs a deny-listed command. Pre-existing failures are already baselined — handoff note, not a blocking question, unless the task requires touching them.

## 5. Stuck

Stuck is a counter: the same check fails on two consecutive runs with **no new information in the output** (ignore timestamps, memory addresses, and line-number drift when comparing), or 3 total attempts against one check. Then the ladder — no skipping rungs:

1. **Stop editing.** Read-only actions only: print the actual value, log the actual request, view the file at the failing line.
2. **Boring causes, each verified by a command:** wrong directory? wrong branch? stale build/cache? service not restarted since the edit? wrong venv/node version? running code ≠ edited code? typo in a name or path?
3. Write remaining hypotheses to `worklog.md`; test cheapest first; one change per test; revert each dead one.
4. **Bisect** from your checkpoint commits: `git reset --hard <last green checkpoint>` (safe — your branch, your commits), re-apply step by step; or build a minimal repro in `/tmp`. Bisection needs no insight; that's the point.
5. **Escalate** with this template filled in — never with "I'm stuck": *Goal / Observed (exact error, verbatim) / Verified (what I checked, with commands) / Eliminated (2–3 hypotheses and how each died) / Needed (the specific question or decision).* Escalating with state on disk is a designed outcome. **Never ship a change you cannot explain** — if the ladder ends without understanding, your output is the report.

## 6. Done — all seven, in order

1. **Trace to the request, not the plan.** Re-read the user's original message; map every explicit requirement to a specific place in the diff.
2. **Full TEST now**, regardless of duration. GREEN (baseline-relative). Earlier greens are void.
3. **Stub sweep, exact command:** `git diff <branch-base>...HEAD | grep '^+' | grep -nE 'TODO|FIXME|XXX|placeholder|console\.log|print\(|dbg!'` — only lines *you added* count; pre-existing hits elsewhere in the repo are not yours to fix. Also scan your diff for commented-out code and hardcoded values you meant to parameterize.
4. **Diff audit:** read the entire branch diff; every hunk attributable to the task, or reverted, or explicitly declared in the handoff.
5. **Negative-space — check, don't add:** endpoint → error path, auth, empty input; parser → empty + malformed input; script → nonzero exit on failure; CLI → `--help` and bad-args behavior. For each unhandled item: handle it **only if the repo's sibling examples handle it**; otherwise note it in the handoff. Never silently add behavior siblings don't have (e.g., auth on an intentionally public endpoint).
6. **Oracle rules:** changed behavior with no covering test → write the test first. Mutation-check once, mechanically: commit; introduce a deliberate break; run the check — it must go red (if it doesn't, the check is fake; fix that first); `git reset --hard HEAD`; run the check — green again. The reset, not your memory, guarantees the sabotage is gone.
7. **Handoff:** what changed; every assumption made; plan deviations and why; what was **not** done; baseline pre-existing failures; which claims are *verified by checks* vs. *self-reviewed only*, in those words; offer to squash the checkpoint commits. Improvements beyond the request are proposed here — never implemented unilaterally.

Self-judged artifacts (docs, refactors where no oracle is possible): convert the request into an itemized checklist *before starting*; review item-by-item at the end, reading the artifact from disk, not from memory of writing it.

## 7. When you're not sure

The master rule above all others: **never resolve uncertainty by generating plausible content.** Resolve it by exactly one of:

1. **Run a command or read a file** — if the answer exists in the repo or environment, that's where you get it.
2. **Ask the user** — for desired behavior: batched, multiple-choice, marked default; one question per mid-task blocker.
3. **Neither possible:** take the most reversible option, write the assumption into `assumptions.md` with a probe if one exists, and state it prominently in the handoff.

On anything touching the deny-list or the user-visible surface list, fail closed: uncertainty there always means option 2. A 10-second question costs one round-trip; a confident wrong action can cost the task.
