# claude-skills

Personal Claude Code skills. One so far: **codex-review**. Version française : [README-fr.md](README-fr.md).

## codex-review — the adversarial review loop

Two models, one document, a bounded argument. Claude writes (spec, plan, then code) and referees; OpenAI Codex critiques read-only, round after round, until `VERDICT: APPROVED` or the round cap. The conversation with the critic survives across invocations: fix the document tomorrow, run again, and the reviewer picks up with all its earlier critiques in mind instead of rediscovering everything.

The skill runs after a superpowers brainstorming/plan (`/codex-review` with no argument finds the document by itself) and covers three stages of the same feature: the **spec**, the **plan**, then the **implementation** (the diff reviewed against the plan, ideally before commit).

Four diagrams tell the story, from the overview down to the details.

### Level 1 — the architecture

![Architecture of the skill](codex-review/architecture.png)

Everything is there, left to right:

- **The column of three stages.** Spec, plan, implementation (the diff). One feature walks down these three steps, and the annotation says it: *one feature, one session*. The same Codex thread follows all three stages, so the reviewer that attacked the spec and then the plan ends up judging the code with full context.
- **The central loop.** Claude (author and referee) sends *doc + prompt*; Codex (a terminal, read-only critic) returns a *VERDICT*: `APPROVED` or `REVISE`. The *MAX ROUNDS* counter bounds the argument. It always ends.
- **The simplicity ratchet** on the way back: every accepted fix passes this filter before touching the document (detailed in plate 4).
- **The persistent state**, on the right: `sessions.json` (each feature's Codex thread, linked by its *thread_id* and *written only after a successful round*, never at launch), `review-log.md` (the whole debate, verbatim) and `stream.jsonl` (live progress while a round runs).
- **The human sign-off**, at the bottom: nothing gets implemented without the user's final yes.

### Level 2a — the kickoff: which doc, which session

![Kickoff — choosing the doc and the session](codex-review/architecture-kickoff.png)

What happens before the first round.

On the left, **the four-tray funnel**: invoked with no argument, the skill guesses the document by trying four sources in order: ① the document just written in the conversation, ② an argument in progress in `sessions.json`, ③ the disk (`git status`, the most recent doc), ④ otherwise a single question. The result (*doc + type + feature*) is always announced, never silent: the user can correct it before a round is spent.

On the right, **three tracks into the spool** (the Codex thread): same doc → resume the thread; spec → plan → impl of one feature → still the same thread; another feature → a new thread. A broken thread is handled with care: only a thread that is *truly dead* justifies starting over (logged), while a *passing error* (auth, model, timeout) stops the run and leaves the state untouched.

At the bottom, **the coupling triangle**: the document, `sessions.json` and the *thread_id* hold together. One entry per feature, no duplicates.

### Level 2b — one round, up close

![One round, up close](codex-review/architecture-round.png)

The anatomy of one round trip.

1. **The brief.** Claude writes the review prompt, which imposes the critique discipline: every finding is tagged *it breaks* (with the failure scenario) or *too complicated*. Every critique must prove its cost. Before the brief leaves, a stamp asks *doc really changed? hash check*: the document's hash is compared with the previous round's, and an unchanged document is never resubmitted (two real rounds were once burnt that way after a failed edit script).
2. **The read-only pen.** Codex works locked in: its shell and file access can write nothing, temp directories included. That is also why it cannot run the test suite itself. In implementation mode the *test results come from Claude*, pasted into the prompt.
3. **The ticker.** Activity streams continuously to `stream.jsonl` (started, reading files, done), watched every 30-60 s. The guard is an inactivity guard, not a wall clock: *10 minutes of silence* on the stream stops the round, with a hard cap of 45 minutes. Real first rounds on a large document take 14 to 18 minutes.
4. **The verdict.** `APPROVED (accepted)` or `REVISE (fix and resubmit)`. On REVISE, Claude fixes and goes again, same conversation. The whole debate accumulates in `review-log.md`, one file per feature.

### Level 2c — the simplicity ratchet

![The simplicity ratchet](codex-review/architecture-cliquet.png)

The guard against over-engineering, applied by Claude on every REVISE.

Critiques arrive on the conveyor. First stop, the magnifying glass: *is the claim true? check the code first*. A concretely worded critique can still be wrong, and a disproven premise is rejected with the disproof logged. Each candidate fix then passes **two doors**: *smallest change?* (is this the smallest modification that resolves the failure?) and *simplest system?* (does the fix leave the machine with as many or fewer moving parts?). The scale below sets the rule: *adding a gear needs proof*. No speculative mechanism.

The spyglass on the horizon fixes the yardstick: *judge it in tomorrow's production, not today's numbers*. During development the affected population is empty by construction, so "zero users hit this today" is never a reason to reject or shrink a fix. The review protects the state that will ship.

The hanging sign bounds the filter itself: *a real break is always fixed*. The ratchet governs the shape of the fix, never whether a verified bug gets fixed. The ratchet wheel turns one way only: *a validated fix is not re-judged*. And the three exits kill the infinite loop: *rejected once = rejected* (unless new proof), *hair-splitting: noted, not blocking*, and *nothing breaks → we stop*. What is left is shown rather than polished forever. Every rejection is written down.

## Regenerating the plates

The plates are drawn by `gpt-image-2` (OpenRouter, `POST /api/v1/images`, aspect ratio 3:2, 2K). Each `codex-review/architecture*-body.txt` holds the exact scenario; append `codex-review/STYLE.txt` to it to get the full prompt. Generate the architecture plate first, then pass it as `input_references` for the three others so the same hand draws all four. Check the lettering on every plate: about one plate in four drops or misspells a label and needs a second run. Cost is around $0.05 to $0.18 per plate.

## Installation

```bash
git clone git@github.com:darksip/claude-skills.git
ln -s "$PWD/claude-skills/codex-review" ~/.claude/skills/codex-review
```

Usage: `/codex-review` (automatic inference), or with arguments: `feature=<slug>`, `type=spec|plan|impl`, `rounds=<n>`, `reasoning=low|medium|high|xhigh|max`, `fresh=1`. Prerequisite: `codex` CLI ≥ 0.130, authenticated (`codex login`). The full contract is in [codex-review/SKILL.md](codex-review/SKILL.md).
