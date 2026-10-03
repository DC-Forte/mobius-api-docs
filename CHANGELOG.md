---
layout: default
title: Changelog
---

# Changelog — Hiresense Handoff API

All notable changes to the Mobius-facing handoff API ([openapi/mobius-handoff-v1.yaml](openapi/mobius-handoff-v1.yaml),
[guide.md](guide.md)) are recorded here. Versions follow the OpenAPI spec's own
`info.version`.

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
