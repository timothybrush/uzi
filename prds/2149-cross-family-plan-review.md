# PRD #2149: Cross-family plan review for auto-approved runs (Claude lead, Codex reviewer)

**Status**: Draft. Child 1 of 3 under umbrella #2148. Followed by PRD #2150 (automatic revise rounds and the Codex-lead direction) and PRD #2151 (reviewer model and effort pins).

Resolved facts below were read at `main` `8a5f138e`.

## Problem

On an auto-approved run nobody reviews the plan. The autopilot branch of `RunRunner.gatePlan` (`agent/src/runner.ts`) reports `running` with the plan and approves at once. The only automated checks are the history-rewrite nudge (`planProposesRewrite`) and the forced human gate for a CI-config `ci_fix` plan (`isCIConfigPlan`). Auto-approve is the default for every catalog schedule and sweep, for `self_improve`, CI autofix and `mr_rework`, so most unattended work implements whatever the lead planned.

When a maintainer steers plans by hand with a second session on the other model family reviewing each one, the reviewer asks for changes on most plans. In five such pairings (2026-09-30 to 2026-10-02, 11 plan gates) the cross-family reviewer caught gate commands with inlined environment variables, regression tests that could not fail on the unfixed code, unrequested behaviour changes, an overclaimed security property, stale ADR text, a missing changelog entry, and non-deterministic acceptance procedures. Unattended runs get none of this, although they are the runs with no human at the gate.

## Outcome

A user with both model families usable can opt in to plan review. Every new auto-approved run of theirs that reaches the plan gate on the Claude harness has its plan reviewed by a separate, read-only run on Codex before it may implement. Only an APPROVE of the exact current plan lets the run implement. A REVISE, a BLOCK, or any failure parks the run at the plan gate for a human, with the reason and the reviewer's findings visible; the human then approves, revises or rejects as on any manual gate. Automatic revise rounds and Codex-lead runs come in PRD #2150; until then an opted-in Codex-lead run parks for a human instead of implementing unreviewed. With the setting off, behaviour is identical to today.

Acceptance examples:

1. Plan review on; Claude and Codex both usable; a worker with a free slot that runs Codex. The nightly sweep starts issue run R on the Claude harness. R's lead submits its plan; a `plan_review` run on Codex reads the repository read-only and returns APPROVE. The server stores the plan and R implements. R's activity feed shows the review.
2. Same setup; the reviewer returns REVISE with two items (or BLOCK, or no reviewer slot frees up before the deadline). R parks at `awaiting_approval` with reason `plan review: changes requested` (or `blocked`, `timed out`); the findings show on the run page and in `uzi run get`; the Slack gate card shows only the reason line. The owner revises, approves or rejects as on a manual gate.
3. The same user's sweep starts a run on the Codex harness. It parks with reason `plan review: not yet supported for a Codex lead`. It never implements an unreviewed plan.
4. Plan review off. No `plan_review` run is created and every write path is today's.

## Out of scope

- Automatic revise rounds and the Codex-lead direction: PRD #2150.
- Reviewer model and effort pins and the epic #1703 Settings → Models row: PRD #2151. Here the reviewer uses what its claim resolves today (below).
- A wait that releases the lead's worker slot. The lead holds its slot, so a reviewed run needs a second free slot or an ephemeral worker; otherwise it parks (Decision D4, user decision 2026-10-03).
- Human-gated runs, seeded plans (`plan_source='seeded'`), gateless kinds (`task`, `chat`, `judge`, `job`), and isolated-lane runs (the isolated lane refuses Codex, `errIsolatedClaimRefused`; such a run parks with `plan review: reviewer unavailable`).
- An admin instance-wide switch. Only the user's own credentials are spent, as with `self_improve`.
- Co-planning by two leads.

## Modules and seams

### Opt-in setting

- `users.plan_review_enabled BOOLEAN NOT NULL DEFAULT false`, on the user DTO.
- `PUT /api/me/plan-review` in the cookie-only `RequireAuth` group in `api/internal/handler/routes_me.go`, beside `/me/judge` and `/me/autopilot`: a consent switch that spends credentials must not be flippable with a CLI token.
- Enabling is refused unless both families are usable for the user (`harnessAvailability` in `api/internal/workersvc/harness_resolver.go`). The response warns, without refusing, when no online worker advertises `plan_review_v1` or can run Codex and ephemeral workers are off for the user. Disabling is always allowed.
- Web: a toggle on Settings → Run defaults (`web/src/pages/RunDefaults.tsx`), mock-mode fixture. CLI: no write verb (consent toggles stay cookie-only, `docs/cli.md`); the account view shows the value.

### Run snapshot

- `runs.plan_review_required BOOLEAN NOT NULL DEFAULT false`, computed inside each INSERT from `users.plan_review_enabled` and never updated: true when the run is inserted with `auto_approve = true`, its kind is in a new `runkind.PlanReviewable` set, and the plan is not seeded.
- `PlanReviewable` = `issue`, `prompt`, `self_improve`, `ci_fix`, `mr_rework`: the kinds whose executors reach the plan gate on an auto-approved run. `runkind.PlanningCapable` is the wrong set; it admits `task`, which is auto-approved but gateless (`docs/handoff.md`). The implementation confirms each member against the executors, and a test pins the set.
- A parity test enumerates every SQL INSERT in `api/internal/store/queries/` that can write `auto_approve = true` on a `PlanReviewable` kind and asserts it computes the flag, in the style of `runkind_sql_test.go`. (The Go callers `CreateAutopilotRun` in `poller/autopilot.go` and `CreateScheduledAutopilotRun` in `schedsvc/scheduler.go` reach those INSERTs.)
- Toggling the setting never changes an existing run; a resume or requeue cannot clear the flag.

### Claim gates

- A run with `plan_review_required = true`, and every `plan_review` run, is claimable only by a worker advertising the new protocol capability `plan_review_v1` (`api/internal/capability`). An older worker would route an unknown kind to `RunRunner.execute` (`agent/src/worker.ts`, `resolveRunKind` passes unknown kinds through) and implement and push it; the gate makes that impossible.
- The predicate is mirrored wherever claimability is computed: ClaimRun, `CountOnlineWorkersClaimableForRun`, the peer mirrors in `runtime.sql`, `ListUnplaceableQueuedRunsForEphemeral` and `ListSaturationQueuedRunsForEphemeral`.
- `health.go` gains a queued reason naming the missing capability, so a run created before the worker fleet rolls is visibly waiting, not silently stuck (the worker image pin is decoupled from app releases).

### Candidate and review records

- New table `plan_reviews`, one row per review:
  - `lead_run_id`, `round` (1 in this PRD; PRD #2150 adds more), unique on `(lead_run_id, round)`; `lead_claim_generation` at submit.
  - The candidate: `plan_md`, `milestones` (jsonb), `required_capabilities`, `required_tools`, `size_class`, `base_commit` (the immutable SHA the lead planned on), `planning_diff` (bounded, scanned, below), and `candidate_digest`: sha256 over the canonical JSON of those fields, canonicalised exactly as `gatePresentedPayload.digest` (`api/internal/workersvc/gate_revision.go`), computed by the server from the stored, already-scrubbed values. The server returns the digest; the worker never supplies one.
  - `review_run_id`, `reviewer_harness`, `reviewer_model`, `reviewer_effort`: the latter two as actually delivered on the child's claim, recorded at claim time.
  - `verdict` in `pending | approve | revise | block | failed`, `reason_class`, `findings` (jsonb, bounded), `decided_at`, `deadline_at`, `created_at`.
- At most one `pending` row per lead (partial unique index).

### Child run kind `plan_review`

- A new run kind via the runbook in `api/internal/runkind` (migration widening `runs_kind_check` and `runs_kind_shape`, the const and property helpers, `fixtures/run-kinds/registry.json`, agent `RUN_KINDS` and `RUN_KIND_PROFILES`, web `RUN_KINDS`). Not judge-eligible, not `Listed` in the Runs list and never a board card, not planning-capable, and not wall-timed by the run budget (it has its own cap). A new kind rather than the task-review precedent's `task` row with `review_target_run_id` (`CreateTaskReviewRun`, created via `createRunResolved` in `workersvc/task.go`): a `task` row is listed, wall-timed and planning-capable, and the worker routes `task` claims with a review target to the diff `ReviewRunner`, so reuse would need a discriminator threaded through all of those.
- Created through `createRunResolved` / `createRunAtomic` with the reviewer harness as an explicit selection, the task-review precedent; the reviewer harness is always the opposite of the lead's `runs.harness`. Credential selection, disabled-credential refusal, the Codex capability mint, epoch and revocation, claim fences and `run_usage` attribution then come from existing code unchanged, under the child's own harness. No claim ever carries two model credentials. The child's claim carries the forge PAT for the clone, as task-review runs do today; the worker holds it, never the model.
- Created with `runs.priority = 2`, the existing expedite rank in `fn_run_priority` (`api/internal/handler/priority.go`), so a child is claimed ahead of queued leads from the same sweep.
- Repo-ful, report-only: it checks out `base_commit` under a clone key derived from its own run id (never the lead's `issue-<iid>` slug), pushes nothing, opens no merge request, and spawns no review.
- The `plan_reviews` row and the child are inserted inside the same create closure, so they commit together.
- Lifecycle: when the lead parks, is requeued, reclaimed, cancelled or reaches a terminal state, its pending child is cancelled and its row marked `failed`, `reason_class = superseded`.
- Recovery, kept deliberately simple in this PRD (one review per lead): a lead whose claim generation changed after its candidate was submitted (worker death, requeue, reclaim) never resubmits. On its next pass through the gate the submit route refuses (round 1 is decided or superseded), and the lead parks with reason `plan review: interrupted`, presenting the stored candidate. This covers an APPROVE that landed before a plan-write ACK was lost: the reclaimed lead's generation no longer matches the row, so the guarded write refuses and the lead parks. A retried submit within the same claim generation and with the same digest (a lost submit ACK) returns the existing row. PRD #2150 replaces the interrupted park with a fresh round.

### Worker routes (Bearer)

- `POST /api/worker/runs/{lead}/plan-review`: the lead's worker submits a candidate, decoded with `httpx.DecodeJSONStrict`. The server checks the lead is owned by this worker at its current claim generation, is `plan_review_required` and `auto_approve`, has no decided round 1 already, and re-checks that the Codex family is still usable (else it returns a refusal the worker turns into a `reviewer unavailable` park at once). A retry while round 1 is pending with the same digest returns the existing row and child. Returns `{round, review_run_id, candidate_digest, deadline_at}`.
- `GET /api/worker/runs/{lead}/plan-review/{round}`: verdict, reason class and findings, to the lead's owning worker only.
- `POST /api/worker/runs/{child}/plan-review-verdict`: the reviewer posts `{verdict, items[], summary}`, validated and scrubbed like `validateAndScrubTaskReview` (`api/internal/handler/task_review.go`). Accepted only when the child is `kind = 'plan_review'`, owned by this worker at its current claim generation, its row is `pending` and before `deadline_at`, and the lead's current claim generation still equals `lead_claim_generation`. Anything else is refused (409) and stored nowhere. The server, not the worker, then emits the `plan_review` run message on the lead from the stored row.

### Reviewer runner (agent)

- `PlanReviewRunner`, routed by claim kind, structured like `ReviewRunner` (`agent/src/review-runner.ts`): report `running`, one bounded review, post one structured result, report `completed`; any model error posts `failed`.
- The Codex reviewer runs on the Codex executor's broker lane, not the tool-less advice harness (which takes no tools and no working directory by contract, `codex-advice-harness.ts`). Its `RunGrants` allow only read-only tools: `Read`, plus a new bounded read-only search tool over the checkout (the broker has no `Grep`/`Glob` today; `agent/src/codex/dynamic-tools.ts`). No `Bash`, no `apply_patch`, no delegation. The native shell stays disabled (`shell_tool = false`, `agent/src/codex/config.ts`), `project_doc_max_bytes = 0` so `AGENTS.md` is not loaded, and file access is confined to the child's checkout by the same Landlock wrapper the Codex executor uses for broker tools. **This is the riskiest assumption; the implementation spikes it first** and stops to report if the broker cannot express a read-only grant set.
- Inputs: the candidate (plan, milestones, required capabilities and tools, size class), the issue title and body, the planning diff, and a fixed brief: verify cited anchors, flag regression tests that cannot fail, scope beyond the issue, gate commands with inlined environment variables, missing changelog or docs, and security or data-integrity claims stronger than their evidence. The issue body, repository content and diff are rendered as nonce-fenced untrusted data.
- Output is exactly one JSON object; anything else is `failed`, reason `malformed`.
- Model and effort: whatever the child's claim resolves through today's lanes (`GetUserHarnessModelDefaults`, the per-harness effort in `claim_effort.go`), recorded on the row as delivered. Claim assembly currently skips model lanes for Codex task reviews (`isReviewRun` in `claim_assembly.go`); `plan_review` claims take the ordinary lane path, and a test asserts it.

### Lead side (agent)

- In the autopilot branch of `gatePlan`, when `claim.plan_review_required` is true:
  1. Capture the planning diff relative to `base_commit`: tracked and untracked, excluding ignored and binary files, at most 200 untracked files, read through the hardened `runGit` with `--no-ext-diff --no-textconv`, stream-stopped at 512 KiB + 1 byte actually read (never buffered to `GIT_MAX_BUFFER` and truncated afterwards), and scanned with gitleaks before upload, as `preserved_patch` is. Over a bound, or a gitleaks hit, parks with reason `plan review: planning diff refused`.
  2. Submit the candidate, then poll the verdict every 15 s until it is decided or `deadline_at` passes.
  3. APPROVE: send the `running` report with the plan, carrying the server-returned `candidate_digest`. Anything else (REVISE, BLOCK, timeout, `failed`, a refusal, a Codex lead): take the existing forced-human-gate path (the gate with auto-approve off, as the CI-config `ci_fix` gate does), which reports `awaiting_approval` with the candidate as the plan; the server clears `auto_approve`.
- The wait does not consume the lead's run budget. While a row is pending, the live wall-deadline computation behind `RequestWallParks` (`runtime.sql`) excludes the elapsed pending time, so a lead near its budget is never wall-parked mid-review; when the wait ends (decided, superseded or past deadline), the wait is banked into `budget_paused_seconds` exactly once, as human-gate parks are. While a row is pending, `health.go` reports `waiting for plan review` instead of the no-updates stall, and no stall nudge is sent.

### Server enforcement

`SetRunRunning` and `SetRunAutopilotPlan` (`api/internal/store/queries/runtime.sql`) change for a run with `plan_review_required = true` and `auto_approve = true`:

- `SetRunAutopilotPlan` stays write-once and gains a `@candidate_digest` parameter. Its guarded UPDATE requires, in the same statement, the run's latest `plan_reviews` row to have `verdict = 'approve'`, `reviewer_harness <> runs.harness`, a `review_run_id` whose run is `kind = 'plan_review'` with that harness, `lead_claim_generation = runs.claim_generation`, and `candidate_digest = @candidate_digest`, and the submitted plan, milestones, capabilities, tools and size class to equal the stored candidate's. An APPROVE of an older round never satisfies a newer one.
- `SetRunRunning` writes inferred capabilities, tools, size class and frozen milestones only in the transaction of a guarded plan write that passed, and refuses any change to them once `plan_md` is set.
- `SetRunCompleted` and milestone-progress writes refuse while `plan_md` is NULL.
- A refused plan write is today's rowcount-zero refusal, which the worker already treats as fatal ("autopilot plan not durably stored"), so the run fails rather than implements. The guard is a backstop on what the server stores and completes; it cannot stop a worker that ignores the refusal from running code, which is why the claim gate keeps unaware workers out and the worker obeys the refusal.

### Visibility

- Web: an `ActivityFeed` case for `plan_review` messages; `PlanPanel` (`web/src/pages/runView/PlanPanel.tsx`) shows the latest review's reason and findings when the run is parked after a review, through the hardened Markdown renderer; the lead's run page links the child run and its cost.
- CLI: `uzi run get` prints the reason and findings through `renderer.Plain` (`.claude/rules/tui.md` D7).
- Slack: `gateBlocks` (`api/internal/slacksvc/gate.go`) gains one line, `Plan review: <reason>`, and no findings text.

### Resource bounds (plan, issue body, planning diff, repository content and reviewer output are untrusted)

| Resource | Bound | Enforced in |
|---|---|---|
| Reviews per lead | 1 in this PRD | submit route predicate |
| Concurrent reviews per lead | 1 | partial unique index |
| Candidate plan_md / milestones | 256 KiB / 64 items; over cap parks, never retries | submit route (`DecodeJSONStrict`) |
| Planning diff | 512 KiB read, 200 untracked files, no binaries, gitleaks-clean | lead capture; submit route re-checks size |
| Wait for a verdict | `PLAN_REVIEW_TIMEOUT`, default 30 min, max 2 h, stored as `deadline_at` | submit, verdict route, lead poll |
| Lead poll rate | every 15 s | lead |
| Reviewer model turn | `PLAN_REVIEW_MODEL_TIMEOUT`, default 15 min (the text-only `REVIEW_MODEL_TIMEOUT_MS` is 5 min; a tool-using review needs more) | `PlanReviewRunner` |
| Reviewer tool calls | 200 per review | broker grant |
| Findings | 20 items, 2 KiB each, 32 KiB total, summary 4 KiB | verdict route |
| Worker slots | normal concurrency caps, no exception | existing claim path |

Boot validation refuses malformed or out-of-range knobs, and `PLAN_REVIEW_TIMEOUT` must be below `RUN_TIMEOUT`.

## Testing decisions

- **Server enforcement (LiveDB via `./e2e/run-store-it.sh`):** on a review-required run, `SetRunAutopilotPlan` refuses with no review, pending, REVISE, BLOCK, an APPROVE whose candidate differs in milestones, tools or size class only, an APPROVE from a same-family reviewer, an APPROVE for a superseded lead claim generation, and a mismatched digest; it accepts only the matching APPROVE. `SetRunRunning` refuses to set capabilities, tools, size class or milestones without a passing plan write; `SetRunCompleted` refuses with `plan_md` NULL. Not-required runs are unchanged (follow `setrunautopilotplan_livedb_test.go`, `TestAutopilotPlanWriteAtomicWithRunningLiveDB`). Each guard has a mutation that removes it and reddens a case.
- **Snapshot:** the INSERT parity test, and the `PlanReviewable` set pinned.
- **Claim gates:** neither a review-required run nor a `plan_review` run is ever claimed by a worker without `plan_review_v1`, and every mirror agrees; the health reason appears.
- **Lifecycle and fences:** verdict after deadline, after lead park, cancel, requeue or reclaim, and from a stale child or lead claim generation are refused and change nothing; a retried submit with the same digest reuses the row; the pending child is cancelled on every lead exit; the child sorts ahead of queued leads.
- **Worker (`agent/test/runner-plan-gate.test.ts` style):** APPROVE implements; REVISE, BLOCK, timeout, malformed, refused, Codex lead and planning-diff refusal each park through the forced-gate path with the right reason; a not-required autopilot claim behaves exactly as today.
- **Wall budget (LiveDB):** a lead with less budget left than the review wait is not wall-parked while its row is pending; the wait is banked into `budget_paused_seconds` exactly once on each way the wait ends (decided, superseded, deadline), with a mutation that double-banks or skips the live exclusion reddening a case.
- **Reviewer isolation:** asserted on the built configuration: the Codex reviewer's `RunGrants` contain only the read tools, native shell off, `project_doc_max_bytes = 0`, Landlock rooted at the child checkout; a read outside the checkout is refused.
- **Custody unchanged:** a Claude lead's claim carries no Codex secret; the child's claim carries no Anthropic token; the child's usage lands in its own `run_usage` rows under Codex; the child claim takes the ordinary model lane and the recorded model and effort equal the claim's.
- **Planning diff:** the stream stop at the byte bound, the file-count cap, binary exclusion, and a gitleaks-detected fixture assembled at runtime (`.claude/rules/prds.md`).
- **Web and CLI:** feed and panel rendering of findings, including hostile Markdown; `uzi run get` through `renderer.Plain`; mock-mode fixture.

## Milestones

- [ ] **M1: Opted-in auto-approved runs wait at a fail-closed review gate.** Setting and route, run snapshot with its parity test and `PlanReviewable`, the `plan_reviews` table (its migration lands here because the guards below read it; no row is written until M2), both claim gates with mirrors and the health reason (the `plan_review` kind's claim gate lands with the kind in M2), server enforcement in `SetRunAutopilotPlan` / `SetRunRunning` / `SetRunCompleted`, and the lead-side branch that, with no reviewer yet, parks every review-required run with `plan review: reviewer unavailable` through the forced-gate path, with the Slack reason line, feed message and CLI line. Docs: a new `docs/plan-review.md` (audience `user`, stating this PRD reviews Claude-lead runs only and Codex-lead runs park), `docs/autopilot.md`, `docs/scheduling.md`, `docs/configuration.md`, `docs/slack.md`, then `task docs:sync`; `specs/human.md` (amend the l.201 "zero uzi interaction" autopilot promise for opted-in users); CHANGELOG. Blocked by: none. Gates: `task gate:api`, `task gate:agent`, `task gate:web`, LiveDB via `./e2e/run-store-it.sh`, `task gate:repo`.
- [ ] **M2: A Claude lead's plan is reviewed on Codex; APPROVE implements, anything else parks with findings.** Spike the read-only broker grant set first. Then `plan_reviews`, the `plan_review` kind (full runbook), the three worker routes, `PlanReviewRunner` (Codex), planning-diff capture, priority, lifecycle and fences, wait-time crediting, `PlanPanel` findings and run-page child link, `uzi run get` findings, `docs/plan-review.md`, `docs/run-activity.md`, `docs/cli.md` then `task docs:sync`, `ARCHITECTURE.md` (autopilot section), an ADR for the server-enforced review seam (adr/2149-cross-family-plan-review.md), CHANGELOG. Blocked by: M1. Gates: as M1.

No `.github/workflows/**` change in implementation or validation.

## Acceptance (hosted k8s, maintainer-owned)

Tracked in the `acceptance` issue #2152: a reviewed Claude-lead sweep run is reviewed by a Codex child on a Codex-capable worker and implements after APPROVE; a REVISE or BLOCK parks it with the reason visible on web, CLI and Slack; with no free Codex-capable slot it parks on timeout; an ephemeral worker provisioned for the child after the saturation debounce is reviewed normally; the child's usage is attributed to the user's Codex credential. Implementation may merge before it; the PRD moves to `prds/done/` only after it.

## Decision Log

- **D1. A child run on the other harness, not a second credential on the lead's claim.** Rejected: one claim carrying both credentials. `runs_codex_harness_coherence_check` (migrations 00226/00227) forbids Codex binding columns unless `harness = 'codex'`; the worker picks its executor by the presence of `claim.secrets.codex`; the claim-time disable re-check in `claim_recovery.go` switches on the lead's harness, so a Codex lead's Anthropic reviewer would go unchecked; `run_usage` derives harness and cost from `runs.harness` (`usage_fold.go`); a Claude-lead run never reaches the `codex_harness_v1` claim gate. About twenty harness branch sites would change. A child run reuses that custody unchanged.
- **D2. Review, not co-planning.** The hand-steered shape; about one extra turn per plan.
- **D3. The reviewer reads the repository through read-only broker tools under Landlock.** Rejected: a text-only review like `ReviewRunner`, since checking anchors against code was most of the value; and the tool-less advice harness, which cannot read files by contract. Read-only tools alone do not stop network use or instruction loading, so the native shell and `AGENTS.md` loading are off and asserted.
- **D4. Keep worker concurrency caps; park on timeout; expedite the child.** Rejected: letting the lead's worker claim its child past its cap, because a SQL eligibility exception does not prove the worker's scheduler can run another run or that it runs Codex. Rejected for now: a durable wait that releases the lead's slot. User decision 2026-10-03.
- **D5. The candidate is stored before the child exists; approval binds to the latest round's server-computed digest and the lead's claim generation.** The approved `plan_md` stays write-once, so the candidate lives in `plan_reviews`; the digest covers every approval-bearing field (`gatePresentedPayload` includes size class) plus the base commit and planning diff.
- **D6. The server is the backstop on what is stored and completed.** It guards the plan write, the approval-bearing fields `SetRunRunning` would otherwise write on any report, and completion. The claim gate keeps unaware workers out; a refused write is fatal to the run.
- **D7. Every non-APPROVE outcome parks.** Nothing proceeds with dissent. Automatic revision is PRD #2150, which adds the fenced untrusted-feedback prompt it needs: today's revise prompt tells the lead the feedback is an authoritative human instruction (`buildRevisePlanPrompt`, `agent/src/prompt.ts`).
- **D8. Snapshot the requirement on the run, computed inside the INSERT.**
- **D9. Per-user opt-in, cookie-only, refused without both families.** Follows the judge and autopilot consent switches.
- **D10. Slack shows the reason only.** Findings can quote repository content; the Slack content-minimisation rules in `ARCHITECTURE.md` allow only whole-plan exceptions.
- **D11. A new run kind, not a `task` row.** See the child run kind section.
- **D12. Split into three PRDs.** User decision 2026-10-03: this PRD (one review, Claude lead), PRD #2150 (automatic rounds, Codex lead), PRD #2151 (pins).
