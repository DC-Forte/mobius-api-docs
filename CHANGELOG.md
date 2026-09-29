---
layout: default
title: Changelog
---

# Changelog — Hiresense Handoff API

All notable changes to the Mobius-facing handoff API ([openapi/mobius-handoff-v1.yaml](openapi/mobius-handoff-v1.yaml),
[guide.md](guide.md)) are recorded here. Versions follow the OpenAPI spec's own
`info.version`.

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
