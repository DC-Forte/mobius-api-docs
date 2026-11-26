---
layout: default
title: Recruiter Report
wide: true
---

# Hiresense Recruiter Report (JD vs resume) — Integration Guide

You send a job description and a candidate resume; Hiresense scores the fit and
sends back a recruiter-facing report by webhook. This page covers the request,
the webhook, the report shape, and **report v3**, a new version of the shape
that Hiresense proposes and that Mobius needs to add before Hiresense sends it.

> **Status of v3: proposed, not live.** Hiresense keeps sending today's shape
> (below, "v2 shape") until Mobius confirms v3 is deployed. Nothing in this
> page changes what you receive today. The v3 shape is final unless this
> page says otherwise; the open points are listed at the end.

This is a different API from the interview handoff in the
[interview handoff guide](guide.md). It has its own host, its own request and its
own webhook.

## Flow

1. Mobius POSTs the resume and job to Hiresense (`POST /recruiterReport`).
2. Hiresense answers at once with the job status. The report is built in the
   background (typically a few minutes).
3. When it is ready, Hiresense POSTs `{ "reportId", "report" }` to your
   webhook. A `2xx` response marks it delivered.

## 1. Request a report — `POST /recruiterReport`

| Environment | Host |
| --- | --- |
| Production | `https://recruiter-report-api.dc-forte.com` |
| Staging | `https://recruiter-report-api-staging.dc-forte.com` |

Header: `X-Internal-Token: <shared secret>`.

Body (unchanged by v3):

```json
{
  "reportId": "your-correlation-id",
  "resume": { "candidate": { "...": "your ReportResume shape" } },
  "job": { "title": "...", "description": "...", "skills": [ { "skill_name": "...", "skill_type": "must_have" } ] },
  "force": false
}
```

- `reportId` correlates the webhook to this request and is required.
- `resume` and `job` are required the first time a `reportId` is sent. The
  job's `skills[]` (with `skill_type` of `must_have` or `nice_to_have`) is
  the list of skills the report scores against.
- `force: true` builds a fresh report even if one already exists. Re-sending a
  `reportId` whose report failed re-queues it; one already `sent` is sent
  again without recomputing; otherwise the current status is returned.

Response `200`:

```json
{ "status": { "success": true }, "reportId": "your-correlation-id", "jobStatus": "pending" }
```

`jobStatus` is `pending`, `processing`, `processed`, `sending`, `sent` or
`failed`. `400` is returned for invalid JSON or a missing `reportId`, `job` or
`resume`.

## 2. Receive the report — your webhook

Hiresense POSTs to the webhook URL you gave us, with `X-Internal-Token` set
when a shared secret is configured:

```json
{ "reportId": "your-correlation-id", "report": { "...": "see below" } }
```

- Any `2xx` marks the report delivered. Anything else marks the delivery
  failed and Hiresense does not retry on its own; re-send the request for the
  same `reportId` to have it delivered again.
- Hiresense sends no version tag. You identify the shape by validating it,
  newest version first (your `validate_report` already does this).
- Hiresense removes every field outside the contract before sending, so an
  unknown field never reaches you.

## 3. Report versions

| Shape | `report_metadata.schema_version` | Mobius version | Sent |
| --- | --- | --- | --- |
| v2 shape | `3.1` | validates as your `v1` today (as `v2` when `skill_development_required` is empty) | now |
| v3 | `4.0` | new `v3`, to be added | after you confirm |

The v2 shape is what you receive today. Its scores follow the current rules
(skills, experience, education and responsibilities weighted 40, 30, 15, 15).
Reports in v3 are scored by the new rules (see "What the numbers mean"), so a
candidate's score and fit band can differ between a v2-shape and a v3 report.
Stored reports keep the version they were produced with.

## 4. Report v3

`report` is one JSON object with `report_metadata` and six sections, always
all present: `hiring_recommendation`, `qualification_match`,
`experience_background`, `candidate_strengths`, `risk_factors_gaps`,
`decision_framework`. Enum values are plain strings, never emoji.

### What changes from the v2 shape

- `report_metadata.schema_version` is `"4.0"`.
- `qualification_match.categories[]` has a new `level` field, and there are
  always four categories in this order: Skills match, Responsibility
  alignment, Experience match, Education match.
- `must_have_skills` and `nice_to_have_skills`:
  - `present_listed` entries become objects `{ skill, source }` (they were
    plain skill names). **This is the breaking change** that needs a new
    version.
  - `percentage` is now the credit percentage described below, not
    `matched / total`.
- `must_have_skills` only: new `related` (a count) and `present_related` (a
  list of `{ skill, related_skill, evidence }`), and `total` is
  `matched + related + missing`. `nice_to_have_skills` has neither field, and
  its `total` is `matched + missing`: a nice-to-have skill that is not matched
  is missing.
- `experience_background` has a new `experience_level` (`level`, `reason`),
  and each `domain_match.candidate_background[]` entry has a new `relevant`
  flag.
- `responsibility_comparison[].match_level` now really uses `SIMILAR` (the
  value was already allowed, but was not produced before).

Everything else (`hiring_recommendation`, `candidate_strengths`,
`risk_factors_gaps`, `decision_framework`, `education`) is unchanged.

### Example

```json
{
  "report_metadata": {
    "job_title": "Commissioning Specialist",
    "candidate_name": "Candidate",
    "overall_fit_score": 54.8,
    "fit_category": "MEDIUM_FIT",
    "analysis_timestamp": "2026-11-25T12:00:00Z",
    "schema_version": "4.0"
  },
  "hiring_recommendation": {
    "badge": "CONSIDER_WITH_CONDITIONS",
    "badge_label": "Consider with conditions",
    "badge_color": "ORANGE",
    "verdict": "Partial match",
    "overall_fit_score": 54.8,
    "time_to_productivity": "3-6 months",
    "readiness_level": "SIGNIFICANT_GAP",
    "summary": "..."
  },
  "qualification_match": {
    "categories": [
      { "category": "Skills match", "score": 41.9, "level": null, "weight": 0.4, "status": "GAP",
        "detail": "Must-have 4 of 8, 2 listed only, 2 related · Nice-to-have 1 of 4" },
      { "category": "Responsibility alignment", "score": 60.0, "level": null, "weight": 0.3, "status": "ACCEPTABLE",
        "detail": "4 of 5 core responsibilities evidenced, 2 similar" },
      { "category": "Experience match", "score": 50, "level": "ADJACENT", "weight": 0.2, "status": "ACCEPTABLE",
        "detail": "6 relevant years in data centre commissioning of 8 total · Senior, exact" },
      { "category": "Education match", "score": 100, "level": "MATCH", "weight": 0.1, "status": "STRONG",
        "detail": "BEng in Electrical Engineering, exact field · no required certifications" }
    ],
    "must_have_skills": {
      "matched": 4, "total": 8, "percentage": 43.8,
      "present_applied": [ { "skill": "Commissioning", "evidence": "Acme Ltd: led Level 4 and 5 commissioning" } ],
      "present_listed": [ { "skill": "BMS", "source": "Technical skills section" } ],
      "related": 2,
      "present_related": [ { "skill": "System verification", "related_skill": "Commissioning and testing", "evidence": "Acme Ltd" } ],
      "missing": [ "Record keeping", "Digital project tools" ]
    },
    "nice_to_have_skills": { "matched": 1, "total": 4, "percentage": 25.0,
      "present_applied": [ { "skill": "English", "evidence": "Acme Ltd: reports and meetings in English" } ],
      "present_listed": [], "missing": [ "..." ] },
    "education": { "...": "unchanged" }
  },
  "experience_background": {
    "experience_level": { "level": "ADJACENT", "reason": "Adjacent: 6 relevant years in data centre commissioning against 7 required, at the required Senior level." },
    "profile": { "total_years": 8, "relevant_years": 6, "relevant_domain": "Data centre commissioning",
      "role_level_candidate": "Senior", "role_level_required": "Senior",
      "role_level_assessment": "EXACT", "role_level_implication": "..." },
    "domain_match": {
      "requirement": "Hyperscale data centre construction",
      "candidate_background": [ { "domain": "Data centre commissioning", "years": 6, "relevant": true },
                                { "domain": "Retail management", "years": 2, "relevant": false } ],
      "carries_over": [ "..." ], "does_not_carry_over": [ "..." ],
      "transferability": "HIGH", "transferability_assessment": "..."
    },
    "role_readiness": "...",
    "responsibility_comparison": [ { "responsibility": "...", "candidate_experience": "...", "match_level": "SIMILAR" } ]
  },
  "candidate_strengths": [ { "...": "unchanged" } ],
  "risk_factors_gaps": { "...": "unchanged" },
  "decision_framework": { "...": "unchanged" }
}
```

### Field reference for the new and changed fields

| Field | Type | Values / meaning |
| --- | --- | --- |
| `qualification_match.categories[].level` | string or null | `MATCH`, `ADJACENT`, `NO_MATCH` for Experience match and Education match; `null` for Skills match and Responsibility alignment. Show the level in words where it is set, the percentage where it is not |
| `...present_listed[].source` | string | Resume section where the skill appears, e.g. "Technical skills section" |
| `must_have_skills.related` | integer | Number of skills in `present_related`. Not present on `nice_to_have_skills` |
| `must_have_skills.present_related[]` | object | `skill` (the job skill), `related_skill` (what the candidate applied instead), `evidence` (the role) |
| `experience_background.experience_level.level` | string | `MATCH`, `ADJACENT`, `NO_MATCH`; equals the Experience match category's `level` |
| `experience_background.experience_level.reason` | string | One sentence that starts with "Match:", "Adjacent:" or "No match:" |
| `...candidate_background[].relevant` | boolean | Whether this domain counted toward `profile.relevant_years` |
| `...responsibility_comparison[].match_level` | string | `DIRECT`, `SIMILAR`, `NONE` |

## 5. What the numbers mean

Hiresense computes every number. Display them; do not recompute them.

- **Overall fit** = skills 40% + experience 20% + education 10% +
  responsibilities 30%. Fit bands: `HIGH_FIT` 80 and up, `THRESHOLD_FIT`
  70 to 79.9, `MEDIUM_FIT` 50 to 69.9, `LOW_FIT` under 50.
- **Skills**: a skill present with applied evidence counts 1, present only in a
  skills list counts 0.5, missing counts 0. For must-have skills only, a
  related skill (not shown on the resume, but the same job was done with a
  different skill) counts 0.25. A task or method skill (for example
  "Functional testing") needs applied evidence in a role; a mention in a
  skills list does not count. A language skill counts only when the resume
  states the language, with one exception: a resume written in English is
  proof of English, so the English skill is listed (0.5) with the source
  "Resume written in English". A bucket's
  `percentage` is that credit divided by `total`. The skills score is 90%
  must-have and 10% nice-to-have; when the job has only one of the two, the
  score is that bucket's percentage.
- **Experience** is a level worth 100 (`MATCH`), 50 (`ADJACENT`) or 0
  (`NO_MATCH`). Only years in a relevant domain count. A different domain, or
  relevant years under half the requirement, or two levels below is
  `NO_MATCH`. Each candidate domain is `EXACT`, `SIMILAR` (an accepted domain,
  or at least half of the role's core systems), `TRANSFERABLE` (at least one
  core system, fewer than half) or `DIFFERENT` (a different type of work, or
  the same type of work that covers none of the core systems). When roles have
  no dates the years are estimated, and the "under half" limit becomes a
  quarter of the requirement. Otherwise count shortfalls (a transferable domain, relevant
  years under the requirement, one level below): none is `MATCH`, one is
  `ADJACENT`, more is `NO_MATCH`.
- **Education** is a level worth 100, 50 or 0: nothing required or degree and
  field met with all required certifications is `MATCH`; an unrelated field or
  a degree two or more steps below is `NO_MATCH`; anything else is
  `ADJACENT`.
- **Responsibilities**: each is `DIRECT` (1), `SIMILAR` (0.5) or `NONE` (0),
  from how many of its parts the resume covers.
- **Status pills** (`status`): Skills and Responsibilities from the score
  (75 and up `STRONG`, 50 to 74.9 `ACCEPTABLE`, under 50 `GAP`); Experience and
  Education from the level (`MATCH` is `STRONG`, `ADJACENT` is `ACCEPTABLE`,
  `NO_MATCH` is `GAP`).

## 6. What Mobius needs to do

1. Add `mobius_schemas/recruiter_report/v3.py`: copy `v2.py` and change only
   the classes below. Never edit `v1.py` or `v2.py`.
2. Register `"v3"` in `jobs/report_schema.py` and put it first in
   `_DETECT_ORDER` (`["v3", "v2", "v1"]`), then make it
   `CURRENT_REPORT_VERSION` when you are ready to receive it.
3. Render the new fields (levels in words, the related list on must-have skills, the listed
   source, the experience level reason, the relevant flag on domains).
4. Tell Hiresense when v3 is deployed on staging, then on production.

Changed classes for `v3.py` (everything else is identical to `v2.py`):

```python
class QualCategory(MobiusBaseModel):
    category: str          = Field(default="",   description="Scored category, e.g. 'Skills match'")
    score:    float | None = Field(default=None, description="Category score, 0-100")
    level:    str | None   = Field(default=None, description="MATCH | ADJACENT | NO_MATCH for Experience and Education; null otherwise")
    weight:   float | None = Field(default=None, description="Category weight in the overall score, 0-1")
    status:   str          = Field(default="",   description="GAP | ACCEPTABLE | STRONG")
    detail:   str          = Field(default="",   description="Explanation of the category score")


class SkillListed(MobiusBaseModel):
    skill:  str = Field(default="", description="Skill name")
    source: str = Field(default="", description="Resume section where the skill is listed")


class SkillRelated(MobiusBaseModel):
    skill:         str = Field(default="", description="Job skill")
    related_skill: str = Field(default="", description="The different skill the candidate applied")
    evidence:      str = Field(default="", description="Role where they applied it")


class SkillsBlock(MobiusBaseModel):
    matched:         int                  = Field(default=0,    description="Skills present (applied + listed)")
    total:           int                  = Field(default=0,    description="Must-have: matched + related + missing. Nice-to-have: matched + missing")
    percentage:      float | None         = Field(default=None, description="Credit percentage: applied 1, listed 0.5, related 0.25 (must-have only)")
    present_applied: list[SkillEvidence]  = Field(default_factory=list)
    present_listed:  list[SkillListed]    = Field(default_factory=list)   # v3: objects, were names
    related:         int                  = Field(default=0)               # must-have only; never sent for nice-to-have
    present_related: list[SkillRelated]   = Field(default_factory=list)   # must-have only; never sent for nice-to-have
    missing:         list[str]            = Field(default_factory=list)


class ExperienceLevel(MobiusBaseModel):
    level:  str = Field(default="", description="MATCH | ADJACENT | NO_MATCH")
    reason: str = Field(default="", description="One sentence: domain match, relevant years against required, role level")


class DomainBackground(MobiusBaseModel):
    domain:   str          = Field(default="",   description="A domain the candidate has worked in")
    years:    float | None = Field(default=None, description="Years in that domain")
    relevant: bool         = Field(default=False, description="Counted toward relevant years")


class ExperienceBackground(MobiusBaseModel):
    experience_level:          ExperienceLevel                = Field(default_factory=ExperienceLevel)
    profile:                   ExperienceProfile              = Field(default_factory=ExperienceProfile)
    domain_match:              DomainMatch                    = Field(default_factory=DomainMatch)
    role_readiness:            str                            = Field(default="")
    responsibility_comparison: list[ResponsibilityComparison] = Field(default_factory=list)
```

`DomainMatch.candidate_background` uses the new `DomainBackground`.
`ReportMetadata.schema_version` is `"4.0"` (a free-text field, no change).

## 7. Rollout

- **Staging**: Hiresense runs the new report generation in staging first and
  keeps sending the v2 shape to your staging webhook, so nothing breaks.
  When you confirm v3 on your staging, Hiresense switches staging to v3.
- **Production**: after you confirm v3 on production, Hiresense switches it.
  Rolling back is one setting on our side.

## Open points

- **`risk_factors_gaps.hiring_risk_assessment.skill_development_required`**:
  Hiresense sends a list of skill-area strings (your v1 form). Your v2
  expects structured objects. For v3 either type it as `list[str]`, or tell us
  and Hiresense will send the structured objects
  (`skill_area`, `priority`, `training_method`, `cost_estimate`,
  `estimated_time_weeks`), which it already has.
