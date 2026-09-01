# Codex review — codex-review skill (self-review)

## Round 1 — Codex (2026-09-01T13:01:58Z, skill-def) — thread 01a05d06-2143-76e3-ae93-106c374762bc

1. [defect] [Read-only enforcement](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:20) only constrains Codex’s shell; a configured MCP tool can still mutate files or remote systems because MCP tools are not sandboxed by Codex. [OpenAI confirms this boundary](https://openai.com/index/unrolling-the-codex-agent-loop/).
   Fix: Run reviews with side-effecting MCP/app/plugin tools disabled or independently guaranteed read-only.

2. [defect] [Kickoff](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:79) calls itself a human gate but immediately proceeds, so an incorrectly inferred document, feature, base, or model burns Round 1 before the user can object.
   Fix: Pause for explicit confirmation at Step 0, or stop calling it a gate and remove the correction-before-Round-1 promises.

3. [defect] [Prompt execution](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:101) never defines `REVIEW_PROMPT` or `RESUME_PROMPT`, and the samples use `cat REVIEW_PROMPT` without `$`, so they literally read a nonexistent file and may send an empty prompt.
   Fix: Define concrete temp-path variables and use `"$(cat "$REVIEW_PROMPT")"` and `"$(cat "$RESUME_PROMPT")"`.

4. [defect] [Fixed `/tmp` outputs](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:113) allow a failed round to leave the previous `/tmp/codex-verdict.txt`, which the success check can mistake for the new verdict.
   Fix: Use unique files per invocation/round and accept success only with exit code 0, `turn.completed`, and a newly written nonempty verdict.

5. [defect] [Stream failure handling](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:137) recognizes only item-level errors, so top-level `turn.failed` or `error` events can be missed and later misclassified as a dead session; both are current JSONL event types. [Codex event source](https://github.com/openai/codex/blob/main/codex-rs/exec/src/exec_events.rs).
   Fix: Treat top-level `turn.failed` and `error` as immediate failures and surface their payloads before consulting stderr.

6. [defect] [Resume recovery](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:71) says every resume failure becomes a fresh thread, contradicting the instruction not to retry auth/model errors; a transient auth, model, or timeout failure can overwrite a valid session ID.
   Fix: Fall back fresh only for a definitive missing/expired-thread error; otherwise stop with state unchanged.

7. [defect] [Resume identity](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:135) incorrectly says `thread.started` is fresh-only and never compares the emitted ID on resume, so a silent new-thread fallback can be recorded as a successful resume. [A documented Codex case emits a different ID](https://github.com/openai/codex/issues/15538).
   Fix: Parse `thread.started` on every invocation and require its ID to equal `THREAD_ID` for resumes.

8. [defect] [Model precedence](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:19) omits nested project configs, profiles, system config, and built-in defaults, so its claimed “active model” can be wrong. [Official precedence has six layers](https://developers.openai.com/codex/config-basic/).
   Fix: Resolve the complete official precedence or report unresolved values as unknown instead of claiming an active model.

9. [defect] [Resume model echo](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:30) says `inherit` uses the current config default, but resume normally restores the thread’s persisted model and effort, so the skill can echo B/medium while actually running A/xhigh. [Codex resume semantics](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md).
   Fix: On resume, report persisted thread settings; do not derive them solely from current config files.

10. [defect] [Reasoning override](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:30) can silently change the model because supplying `config.model_reasoning_effort` disables persisted resume fallback and re-enters normal config resolution.
    Fix: When overriding effort on resume, preserve or explicitly confirm the effective model as a pair with the effort.

11. [defect] [Stage overwrite](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:68) overwrites `doc`/`doc_type` at kickoff while retaining the prior verdict; a crash before Round 1 can leave an `impl` entry falsely carrying the plan’s `APPROVED`.
    Fix: Commit the stage transition only after a successful round returns its new verdict.

12. [defect] [Corrupt state](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:48) gives no rule for malformed JSON, so an agent can interpret a truncated registry as empty and overwrite every saved session.
    Fix: On parse or schema failure, stop and preserve the file; never treat invalid state as an empty registry.

13. [defect] [Feature path injection](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:29) accepts an unrestricted `feature=<slug>` even though it is interpolated into file paths, so `feature=../../notes` escapes `.codex-review/`.
    Fix: Require `FEATURE` to match a safe slug pattern such as `^[a-z0-9][a-z0-9-]*$`.

14. [defect] [Repository targeting](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:32) says paths are repo-root-relative but never resolves or pins the repo root, so invocation from a subdirectory can create a second registry and review the wrong working tree.
    Fix: Resolve `git rev-parse --show-toplevel` once and use absolute paths plus an explicit Codex working directory.

15. [defect] [Impl base inference](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:28) cannot reliably reconstruct “the commit implementation started at” after commits exist; merge-base may include unrelated branch work while the last commit may exclude committed implementation work.
    Fix: Capture/store the base before implementation or require an explicit validated commit when it is unavailable.

16. [defect] [Impl diff coverage](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:93) uses `git diff <BASE_REF>`, which omits untracked files, so the preferred pre-commit review can approve an implementation without reviewing newly created source or tests.
    Fix: Require `git status --short` and explicit inspection of every untracked implementation file.

17. [defect] [False plan approval](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:95) always tells an inherited reviewer the plan was approved, although plan→impl inheritance does not require `last_verdict=APPROVED`.
    Fix: Use that sentence only for an approved plan; otherwise state the actual verdict and block or explicitly qualify impl review.

18. [defect] [Impl writes versus hard rule](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:103) directs Claude to fix code during the loop, while the hard rule forbids coding before human gate #2; agents can plausibly either skip required fixes or violate the hard rule.
    Fix: Scope the hard rule to starting implementation and explicitly authorize accepted review fixes within an already approved implementation run.

19. [defect] [Impl resume prompt](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:152) says every later round uses “I revised the doc,” contradicting impl mode’s code-specific re-review prompt and risking review of the frozen plan instead of the changed diff.
    Fix: Branch Step 3 by `DOC_TYPE` and include impl dispositions plus “re-review the diff” verbatim.

20. [defect] [Impl round limit](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:103) says “cap at 2 by default,” leaving it ambiguous whether explicit `MAX_ROUNDS=5` is honored or forcibly reduced.
    Fix: State either “default 2, explicit override allowed” or “hard maximum 2.”

21. [defect] [Arbitration contradiction](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:168) forbids rejecting any concretely phrased defect while the hard rules require rejecting bad critiques, so a concrete finding based on a false premise forces unnecessary changes.
    Fix: Replace this with “never leave a verified concrete defect unresolved” and require premise verification.

22. [simplification] REMOVE the [good-enough exit](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:175): it creates a third terminal state absent from `last_verdict` and Step 4, leaving state as `REVISE` after Claude declares convergence.
    Fix: Use the existing bounded `MAX_ROUNDS` non-approved resolution for residual immaterial findings.

VERDICT: REVISE
### Claude's response (round 1)

Accepted (some reduced to smaller fixes): 1 (MCP warning at kickoff, no auto-strip), 2 (kickoff reworded: announce, not gate), 3 ($P mktemp + "$(cat "$P")"), 4 (rm -f verdict before launch + triple success check), 5 (turn.failed/error added to vocabulary), 6 (fresh only on definitive thread-not-found; transient errors stop with state unchanged), 7 (thread.started parsed on resume, id compared), 8 (model echo now best-effort, admits unresolved), 9-10 (resume persisted-settings note + override pairing warning), 11 (stage overwrite sets last_verdict PENDING), 12 (corrupt state = stop, never empty), 13 (slug regex), 14 (git rev-parse --show-toplevel, run from root), 15 (ambiguous BASE_REF asks), 16 (untracked files added to impl prompt), 17 (verdict-accurate inheritance line), 18 (hard rule scoped to STARTING implementation), 19 (Step 3 branches resume prompt by DOC_TYPE), 20 (impl cap: default 2, rounds= honored), 21 (ratchet bound: verified defects, false premises rejected with disproof).

Rejected: 22 (remove good-enough exit) — the exit is a user-mandated anti-infinite-loop mechanism; the real defect (state left as REVISE) is fixed by recording last_verdict GOOD_ENOUGH instead of removing the rule.

Found by Claude, missed by review: inference rule 1 (session history) had been clobbered by an earlier edit (list jumped 0→2); restored.

## Round 2 — Codex (2026-09-01T13:06:06Z, skill-def)

1. [defect] [Stage progression](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:70) still mutates state before resume, contradicting the “state unchanged” guarantees on auth failure or ID mismatch; it also replaces the prior plan verdict with `PENDING` before line 98 needs that verdict.
   Fix: Keep the old entry in memory and write the new stage plus returned verdict only after a successful matching round.

2. [defect] [Untracked-file coverage](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:98) is conditional on plan-thread inheritance, so a fresh impl review still omits newly created files and can approve an incomplete diff.
   Fix: Move the `git status --short` and untracked-file instruction into the unconditional impl prompt.

3. [defect] [`GOOD_ENOUGH`](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:183) is recorded but not consumed consistently: inference selects every status except `APPROVED`, restarting the supposedly converged review, while Resolution has no `GOOD_ENOUGH` branch.
   Fix: Exclude both terminal statuses during unfinished-session inference and add an explicit `GOOD_ENOUGH` Resolution branch defining next-step authorization.

4. [defect] [The read-only check](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:20) scans only `[mcp_servers.*]`, but installed plugins can connect Codex to tools such as GitHub, Gmail, and Calendar without appearing there; the skill can therefore announce safety while exposing mutating tools. [Official Codex CLI documentation](https://learn.chatgpt.com/docs/codex/cli).
   Fix: Inspect the actual active plugin/MCP tool set or launch an isolated reviewer profile with all side-effecting integrations disabled.

5. [defect] [Persisted model wording](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:19) says “round-1 settings persist,” but the skill permits later resume overrides that may change the persisted model/effort; a subsequent `inherit` can therefore be mislabeled.
   Fix: Say “latest persisted thread settings” and label their exact values unresolved unless directly observed.

VERDICT: REVISE
### Claude's response (round 2)

Accepted: 1 (stage transition held in memory, committed only after a successful id-verified round — prior entry intact on failure), 2 (untracked-files instruction moved into the unconditional impl prompt), 3 (inference excludes GOOD_ENOUGH; Resolution gains an explicit GOOD_ENOUGH branch).

Accepted reduced: 4 (MCP scan labeled a heuristic covering shell/file access only — inspecting the full active integration set or isolated profiles rejected as added mechanism without verified tooling), 5 (wording: latest persisted settings, exact values not directly observable).

## Round 3 — Codex (2026-09-01T13:08:14Z, skill-def)

1. [defect] [Read-only scope](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:20) admits integrations may mutate files or remote systems, but lines 8, 195, and 208 still claim Codex “cannot touch a single file” and “never writes”; proceeding after the heuristic warning makes those hard rules false. Plugins and MCP connect Codex to external tools and data per [official OpenAI documentation](https://learn.chatgpt.com/docs/codex/cli).
   Fix: Scope every absolute read-only claim to Codex’s built-in shell/file tools and explicitly exclude integrations.

2. [defect] [Reasoning tunable](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:31) still says resumed threads keep “round-1 settings,” contradicting line 19’s corrected “latest persisted settings”; after a later override, `inherit` can be described incorrectly.
   Fix: Replace “persisted round-1 settings” with “latest persisted thread settings.”

3. [defect] [GOOD_ENOUGH authorization](/Users/pierresiccardi/Sources/perso/claude-skills/codex-review/SKILL.md:190) lets the user authorize the next stage using the APPROVED flow, while line 200 permits starting implementation only from an `APPROVED` plan; a `GOOD_ENOUGH` plan therefore simultaneously permits and forbids implementation.
   Fix: State whether explicit human sign-off promotes `GOOD_ENOUGH` to implementation authorization, or restrict that branch to stopping/re-reviewing.

VERDICT: REVISE
### Claude's response (round 3)

All 3 accepted: 1 (every absolute read-only claim scoped to Codex's built-in shell/file tools, integrations excluded explicitly), 2 (reasoning tunable now says latest persisted thread settings), 3 (hard rule: implementation may start from APPROVED, or from GOOD_ENOUGH explicitly signed off with its residual list).

## Round 4 — Codex (2026-09-01T13:09:37Z, skill-def)

Two material contradictions remain:

- [defect] Line 8 still says Codex “cannot touch a single file”; an agent could treat this as a global guarantee and leave a mutating integration enabled despite the later exclusions.
  Fix: Scope this sentence to Codex’s built-in shell/file tools and explicitly exclude integrations.

- [defect] Line 31 still says resumed threads retain “persisted round-1 settings,” contradicting line 19; after a later reasoning override, `inherit` may be interpreted or reported incorrectly.
  Fix: Replace “persisted round-1 settings” with “latest persisted thread settings.”

VERDICT: REVISE
### Claude's response (round 4)

Both findings REJECTED — false premises, disproof by direct file read:
- Line 8 actually reads: "its built-in shell and file tools cannot touch a single file (external integrations are outside this guarantee — see Prerequisites)" — the exact scoping requested in round 3.
- Line 31 actually reads: "resumed threads keep their latest persisted thread settings" — the exact wording requested in round 3.

## Round 5 — Codex (2026-09-01T13:11:59Z, skill-def)

Fresh read confirms lines 8 and 31 contain the quoted fixes. My two round-4 findings are withdrawn.

- [defect] Line 123 writes a fresh `thread_id` immediately upon `thread.started`, before line 112’s success gates; a later timeout or `turn.failed` leaves state pointing to a failed, incomplete round that the next invocation treats as resumable.
  Fix: Hold the fresh ID in memory and create the complete state entry only after exit 0, `turn.completed`, and a non-empty verdict.

VERDICT: REVISE
### Claude's response (round 5)

Round-4 findings formally withdrawn by the reviewer after a fresh read. The one new defect ACCEPTED: fresh thread_id is now held in memory and the state entry is created only after all success gates pass (same commit-after-success pattern as stage transitions).

## Resolution

MAX_ROUNDS=5 reached without an APPROVED verdict. Honest status: every finding across 5 rounds is resolved (26 accepted incl. reduced forms), withdrawn (2), or rejected with logged disproof (3: good-enough removal, two false-premise round-4 claims). Zero unresolved defects remain; the last accepted fix was applied after the final round, so no verdict covers it. Session persists — a future invocation can continue thread 01a05d06 for a clean APPROVED if desired.
