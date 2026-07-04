# Audit: Productivity Review ("Accountability") Subsystem

**Date:** 2026-07-04
**Scope:** The productivity review subsystem — the feature the design system calls the
"yellow accountability state on source issues" (`ui/storybook/stories/status-language.stories.tsx`)
and the release notes describe as "Productivity review surfaces stalled work" (`releases/v2026.428.0.md`).
This is the closest thing in the codebase to an "accountability app": it watches agent-assigned
issues for unproductive patterns, opens review issues for a manager, and soft-holds unproductive
run continuations while a review is open.

**Files audited:**

- `server/src/services/productivity-review.ts` (core service)
- `server/src/services/heartbeat.ts` (continuation hold + scheduler wrapper)
- `server/src/services/issues.ts` (`listIssueProductivityReviewMap`, read projection)
- `server/src/routes/issues.ts` (API exposure)
- `server/src/index.ts` (scheduler wiring), `server/src/config.ts`
- `packages/db/src/schema/issues.ts` (partial unique index), migration `0074_striped_genesis`
- `packages/shared/src/types/issue.ts`, `packages/shared/src/constants.ts`
- `ui/src/components/ProductivityReviewBadge.tsx`, `ui/src/components/IssueRow.tsx`, `ui/src/pages/IssueDetail.tsx`
- `server/src/__tests__/productivity-review-service.test.ts`
- `doc/execution-semantics.md`, `doc/PRODUCT.md`

## How it works (summary)

Every heartbeat scheduler tick, `reconcileProductivityReviews` scans up to 250 agent-assigned
`todo`/`in_progress` issues (oldest `updatedAt` first) and evaluates three triggers per issue:

1. **`no_comment_streak`** — ≥10 consecutive terminal issue-linked runs with no run-created issue comment.
2. **`high_churn`** — ≥10 runs or assignee run-comments in 1h, or ≥30 in 6h.
3. **`long_active_duration`** — issue `in_progress` for ≥6h (from `startedAt ?? executionLockedAt`).

On trigger, it opens a child review issue (`originKind = issue_productivity_review`) assigned to
the first invokable, non-budget-blocked candidate among: the agent's manager, the issue creator,
the project lead, then CTO/CEO — and wakes that owner with a cheap model profile. While a
`no_comment_streak`/`high_churn` review is open, `plan_only`/`empty_response` run continuations on
the source issue are held (soft stop) instead of auto-requeued.

Guardrails present: one open review per source issue enforced by partial unique index
(`issues_active_productivity_review_uq`) with a 23505 race fallback; 6h snooze after a review is
marked `done`; refresh comments capped at 3, minimum 1h apart; creation capped at 3 per source
issue per rolling 24h; recursion guard so reviews of review-descendants are never spawned;
poisoned `requestDepth` clamped; per-candidate failure isolation.

## Findings

### F1 (High, behavioral): Cancelling a review restarts it within one scheduler tick, forever

`findRecentResolvedProductivityReview` snoozes only on `status = 'done'`
(`productivity-review.ts:277`), and `countRecentProductivityReviews` explicitly excludes
`cancelled` from the 24h creation cap (`productivity-review.ts:302`, asserted by the test
"does not count cancelled productivity reviews toward the creation cap"). Combined effect: a
board user or manager who **cancels** a review — the natural "dismiss, this is noise" action —
gets a brand-new review, with a fresh owner wake-up (budget spend), on the next reconcile pass.
The pass runs every `heartbeatSchedulerIntervalMs` (default **30s**, `config.ts:333`). While the
trigger condition persists (e.g. a legitimately long-running quiet task), this loops indefinitely:
cancel → 30s → new review → wake manager → cancel → …

**Recommendation:** treat a recently-cancelled review as a snooze (same 6h window as `done`), or
count cancelled reviews toward the creation cap. Cancel and done should both mean "a human/manager
looked at this; stop re-raising it for a while."

### F2 (High, behavioral): The reviewed agent can be assigned its own productivity review

`resolveReviewOwnerAgentId` (`productivity-review.ts:525`) builds the candidate list from
`reportsTo`, `createdByAgentId`, project lead, then CTO/CEO — and never excludes
`sourceAgent.id`. Agents routinely create issues assigned to themselves, and top-of-chain agents
(CEO/CTO) have no manager. So the very agent whose work is flagged as unproductive can be picked
as the review owner — e.g. a CEO stuck in a no-comment loop reviews itself, while its own
continuations on the source issue are simultaneously being held. That defeats the accountability
purpose and, for a stuck agent, tends to produce a stuck review.

**Recommendation:** skip candidates equal to `sourceAgent.id`; if no other candidate survives,
leave the review unassigned (board-visible) rather than self-assigned.

### F3 (Medium, availability): Candidate-window starvation in the reconcile scan

The scan takes the 250 oldest-`updatedAt` eligible issues across **all companies**
(`productivity-review.ts:768–782`). Skipped candidates (no trigger, agent missing, descendant of
a review, snoozed) are not touched, so their `updatedAt` never moves and they occupy the same 250
slots on every pass. A deployment with >250 old, assigned, never-triggering issues (e.g. stale
assigned `todo` work with no runs) permanently starves newer stalled work — and one such company
starves every other company on the instance. The MAX_CANDIDATE_ISSUES cap silently truncates with
no logging.

**Recommendation:** iterate with a persisted cursor/watermark (or per-company round-robin), and
log when the candidate window is full so truncation is observable.

### F4 (Medium, performance/concurrency): Full scan every 30s, N+1 queries, no overlap guard

- The reconcile pass runs at the heartbeat scheduler cadence (default 30s, floor 10s) at
  `index.ts:824`, though all of its write-side guardrails are tuned in hours. Each pass performs,
  per candidate: a parent walk of up to 25 sequential queries
  (`isProductivityReviewDescendant`), plus ~8 evidence queries — several of which filter
  `heartbeat_runs` on JSONB expressions (`contextSnapshot->>'issueId'`, `->>'taskId'`,
  `->>'taskKey'`, `productivity-review.ts:99–105`) that have no expression index. At 250
  candidates that is thousands of queries every 30 seconds.
- `setInterval` in `index.ts:766` fires unconditionally; if a pass (or the recovery chain ahead
  of it) takes longer than the interval, passes overlap. Creation dupes are protected by the
  unique index, but the refresh-comment path is read-check-write with no lock
  (`createOrUpdateReview`, `productivity-review.ts:640–668`), so overlapping passes (or multiple
  server instances) can double-post refresh comments within one interval.

**Recommendation:** run the productivity pass on its own, longer interval; add an in-flight guard
(skip if previous pass still running); add an expression index on
`(heartbeat_runs.contextSnapshot->>'issueId')` or promote a real `issue_id` column.

### F5 (Medium, fairness of the metric): "Progress" only counts run-created issue comments

The no-comment streak breaks only on comments with `createdByRunId` pointing at one of the
sampled runs (`productivity-review.ts:406–430`). But the agent operating contract (the
`paperclip` skill, Step 7) explicitly blesses leaving durable progress in "comments, issue
documents, or work products." An agent that diligently updates the `plan` document, uploads
attachments, or advances status on every run — but doesn't comment — accrues a streak, gets
flagged, and has its `plan_only`/`empty_response` continuations held. The evidence definition is
narrower than the contract agents are told to follow.

**Recommendation:** count document revisions, attachment uploads, and status transitions made by
a run as streak-breaking evidence (or align the skill text so comments are the required liveness
signal).

### F6 (Medium, operability): No production tuning or per-issue exemption

`reconcileProductivityReviews` accepts threshold overrides, but the heartbeat wrapper
(`heartbeat.ts:6862`) and both call sites in `index.ts` pass only `now`/`companyId` — there is no
env var or instance setting for any threshold. The review body invites the manager to "continue
with a snooze window," but the only snooze is the fixed 6h after `done`. A legitimately
multi-day task therefore generates up to ~3 reviews per 24h indefinitely, with no way to say
"this issue is expected to run long" short of code changes. Also, `readPositiveInteger` maps 0 or
negative overrides back to the defaults (`productivity-review.ts:132`), so a trigger cannot be
disabled even programmatically.

**Recommendation:** expose thresholds via config/instance settings; add a per-issue snooze-until
or exemption field the review owner can set; let an explicit 0 disable a trigger.

### F7 (Low): Refresh-state detection via `LIKE` on comment body

`getRefreshCommentState` counts comments whose body starts with
`"Productivity review evidence refreshed."` (`productivity-review.ts:320`). Any human or agent
comment that happens to start with that sentence counts toward the 3-refresh cap and resets the
refresh clock. A structured marker (metadata column, comment kind, or system-author flag) would
be robust.

### F8 (Low): Prompt-injection surface into the review owner

`buildReviewMarkdown` embeds the reviewed agent's latest comment bodies and free-text
`nextAction` strings (truncated but otherwise verbatim) into the description of an issue that is
auto-assigned to a manager agent and auto-wakes it. A misbehaving or compromised worker agent can
therefore place crafted text directly into a supervisor's task description. This is systemic to
the product (managers read worker comments anyway), but this path is *automatic escalation* to a
more privileged agent, so it deserves a note in the threat model and, ideally, clear
"quoted evidence" fencing in the generated markdown.

### F9 (Low): Minor code-quality items

- `PRODUCTIVITY_REVIEW_ORIGIN_KIND` is re-declared as a string literal in
  `services/issues.ts:1170` instead of importing `RECOVERY_ORIGIN_KINDS.issueProductivityReview`.
- `getCompanyIssuePrefix` silently falls back to `"PAP"` (`productivity-review.ts:212`), which
  would render wrong deep links for a missing/renamed company rather than failing loudly.
- `addRefreshComment` rewrites the comment's `createdAt`/`updatedAt` after insert
  (`productivity-review.ts:337–346`) — works, but two extra updates per refresh and a
  slightly surprising audit timestamp mutation.

## What's in good shape

- **Tenant isolation is consistent.** Every query is company-scoped; agent lookups re-verify
  `companyId` before use (owner resolution, continuation hold).
- **Duplicate prevention is done right**: DB-level partial unique index plus a 23505 race
  fallback that re-reads the winner, rather than app-level checks alone.
- **Recursion and blast-radius controls**: review-descendant walk (depth-capped at 25),
  `requestDepth` clamping (tested against poisoned values), per-candidate try/catch with
  `failed`/`failedIssueIds` counters so one bad row can't abort the pass.
- **Cost awareness**: review issues are assigned with the `status_only` recovery profile
  (cheap model), and the continuation hold is deliberately a *soft* stop scoped to
  `plan_only`/`empty_response` continuations — `long_active_duration` intentionally never holds
  continuations (tested).
- **UI**: badge and row indicator are accessible (aria-labels, tooltips), read from a single
  shared type, and only active reviews are surfaced; the API read path batches lookups in chunks
  and attaches data only when present.
- **Test suite** (11 embedded-Postgres integration tests) covers creation, dedup, refresh
  rate-limit and cap, creation cap, snooze, recursion guard, non-assignee churn exclusion,
  continuation-hold logging, and requestDepth clamping.

## Test gaps worth closing

- Owner-resolution fallback chain (manager missing/paused → creator → project lead → CTO/CEO),
  including the self-assignment case in F2.
- The cancel-recreate loop in F1 (current behavior is only indirectly asserted via the cap test).
- Concurrent/overlapping reconcile passes (refresh-comment duplication in F4).
- Candidate-window truncation behavior (F3).

## Suggested priority

1. F1 and F2 — small, self-contained fixes in `productivity-review.ts` with high behavioral payoff.
2. F4's in-flight guard + separate interval — cheap insurance for larger deployments.
3. F5/F6 — product decisions (evidence definition, tunability) worth a maintainer discussion first.
4. F3, F7–F9 — opportunistic.
