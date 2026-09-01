---
name: codex-review
description: Adversarial review loop for a feature's spec, plan, or implementation diff — Claude authors and arbitrates, OpenAI Codex critiques read-only, until VERDICT APPROVED or MAX_ROUNDS. Codex sessions persist per feature in .codex-review/sessions.json (same doc resumes the same thread; spec→plan→impl keeps one thread; another feature starts fresh). Use when the user says "/codex-review", "codex review my spec/plan", "fais reviewer par codex", "adversarial review", "stress-test this doc", right after a spec or plan is written, or right after implementing a reviewed plan (bare invocation auto-detects the freshest artifact). NOT for code unrelated to a plan, NOT for trivial changes.
---

# Codex-Review — Adversarial Doc-Review Loop with Persistent Sessions

Two models, one document, a bounded argument. **Claude is the author and arbiter. Codex is a read-only critic**: its built-in shell and file tools cannot touch a single file (external integrations are outside this guarantee — see Prerequisites). The human gate is the final sign-off; the kickoff is announced (doc, session, model) so the user can interrupt a wrong inference, but it does not block.

Works on any design document — a **spec** (requirements, what to build) or a **plan** (implementation, how to build it). The review prompt adapts to the doc type.

**Sessions persist.** Unlike a one-shot loop, the Codex thread for a feature survives across invocations: fix the doc tomorrow, re-invoke, and the reviewer picks up with all its prior critiques in context instead of re-litigating from zero.

## Prerequisites (verify once, fast)

- Codex CLI installed and recent: `codex --version` (need ≥ 0.130).
- Codex authenticated: a prior `codex login` (ChatGPT account is fine). On auth/model error, surface it — do not silently retry.
- Do NOT pin `-m` unless the user asks. Pinning `gpt-5.x-codex` variants fails on ChatGPT-account auth.
- **Echo the model and reasoning effort before Round 1** — best effort, honestly labeled. Check in precedence order: any `-c` override this skill passes, then project `.codex/config.toml` (repo root, trusted projects), then `~/.codex/config.toml`; report the first `model` / `model_reasoning_effort` found and WHICH layer it came from (e.g. `model gpt-5.6-sol (user config)`). Codex has more layers (profiles, system config, built-in defaults) — if none of the checked layers sets a value, say "unresolved (CLI default)", never assert a model you didn't find. On RESUMED threads the thread's LATEST persisted settings normally apply (round 1's, or a later override's), not the current config — say "resumed thread: persisted settings apply (exact values not directly observable)" instead of re-deriving. If the user objects, stop before burning a round.
- **Read-only guards the shell only.** Codex's sandbox does not constrain MCP tools configured in any `config.toml` layer — a side-effecting MCP server can mutate files or remote systems mid-review. At kickoff, check the config layers for `[mcp_servers.*]`; if any exist, name them to the user before Round 1 and let them decide whether to proceed or strip them for the review. This scan is a heuristic, not a proof — integrations can attach through layers it doesn't see (plugins, managed profiles); the hard guarantee covers shell and file access only.
- **Sandbox flag differs between the two commands.** `codex exec` accepts `-s read-only`. `codex exec resume` does NOT — it rejects `-s`. On resume you MUST force read-only via `-c sandbox_mode="read-only"`, because `config.toml` may default to `danger-full-access` — which would let Codex WRITE files mid-loop. This is the single most important safety detail in this skill.

## Tunables (read from skill args, else default/infer)

| Var | Default | Meaning |
|-----|---------|---------|
| `DOC_FILE` | inferred (see below) | The document under review (spec or plan, any markdown path). |
| `DOC_TYPE` | inferred | `spec`, `plan`, or `impl`. Spec vs plan: infer from path/filename/content (numbered implementation steps = plan; requirements/behavior = spec). `impl` = review the CODE DIFF against the plan — inferred when an implementation just finished (detection below), or forced with `type=impl`. If ambiguous, ask one question. |
| `BASE_REF` | inferred (`impl` only) | Git ref the implementation diff is measured from: the commit the implementation started at (merge-base with main, or last commit before the working-tree changes). Echo it — a wrong base reviews the wrong diff. If commits were already made and the base is not confidently reconstructable, ASK the user to confirm it instead of guessing. |
| `FEATURE` | inferred | Feature slug keying the session. Infer from the filename: strip date prefix, extension, and trailing `-spec`/`-plan`/`-lot-x` qualifiers so spec and plan of the same feature share a slug (`2026-08-30-provenance-receipt-lot-a.md` → `provenance-receipt`). Echo the inferred slug — the user can override with `feature=<slug>`. Inferred or given, the slug MUST match `^[a-z0-9][a-z0-9-]*$` (it is interpolated into file paths); anything else is rejected. |
| `reasoning` | `inherit` | Reasoning effort for the reviewer: `low`/`medium`/`high`/`xhigh`/`max` (upper tiers model-dependent). `inherit` = don't pass anything: fresh threads use the config default, resumed threads keep their latest persisted thread settings. Any other value: append `-c model_reasoning_effort="<value>"` to BOTH the fresh and resume commands, echo it at kickoff — and on a RESUME, warn that the override re-enters config resolution and may change the model too, so echo the model alongside. |
| `MAX_ROUNDS` | `5` | Hard cap on review rounds THIS invocation. The loop ALWAYS terminates at this. |
| `STATE_FILE` | `.codex-review/sessions.json` | Per-feature session registry at the repo root — resolve the root ONCE via `git rev-parse --show-toplevel` and run every codex command from there; invoking from a subdirectory must not spawn a second registry. |
| `LOG_FILE` | `.codex-review/<FEATURE>-review-log.md` | Append-only transcript of the whole argument, across invocations. The artifact. |

Echo the resolved values (doc, type, feature, rounds, session status: fresh / resumed / inherited-from-spec) before starting.

### Inferring `DOC_FILE` when no file is passed

Resolve in this order — take the FIRST hit and announce it ("Reviewing `<path>` — <how it was picked>"):

0. **Implementation just finished → `impl` mode.** Detect: in THIS conversation Claude just implemented a plan (executing-plans / subagent-driven-development run, or a stretch of source edits following a plan) AND the working tree or branch carries the resulting diff AND the feature's plan is known (state-file entry, or the plan doc used during the session). All three true → `DOC_TYPE=impl`, `DOC_FILE` = the plan, review target = the diff since `BASE_REF`. Announce: "Implementation of `<plan>` just finished — reviewing the diff against the plan (impl mode)." Ideal moment: BEFORE the commit, so findings land in the same changeset. Only the doc trail is ambiguous → ask; a plain "review my code" with no plan lineage is NOT this skill.
1. **Session history.** The spec or plan Claude wrote or last edited IN THIS conversation (typically the output of superpowers brainstorming/writing-plans moments ago). Most recent one wins. This is the normal case: the user just finished a doc and invokes `/codex-review` bare.
2. **State file.** The most recently `updated` entry in `STATE_FILE` whose `last_verdict` is neither `APPROVED` nor `GOOD_ENOUGH` (both are converged) — an argument in progress is the natural default to continue.
3. **Disk.** The newest spec/plan by mtime among untracked/modified markdown in the usual locations (`docs/superpowers/plans/`, `docs/specs/`, `docs/plans/`, repo-root `*-spec.md`/`*-plan.md`). Check `git status` first — a doc just written shows up there.
4. **Nothing plausible** → ask one question.

The inferred pick is announced, never silent — if the user meant another doc, they correct it before Round 1 burns.

## Session state — `.codex-review/sessions.json`

Keyed by feature slug. One entry per feature:

```json
{
  "provenance-receipt": {
    "doc": "docs/specs/2026-08-30-provenance-receipt-spec.md",
    "doc_type": "spec",
    "thread_id": "th_abc123",
    "rounds_total": 3,
    "last_verdict": "REVISE",
    "updated": "2026-08-30T14:12:05Z"
  }
}
```

### Resolution rules (in order, at kickoff)

1. **Same feature, same doc/stage** → RESUME `thread_id`. The reviewer continues where it left off.
2. **Same feature, stage progression spec→plan or plan→impl** → RESUME `thread_id` (the reviewer keeps its earlier critiques in context — the one that attacked the plan is best placed to grade the diff against it). Do NOT write the entry at kickoff: hold the stage transition in memory and commit the new `doc`/`doc_type` together with the round's own verdict only after the first successful, id-verified round — the prior entry stays intact if the resume fails, and the impl prompt still reads the plan's actual last verdict from it. Tell the user which session carried over. Prefer cold eyes on the code instead? `fresh=1` forces a new thread (logged).
3. **Same feature, any other stage combination** (plan→spec, impl→plan, plan→other-plan…) → ask one question: continue the thread or start fresh? Default recommendation: fresh.
4. **No entry for this feature** → FRESH thread. Create the entry after Round 1 returns the `thread_id`.
5. **Resume fails** → look at WHY before touching state. Only a definitive missing/expired-thread error becomes a fresh start (new `codex exec`, overwrite the entry, note the reset in `LOG_FILE`). Auth errors, model errors, timeouts: STOP with the state file unchanged and surface the error — going fresh on a transient failure destroys a valid session id. Never retry a failed resume blind.
6. **State file unreadable** (malformed JSON, unexpected schema) → STOP, preserve the file untouched, tell the user. NEVER treat a corrupt registry as empty — that overwrites every saved session on the next write.

After EVERY round, update the entry: `rounds_total` (cumulative across invocations), `last_verdict`, `updated` (ISO timestamp with time, not a bare date — same-day sessions must stay orderable). Create `.codex-review/` on first use. Suggest gitignoring `.codex-review/sessions.json` once (thread ids are machine-local); the log files are worth committing.

## Flow

### Step 0 — Kickoff (human gate #1)

Resolve tunables + session state (rules above). Confirm in one line: doc, type, feature, session status. Then proceed — no round-by-round approvals; the human gate is at the end.

### Step 1 — The review prompt (per doc type)

**Spec review** (`DOC_TYPE=spec`):

> You are an adversarial reviewer for a product/technical SPEC. Be skeptical and specific — find what's wrong or missing, don't be agreeable. Read the spec at `<DOC_FILE>` (and any repo files you need for context; you are read-only). Attack: ambiguous or contradictory requirements, undefined terms, missing edge cases and error paths, unstated assumptions, unverifiable acceptance criteria, scope holes, requirements the existing codebase makes impossible or already satisfies differently. For each finding, give a one-line fix. Do NOT modify any files. End your reply with EXACTLY one line: `VERDICT: APPROVED` if the spec is sound enough to plan from, or `VERDICT: REVISE` if it still has material problems.

**Plan review** (`DOC_TYPE=plan`):

> You are an adversarial reviewer for an implementation PLAN. Be skeptical and specific — your job is to find what breaks, not to be agreeable. Read the plan at `<DOC_FILE>` (and any repo files you need; you are read-only). Identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. Do NOT modify any files. End your reply with EXACTLY one line: `VERDICT: APPROVED` if the plan is sound enough to implement, or `VERDICT: REVISE` if it still has material problems.

**Implementation review** (`DOC_TYPE=impl`):

> You are an adversarial reviewer for a freshly implemented feature. The frozen plan is at `<DOC_FILE>`; the implementation is the diff since `<BASE_REF>` — run `git diff <BASE_REF>` (plus `git diff <BASE_REF> --stat` and any file reads you need; you are read-only). Also run `git status --short` and read every untracked file: new files are part of the implementation and the diff does not show them. Review it like a contributor PR against the plan: spec fidelity (every plan step present, no silent redesigns), correctness (bugs, race conditions, error paths, edge cases), scope (nothing touched outside the plan's bounds), tests (the plan's proof exists and covers the change), style consistency with surrounding code. For each finding, give file:line and a one-line fix. Do NOT modify any files. End your reply with EXACTLY one line: `VERDICT: APPROVED` if the implementation faithfully realizes the plan, or `VERDICT: REVISE` if it has material problems.

(If the thread is inherited from the plan review, prepend a verdict-accurate line: *"You previously reviewed the PLAN for this feature (your last verdict: <actual last_verdict>). It has now been implemented."* — never claim approval the thread didn't give. The reviewer then attacks the code with its own plan critiques in mind.)

**Finding discipline — append this paragraph to EVERY review prompt (all three types):**

> Tag every finding `[defect]` (something concretely breaks: state the failure scenario in one line) or `[simplification]` (the doc/code does more than needed: name what to REMOVE or shrink). A finding that proposes ADDING mechanism — a new abstraction, layer, config option, dependency, or speculative flexibility — is only valid with a concrete failure scenario attached; otherwise don't raise it. Prefer the fix with the smallest diff that fully resolves the failure. `VERDICT: REVISE` requires at least one `[defect]` or a `[simplification]` with material payoff — style preferences and hypothetical futures are not material.

Write the prompt to a temp file (`P=$(mktemp)` — the commands below read it as `"$(cat "$P")"`); never inline-quote it.

**Impl-mode round mechanics differ in one place:** on `VERDICT: REVISE`, Claude arbitrates the findings as usual, but fixes the CODE (not the doc) — apply accepted fixes, rerun the affected tests, then resume with *"Findings addressed (or rejected with reasons below). Re-review the diff since `<BASE_REF>`."* (These accepted in-loop fixes are authorized — the "code only after human gate #2" hard rule governs STARTING an implementation, not repairing one under review.) In impl mode `MAX_ROUNDS` defaults to **2** (initial review + one re-inspection — mirrors claudex-loop's `MAX_INSPECTION_ROUNDS`; delegating trivia back and forth burns more than it saves); an explicit `rounds=` argument is honored. At the cap, Claude finishes the remaining accepted fixes directly and logs it.

### Step 2 — First call of this invocation

Every round streams its JSONL events to `STREAM_FILE=.codex-review/<FEATURE>-stream.jsonl` (overwritten each round) — that file is both the `thread_id` source and the live progress monitor (below).

Before EVERY launch: `rm -f /tmp/codex-verdict.txt` — a stale verdict from a previous round must never be readable as this round's result. Success requires ALL of: exit 0, a `turn.completed` event in the stream, and a freshly written non-empty verdict file.

**If FRESH** (creates the session — capture `thread_id`; `$P` is the prompt temp file):

```bash
codex exec -s read-only --json \
  -o /tmp/codex-verdict.txt \
  "$(cat "$P")" \
  < /dev/null 2>/tmp/codex-review-stderr.txt > "$STREAM_FILE"
```

Parse `thread_id` from the `{"type":"thread.started","thread_id":"..."}` line (first line of `STREAM_FILE`) and hold it in memory; create the state entry only once the round passes ALL success gates (exit 0, `turn.completed`, non-empty verdict) — same commit-after-success pattern as stage transitions, so a failed round never leaves a resumable-looking entry. The critique text lands in `/tmp/codex-verdict.txt`.

**If RESUMED / INHERITED** (entry exists — echo the id visibly into the command before running; `$P` is the resume-prompt temp file):

```bash
# resume rejects -s. Force read-only via -c sandbox_mode, or Codex inherits
# config.toml (possibly danger-full-access) and could WRITE files.
codex exec resume "$THREAD_ID" -c sandbox_mode="read-only" --json \
  -o /tmp/codex-verdict.txt \
  "$(cat "$P")" \
  < /dev/null 2>/tmp/codex-review-stderr.txt > "$STREAM_FILE"
```

Resume identity check: the stream emits `thread.started` on resumes too — compare its `thread_id` to `$THREAD_ID`. A mismatch means Codex silently fell back to a NEW thread (observed upstream): stop, don't record the round, state unchanged.

### Progress monitoring (long rounds)

A real review round runs minutes; don't sit blind. Launch the round with `run_in_background: true` and poll `STREAM_FILE` every ~30-60s with a quick tail. Event vocabulary (verified 2026-08-30, codex-cli 0.151.0):

- `{"type":"thread.started","thread_id":...}` — session up (emitted on fresh AND resumed calls; on resume, verify the id matches).
- `{"type":"turn.started"}` — Codex is working.
- `{"type":"item.completed","item":{"type":...}}` — one unit of work done. `item.type` values worth surfacing: `command_execution` (the `command` field shows what Codex is reading/grepping — the best progress signal), `reasoning` (thinking summary), `agent_message` (the final critique text), `error` (surface it verbatim).
- `{"type":"turn.completed","usage":{...}}` — round over; token usage attached. Completion = this line present + the `-o` verdict file written + background command exited.
- `{"type":"turn.failed"}` or a top-level `{"type":"error"}` — immediate failure: surface the event payload (then the stderr tail), don't misread it as a dead session.

Per poll, report one short line to the user from the last few events, e.g. "Codex: 7 commands run, currently `rg -n auth docs/specs/...`". Don't parse the stream for the critique content — that's what the `-o` file is for. If the stream stalls with no new event for ~5 min and the process is still alive, say so and keep waiting until the timeout ceiling; if the process died with no `turn.completed`, treat as a failed run (state rule 5 or stop).

Resume prompt, adapted to the case:
- Same doc, revised since last invocation: *"The doc at `<DOC_FILE>` has been revised since your last review. Re-review it — check whether your prior findings are addressed and flag anything new. Same rules. End with VERDICT: APPROVED or VERDICT: REVISE."*
- Spec→plan inheritance: *"You previously reviewed the SPEC for this feature. The feature now has an implementation PLAN at `<DOC_FILE>`. Review the PLAN with your spec knowledge: does it faithfully implement the spec, and does it hold up on its own? <plan attack list from Step 1>. End with VERDICT: APPROVED or VERDICT: REVISE."*

> **`< /dev/null` is mandatory on both commands:** `codex exec` reads stdin *in addition to* the prompt arg, so under a non-interactive driver (Claude Code's Bash tool, CI) it blocks forever waiting on stdin EOF — a silent ~0% CPU hang. stderr goes to `/tmp/codex-review-stderr.txt`, not `/dev/null`: on success ignore it (cosmetic MCP/auth noise), on failure READ it and surface the tail to the user — auth and model errors land there. Confirm success by the verdict file (and `thread.started` line on fresh calls). Neither → failed run (auth/model/dead thread) — apply state rule 5 or stop and tell the user, quoting the stderr tail.
>
> **Timeout guard:** every `codex exec` / `codex exec resume` gets a 10-minute ceiling. Foreground via Claude Code's Bash tool: pass `timeout: 600000` (the default 2-minute tool timeout kills real reviews). Background runs (the monitored path below): prefix the command with `timeout 600` (Linux) / `gtimeout 600` (macOS coreutils) since the tool timeout doesn't bound background commands. No `gtimeout` on the machine? Don't install anything — enforce the ceiling through the polling loop instead: past 10 min of wall clock, kill the process and treat it as a failed run. Ceiling trips = failed run: stop and tell the user, don't retry blind.

### Step 3 — The loop (rounds 2..MAX of this invocation)

All subsequent rounds resume the SAME `THREAD_ID` with the resume prompt matching `DOC_TYPE`: doc revision ("The doc has been revised…"), stage inheritance, or impl re-review ("Re-review the diff since `<BASE_REF>`") — see Steps 1-2; never send the doc-revision prompt on an impl round. Each round, after Codex returns:

1. Read `/tmp/codex-verdict.txt`. Append to `LOG_FILE`: `## Round <rounds_total> — Codex (<date>, <doc_type>)` + the full critique VERBATIM — never a summary; a condensed log defeats the artifact.
2. Grep the last line for the verdict token.
   - `VERDICT: APPROVED` → update `STATE_FILE`, break to Step 4 (converged).
   - `VERDICT: REVISE` → Claude decides **what's actually worth acting on** (Claude is final arbiter — Codex advises, it does not command). Revise `DOC_FILE`. Append to `LOG_FILE`: `### Claude's response` + what changed, what was rejected, why. Update `STATE_FILE`. Increment round.
3. Rounds this invocation > `MAX_ROUNDS` → break to Step 4 (paused, not deadlocked — the session persists; the user can fix the doc and re-invoke to continue the argument).

### Arbitration doctrine — the simplicity ratchet

Applied by Claude on EVERY `REVISE`, before touching the doc or the code. Two standing questions, asked of each accepted fix — never of the finding's existence:

1. **Minimal modification** — is this the smallest change that fully resolves the stated failure? If a smaller fix resolves it, take the smaller fix (log: "accepted, reduced").
2. **Simplest system** — does the fix leave the system with fewer or equal moving parts? A fix that adds mechanism (abstraction, layer, config, dependency) is accepted ONLY if the finding is a `[defect]` with a concrete failure scenario AND no removal-or-adjustment fix exists. Otherwise reject as over-engineering, with the reason logged.

Bounds on the ratchet itself — it must not be over-interpreted either:
- It never leaves a VERIFIED concrete `[defect]` unresolved — but verification comes first: check the premise against the actual doc/code, and a finding whose premise is disproven is rejected with the disproof logged (a concretely-phrased claim can still be wrong). For verified defects, simplicity governs the SHAPE of the fix, not whether it gets fixed.
- It is not a license to strip things the spec/plan requires. "Simplest" = simplest that meets the locked requirements, not simpler than them.
- One pass per finding. Don't re-litigate an accepted-and-applied fix in a later round unless Codex raises a new failure against it.

Anti-loop rules (the convergence side of the same coin):
- **Rejections are sticky.** A finding rejected with a logged reason stays rejected in later rounds unless Codex brings NEW evidence (a failure scenario it didn't state before). Repeat findings without new evidence: reply in the resume prompt "already adjudicated, see log" and don't re-argue.
- **Late nitpicks don't loop.** From round 3 on, new `[simplification]` findings with no material payoff are batched into the log as "noted, not blocking" — they never cause another round on their own.
- **Good-enough exit.** If a round's findings contain zero `[defect]`s and only immaterial simplifications, Claude may declare convergence and go to Resolution even without `VERDICT: APPROVED` — presenting the residual list to the user, and recording `last_verdict: "GOOD_ENOUGH"` in `STATE_FILE` so the state matches the declared outcome. Iterating to zero nitpicks is the infinite loop this rule exists to kill.

Each resume prompt after round 2 appends: *"Do not raise new minor points — only material defects or simplifications with clear payoff justify REVISE. Previously adjudicated findings are settled unless you have new evidence."*

### Step 4 — Resolution (human gate #2)

- **APPROVED:** present the final artifact, a 3-bullet summary of what the argument improved, and the round count (this invocation + cumulative). For a spec: *"Spec survived N rounds. Write the plan now (the review session carries over to it), or stop here?"* For a plan: *"Plan survived N rounds. Implement, or stop here?"* Code only on a yes. For an impl: present the reviewed diff + findings dispositions and ask about committing — the human gate before any commit, and Claude writes the commit only on an explicit yes.
- **GOOD_ENOUGH declared (ratchet exit):** same presentation as APPROVED, plus the residual "noted, not blocking" list verbatim — the user decides whether any residual matters before authorizing the next stage.
- **MAX_ROUNDS hit without APPROVED:** do NOT fake convergence. List each point Codex still flags + Claude's counter-position. The session is saved — remind the user that re-invoking on the same doc continues the same argument.

## Hard rules

- Codex's shell/file access is read-only EVERY round — `-s read-only` on fresh calls, `-c sandbox_mode="read-only"` on every resume. It never writes through those tools; external integrations sit outside this guarantee (heuristic MCP scan at kickoff).
- The loop ALWAYS terminates at `MAX_ROUNDS` per invocation. Persistence is for continuing across invocations, not for unbounded loops within one.
- One entry per feature in `STATE_FILE` — a new doc for the same feature overwrites the entry (rules 2-3), it never creates a duplicate.
- Never resume with `--last` — always the explicit `thread_id` from `STATE_FILE`, echoed visibly before running (a missing/garbage id can silently fall back to the most recent session instead of erroring — a wrong-target resume looks exactly like a successful one).
- Claude is final arbiter on every REVISE — incorporate good critiques, reject bad ones *with a reason logged*, and pass every accepted fix through the simplicity ratchet. Don't cave to everything, don't ignore it.
- STARTING an implementation happens only after human gate #2, and only from an APPROVED plan or a GOOD_ENOUGH plan the user explicitly signed off with its residual list (an approved spec is not an implementation license). Accepted fixes during an impl-mode review loop are not "starting" — they are part of the review.
- `LOG_FILE` is the deliverable — the whole argument, across all invocations, one file per feature.

## What NOT to do

- Don't use this to review existing code.
- Don't pin a `-codex` model variant on ChatGPT-account auth — it 400s.
- Don't parse the JSONL stream for the critique — read the `-o` file.
- Don't let Codex edit files. Shell/file sandbox read-only, always.
- Don't start a fresh thread when a live entry exists for the doc/feature — that throws away the reviewer's context, which is the point of this skill.
