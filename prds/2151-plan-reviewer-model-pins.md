# PRD #2151: Plan reviewer model and effort pins per model family

**Status**: Draft. Child 3 of 3 under umbrella #2148. Blocked by PRD #2149.

Resolved facts below were read at `main` `8a5f138e`.

## Problem

PRD #2149 reviews an auto-approved run's plan on the other model family, using whatever model and effort the reviewer's claim resolves from the owner's worker lanes. A user who wants a different reviewer than their worker default (for example `gpt-6-astra` at `xhigh` for Codex reviews while their Codex workers stay on their default) cannot say so without changing every run they start on that family.

## Outcome

On Settings → Run defaults, next to the plan-review toggle, the user sees two reviewer rows, "Reviewer on Claude" and "Reviewer on Codex", each with a model and a reasoning effort. Each field shows `Default · <value> (<source>)` and follows the user's worker default for that family until the user picks a value, which pins it. A pin is hard: a review either runs on exactly the pinned model and effort or fails visibly. The values a review actually ran with are recorded on it and shown with its findings.

Acceptance examples:

1. Reviewer on Codex is pinned to `gpt-6-astra` / `xhigh`; the Codex worker default is `gpt-6-sol` / `high`. A Claude-lead autopilot run's plan is reviewed by a Codex child on `gpt-6-astra` at `xhigh`; the review record and run page show both. A Codex-lead run's own implementation still runs on `gpt-6-sol` / `high`.
2. Reviewer on Claude is left at Default. The user changes their Claude worker default from `opus` to `sonnet`. The next Claude review runs on `sonnet`.
3. The Codex reviewer is pinned to a custom model and no online worker advertises `codex_custom_model_v1`. The child stays queued (the placement gate keeps other workers from claiming it) until the review deadline, and the run parks with `plan review: timed out`. Separately, if claim assembly cannot honour a pin it is handed (a custom model the user's Codex account can no longer use, or a stored effort the harness no longer accepts), the child fails with `plan review: reviewer unavailable` and the run parks. Neither path ever runs the default model or a clamped effort.

## Out of scope

- The epic #1703 Settings → Models grid. When that tab ships (child I) these rows move into it as a "Plan reviewer" row; epic #1703 is updated to list it.
- A per-run reviewer override (epic #1703 locked "no per-run model or effort override").
- The `uzi handoff --review` model (#1570): same shape, separate setting.
- Admin instance defaults for the reviewer.

## Modules and seams

- **Settings fields** (`users`): `plan_review_claude_model`, `plan_review_claude_effort`, `plan_review_codex_model`, `plan_review_codex_effort`, all nullable; null is Default. On the `/api/me/settings` DTO (`api/internal/handler/user_settings.go`): GET readable with a CLI token, PUT cookie-only, as today.
- **Write validation**: model ids pass `agenttmpl.ValidateModel` (`api/internal/agenttmpl/model.go`, rejects control and format characters) and are valid for their family: Claude and curated Codex ids by `harnessModelCompatible` (`api/internal/workersvc/harness_create.go`); custom Codex ids by the same custom-lane validation the Codex worker-model lane uses (PRD #1551), since `harnessModelCompatible` rejects custom Codex ids. An effort must be in that harness's effort domain. Invalid input is a 400 naming the field.
- **Delivery, a new branch in claim assembly for `kind = 'plan_review'`**: the pin is read from the owner's row at claim time and delivered on the existing `default_model` and effort claim fields. The frozen `runs.model` path is not used: a frozen custom Codex model is never forwarded (`harness_create.go`) and claim assembly falls back to the lane with a note (`claim_assembly.go`), and effort has no per-run freeze (`claim_effort.go`). The branch has no fallback: a pin the claim cannot honour fails the claim assembly for that child with a typed reason that ends the child as `failed`, reason `reviewer unavailable`, and PRD #2149 parks the lead. Default (null) takes PRD #2149's lane resolution unchanged.
- **Placement**: a `plan_review` child with a custom Codex pin is claimable only by a worker advertising `codex_custom_model_v1`, mirrored in ClaimRun, `CountOnlineWorkersClaimableForRun` and the ephemeral provisioning queries, following the custom-root clause in `runtime.sql`.
- **Record**: the `plan_reviews` row stores the model and effort the claim delivered (PRD #2149) and, here, their source (`pin` or `worker default`).
- **Web**: a "Plan reviewer" section on `web/src/pages/RunDefaults.tsx`, two rows of model and effort selects with epic #1703's `Default · value (source)` wording; mock-mode fixture.
- **CLI**: the account settings view shows the four values and their sources; writes stay in the web UI, as for the other `/me/settings` fields. Check `api/cmd/uzi/` and `docs/cli.md`.

## Testing decisions

- Resolution: a pin wins; Default follows a later change of the worker default; the delivered values equal the recorded ones.
- Hard pins: an unsupported custom Codex model, a model incompatible with the harness, and an out-of-domain effort each fail the child with `reviewer unavailable` and never run the default or a clamped value. Regression tests that fail if the frozen-model fallback or effort clamping is reused.
- Placement: a custom-pinned child is never claimed by a worker without `codex_custom_model_v1`; mirrors agree.
- Validation: each field's 400 cases; curated and custom ids accepted per family; control characters refused.
- Web and CLI: rendering of `Default · value (source)`, atomic save, CLI view.

## Milestones

- [ ] **M1: A user pins the reviewer model and effort per family, and reviews run on exactly those or fail visibly.** Migration, DTO and validation, the `plan_review` claim-assembly branch, placement clause and mirrors, recording with source, Run defaults section, CLI view, `docs/plan-review.md` and `docs/cli.md` then `task docs:sync`, `specs/human.md`, CHANGELOG. Blocked by: PRD #2149. Gates: `task gate:api`, `task gate:web`, `task gate:agent`, LiveDB via `./e2e/run-store-it.sh`, `task gate:repo`.

No `.github/workflows/**` change in implementation or validation.

## Decision Log

- **D1. A Claude/Codex pair; Default follows the worker default.** Epic #1703's rule for every model setting, and the user's request: keep the worker default or override it.
- **D2. Pins are hard, for model and effort alike.** Epic #1703's judge decision: an explicit choice that cannot be honoured is a visible failure, never a substitution.
- **D3. Delivered through a dedicated claim-assembly branch, not the frozen `runs.model` path.** That path drops custom Codex ids with a fallback and cannot carry effort.
- **D4. No per-run override.** Locked by epic #1703; these are mostly unattended runs.
- **D5. Separate PRD.** The review is useful on worker defaults; user decision 2026-10-03.
