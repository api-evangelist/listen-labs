---
name: listen-labs-harvest-study-results
description: >-
  Pull Listen Labs study results out over HTTP — list studies, fetch question definitions, page
  through responses with their extracted answers and summaries, and drill into a single
  participant's full transcript with audio and video. Use when syncing Listen Labs research data
  into a warehouse, notebook, or report.
api: openapi/listen-labs-study-data-openapi.json
base_url: https://listenlabs.ai
operations:
  - listStudies
  - getStudyQuestions
  - listResponses
  - getSingleResponse
---

# Harvest Listen Labs study results

## Authentication

Send the API key in the `x-api-key` header on every request. The key is scoped to one
organization, so every endpoint here is implicitly filtered to that organization — a study
that exists but belongs elsewhere returns `404 study_not_found`, not `403`.

## Use the versioned response endpoints

Read responses from `/api/public/v1/responses/...` (`listResponses`, `getSingleResponse`).
The unversioned `/api/public/responses/...` endpoints are **deprecated starting
2026-08-01**. They return the same data but in `snake_case`; the versioned endpoints return
`camelCase`. `listStudies` and `getStudyQuestions` are unchanged and keep `snake_case` —
expect mixed casing across a single sync and normalize on your side.

## Step 1 — find the study (`listStudies`)

`GET /api/public/list_surveys` lists every study in the organization. Each carries:

- `id` — the stable uuid. Key your permanent records on this.
- `link_id` — the URL slug. **This is the path parameter** for both the responses and the
  questions endpoints, and it is editable in study settings, so it can change. Never use it
  as a durable primary key.

## Step 2 — get the question definitions (`getStudyQuestions`)

`GET /api/public/studies/{study_id}/questions` — `study_id` takes the **`link_id` value**,
not the uuid. Returns the study's questions from its latest revision, each with an `id`
(the join key), `text`, `type` (`open_ended`, `multiple_choice`, `ranking`, `statement`),
`question_number`, `is_screener`, and — for concept-test blocks — a `concepts[]` array.

Fetch these first: they are what makes the answers interpretable.

## Step 3 — page through responses (`listResponses`)

`GET /api/public/v1/responses/{linkId}` with:

- `page` — zero-based page index (default `0`).
- `per_page` — default `1000`.
- `updated_since` — an ISO 8601 timestamp. **This is the incremental-sync mechanism.**
  Persist the high-water mark from the previous run and pass it here rather than re-reading
  the whole study.
- `include_in_progress` — defaults to `true`. Set it to `false` when you only want finished
  interviews, or filter on `progress` (`complete`, `screened_out`, `in_progress`).

Each response carries `answersArray[]` (structured, joinable), `answers` (keyed by question
text, plus a `"Summary"` key), `qualityScore`, `tags`, `tagline`, `bulletSummary`,
`shortTranscript`, and `urlParams` (any parameters you appended to the recruit link).

Join `answersArray[].discussionGuideQuestionId` to `Question.id` from step 2.

## Step 4 — drill into one transcript (`getSingleResponse`)

`GET /api/public/v1/responses/{linkId}/{responseId}` — `responseId` accepts either the
response UUID or its `readableId`. Returns the `transcript` array: one row per
moderator/participant exchange with `moderator`, `user`, `responseIndex`, `isFollowUp`, and
the join keys `discussionGuideQuestionId` and `answerId`.

Group rows by `answerId` to reassemble one answer including its follow-ups (`isFollowUp:
true`). Rows with a null `answerId` are non-question rows such as intro messages.

**Media expires.** `audio` is a signed URL valid for roughly one hour; `video` is an object
with an HLS `streamUrl` and a direct `mp4Url`. Download the bytes during the sync — do not
persist the URLs and expect them to resolve later.

## Errors

Versioned endpoints return `{error, code, issues?}`. Branch on `code`, never on the `error`
message. Relevant here: `400 bad_request` (a bad query or path parameter), `401
missing_api_key` / `invalid_api_key`, `403 forbidden`, `404 study_not_found`, `404
response_not_found`, `500 internal_error`. Full catalog in
`errors/listen-labs-problem-types.yml`; the entity graph and every join key is in
`data-model/listen-labs-data-model.yml`.

No rate limits are documented — throttle conservatively and back off on `500`.
