---
layout: default
title: Changelog
---

# Changelog — Hiresense Handoff API

All notable changes to the Mobius-facing handoff API ([openapi/mobius-handoff-v1.yaml](openapi/mobius-handoff-v1.yaml),
[guide.md](guide.md)) are recorded here. Versions follow the OpenAPI spec's own
`info.version`.

## Unreleased — recruiter report v3 (proposed, not yet sent to any partner)

### Added

- New page: [recruiter-report.md](recruiter-report.md) — the JD-vs-resume recruiter report
  (`POST /recruiterReport` and its webhook), which this API reference did not cover before, and
  **report v3** (`report_metadata.schema_version` `"4.0"`): `categories[].level`,
  `present_listed` as `{ skill, source }` objects, `related` and `present_related`
  (must-have skills only), `experience_background.experience_level`, `candidate_background[].relevant`, and new scoring
  rules (weights 40/20/10/30, listed and related skill credit, experience and education as
  levels). Includes the Mobius `recruiter_report/v3.py` changes and the rollout order.

### Changed

- Hiresense keeps sending today's shape (`schema_version` `"3.1"`) until Mobius confirms v3 is
  deployed. `present_listed` changing from names to objects is breaking, so v3 is a new schema
  version on the Mobius side.

## Unreleased — practice reports (not yet enabled for any partner)

### Added

- `candidateReport` on the webhook body, `GET /internal/handoff/report` and the embedded
  `.../feedback` response: for a session created with `isPractice: true`, a candidate-facing
  coaching report delivered under its own key, never inside `reportV2`. Carries no
  `proctoring_summary` and is not held for proctoring data. Off until enabled per integration
  (`PRACTICE_CANDIDATE_REPORT`); partners must accept `candidateReport` before they send
  `isPractice: true`.

### Changed

- `POST /internal/handoff` returns `409` when the `externalCandidateId` + `externalJobId` pair
  already has a session with a different `isPractice` value. A practice run needs its own
  `externalJobId`. Previously the existing session was silently resent.

## 1.5.1 — 2026-11-16

### Changed

- **Live transcript: a one-time backfill is no longer enough.** `GET /ws/transcript/{interviewId}`
  is live only — a turn saved while your socket was down is never replayed. Merge the
  `conversation` backfill and the socket by turn `id`, reconnect the socket with backoff, and
  re-run the backfill after every reconnect and about every 30 s while the interview is live. See
  "Keeping the live transcript in sync" in the integration guide.

### Fixed (documentation)

- `TranscriptMessage.type` is `"user"`, `"ai"` or `"system"`. It was documented as
  `"candidate"` or `"assistant"`, which the API never returns.

## 1.5.0 — 2026-10-21

### Added

- Webhook envelope: optional `candidateDisconnects` (array of `{ leftAt, returnedAt | null,
  durationSeconds }`) and `totalDisconnectedSeconds` — every time the candidate left the
  interview room mid-interview and for how long. Omitted when they never dropped.

### Changed

- A candidate who drops is now given up to 5 minutes 30 seconds (was 90 seconds) to reconnect to
  the same interview; the AI interviewer holds meanwhile (no nudges, no questions, interview
  time paused). One who never returns is reported as `connectionEndReason: "dropped"`.
- Time the candidate spent disconnected is excluded from the interview duration used for the
  5-minute minimum and from the report's duration, so a short interview can no longer clear the
  minimum on the strength of a long absence.
- `POST .../end` now also closes the interview room immediately.

## 1.4.3 — 2026-10-07

### Changed

- `GET /internal/handoff/report` now applies the same completeness gate as the webhook: it
  returns `404` ("not ready") until proctoring has finished resolving — both the browser-session
  report and the video analysis — instead of returning the report as soon as it is generated.
  Previously a pull made right after generation got a report with no `proctoring_summary` and
  no signal that it was incomplete. Proctoring typically resolves a few minutes after the
  interview ends; the report is released regardless one hour after the interview ends. Treat
  this `404` like every other "not ready" `404` on this endpoint — keep polling. No schema
  change.

## 1.4.2 — 2026-10-03

### Changed

- `proctoring.summary` (both the top-level `reportV2.proctoring_summary` and each
  `per_question_breakdown[]` entry's own `proctoring.summary`) is now a ranked list of named
  findings (e.g. `"Mobile phone detected, Candidate looking away from screen + 2 more"`, or
  `"No events recorded"`), not a duration-phrased sentence (`"120 second answer, looked away for
  18 seconds, mobile phone detected"`). If you were pattern-matching on the old shape, update to
  the new one — the field is still a plain string, safe to render as-is either way, just a
  different string.
- Proctoring status/summary now also incorporates the browser-SDK session's own events
  (tab-switching, losing window focus, copying text) alongside video analysis — previously
  video-only. No schema change; this can only make `status` more severe than before for an
  interview where SDK data exists, never less.

## 1.4.1 — 2026-10-01

### Added

- `GET /internal/handoff/interviews/{interviewId}/proctoring-log` response now includes a
  top-level `status` field (`not_started` / `in_progress` / `ended` / `analyzing` / `completed` /
  `failed`) — a live-ish proctoring status for your backend to poll, without needing to run the
  incProc SDK client-side. Derived from Hiresense's own persisted state, not a live incProc call.

## 1.4.0 — 2026-09-30

### Added

- `POST /internal/handoff/interviews/{interviewId}/regenerate-report` lets you retry report
  generation after it failed on our side (transient LLM error, timeout, etc.) — nothing
  previously retried that automatically. Returns `202` and restarts generation asynchronously;
  `409` if the report already completed, is still generating, or was gated out by the 5-minute
  minimum (that last case is permanent by design, not a failure to retry).

## 1.3.0 — 2026-09-29

### Added

- The `end_reason` from 1.2.0 is now propagated back to you: `GET .../feedback` includes it as
  top-level `endReason` on a ready report and as `end_reason` on the insufficient_data 404; the
  webhook/pull-report envelope includes it as `endReason`, alongside the existing
  `connectionEndReason` (a different, AI-interviewer-inferred signal) — and unlike that field,
  `endReason` can appear on an `endedEarly` payload too.

## 1.2.0 — 2026-09-29

### Added

- `POST /candidate-api/v1/interviews/{interviewId}/end` now accepts an optional JSON body
  `{ "reason": "<value>" }` — `user_initiated`, `camera_off`, `mic_off`, `connection_issue`,
  `timeout`, or `other`. Lets an embedding partner tell us why they ended the call instead of
  us only knowing it happened. Omitting it (or the whole body) is unchanged from before. An
  unrecognized value returns `400 invalid_reason` and leaves the interview untouched.

## 1.1.0 — 2026-09-29

### Added

- `POST /internal/handoff/interviews/{interviewId}/pause` and `.../resume` — freeze/unfreeze
  the AI interviewer mid-call (e.g. the candidate's camera or mic drops). No new questions
  asked and no silence-nudging while paused; the interview's overall time budget is **not**
  frozen and keeps counting through a pause. Optional `message` body field lets the AI
  interviewer announce the pause/resume to the candidate verbatim.

## 1.0.0 — 2026-09-08

### Added

- Initial public release: `POST /internal/handoff` (start/resend a session), `GET
  /internal/handoff/report` (pull the scored report), the reportV2/webhook payload shape,
  `GET /internal/handoff/interviews/{interviewId}/clips`, `.../recording`, and
  `.../proctoring-log`.
