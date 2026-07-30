---
name: listen-labs-create-and-launch-study
description: >-
  Create a Listen Labs study from a JSON study guide, pick a funding wallet, and launch it to get a
  self-recruit link — using the Listen Labs Public API v2 directly over HTTP. Use this when driving
  Listen Labs from code or CI rather than through the hosted MCP server.
api: openapi/listen-labs-v2-openapi.yml
base_url: https://listenlabs.ai
operations:
  - createStudy
  - listWallets
  - launchStudy
---

# Create and launch a Listen Labs study

Launching spends real credits from an organization wallet and starts recruiting real
participants. Treat `launchStudy` as an irreversible spending action: always confirm with
a human before calling it.

## Authentication

Every request carries the API key in the `x-api-key` header. The key is scoped to a
single organization — the study is created in, and the wallets are listed for, that one
organization. Keys are created by Admins and Supervisors from the Developer section of
the account page.

```
x-api-key: <api_key>
```

There is no OAuth on the REST API. If you are operating as an agent inside an MCP client,
use the hosted MCP server instead (`mcp/listen-labs-mcp.yml`) — it authenticates with
OAuth 2.1 and acts as the signed-in user.

## Step 1 — create the draft (`createStudy`)

`POST /api/public/v1/studies/create` with a `title` and a `studyGuide`. The guide is
validated up front. On success you get back a draft study's `id` and `linkId`. Nothing is
visible to participants yet and no credits are spent — the draft can still be reviewed or
edited in the dashboard.

Handle these failures before moving on:

- `400 invalid_json` — the body is not valid JSON.
- `400 invalid_request_body` — schema validation failed. Read `issues[]`; each entry has a
  `path[]` and a `message` pointing at the offending field.
- `400 invalid_study_guide` — a cross-field rule broke: screening block placement,
  `minSelect`/`maxSelect` coupling, duplicate `externalId`s, an unresolved reference, or
  `exclusiveOption` placement. The message names the specific rule.
- `409 concurrent_modification` — retry the request.

## Step 2 — choose a wallet (`listWallets`)

`GET /api/public/v1/wallets` returns the wallets granted to the organization with their
recruitment and project credit balances.

You may omit `walletId` at launch **only** when the organization has exactly one wallet
(it is auto-selected). With more than one, omitting it returns `400 wallet_required`. Read
the balance here and fail fast rather than discovering `400 insufficient_credits` at
launch.

## Step 3 — launch (`launchStudy`)

**Gate this on explicit human approval.** `POST /api/public/v1/studies/{studyId}/launch`
publishes the draft and opens its self-recruit link. Use the `id` from step 1 as
`studyId`, and pass the `walletId` from step 2 unless the organization has exactly one
wallet.

The response includes `selfRecruitLink` — distribute it to participants or feed it into
your own recruitment flow. Project responses bill to the launch wallet.

Failures worth branching on:

- `400 insufficient_credits` — the wallet cannot fund the launch. Do not retry blindly.
- `403 launch_permission_denied` — the key's user can edit but not start recruitment.
- `409 study_busy` / `409 conflict` — an update was in flight or the publish raced an
  edit. Wait a moment, refresh the study state, and retry.

## Error handling rules

All `/api/public/v1/*` errors share one envelope: `error`, `code`, and — on 400 schema
failures only — `issues[]`. **Branch on `code`, never on `error`**: the docs state the
human-readable message may change. The full code list is in
`errors/listen-labs-problem-types.yml`.

There is no idempotency key on this API. Retries of `createStudy` after a network timeout
can create a duplicate draft, so re-read state before retrying rather than blindly
re-posting. Only the 409 codes above are documented as safe to retry as-is.

## Carrying your own identifiers

Append URL parameters to the self-recruit link (for example
`https://listenlabs.ai/s/abc123?id=123`). They come back per response as `urlParams`, which
is the supported way to join Listen Labs data to your own systems.
