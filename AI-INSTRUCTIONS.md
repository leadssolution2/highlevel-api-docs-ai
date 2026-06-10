# AI-INSTRUCTIONS — GoHighLevel API V2

**Read this file first.** It is the single source of truth for any AI agent generating code, designing integrations, or answering questions about the GoHighLevel (HighLevel / LeadConnector) API in this repository.

---

## 1. Hard rules (do not violate)

1. **API V2 only.** V1 is end-of-life. Do **not** generate, suggest, or "preserve compatibility with" V1 endpoints, V1 auth headers, or `services.leadconnectorhq.com/v1/*` paths. If you find V1 code, flag it for replacement.
2. **Base URL is `https://services.leadconnectorhq.com`.** Every V2 request goes here. The old `rest.gohighlevel.com` host is V1 and is dead.
3. **Every V2 request requires the `Version` header.** Use `Version: 2021-07-28` unless a specific endpoint's reference page in `docs/` says otherwise. Missing this header is the #1 cause of silent 401s.
4. **`Accept: application/json`** is required on every request.
5. **Never invent fields.** If a field is not in this repo's OpenAPI/markdown source, it does not exist. Webhook payloads in particular are minimal — the AI must look up the actual event schema, not infer it from the contact object shape.
6. **Stoplight links are dead.** Any `highlevel.stoplight.io` URL the user pastes should be treated as deprecated; use the markdown / OpenAPI in this repo instead.

---

## 2. Authentication — pick one, commit to it

There are two auth models. They are **not** interchangeable. Decide before writing any code.

### Private Integration Token (PIT) — single account / internal use
- Use when: building for one HighLevel sub-account (one Location), internal automation, single-tenant tool.
- How: User generates a PIT inside their HighLevel sub-account settings. Token is long-lived.
- Header: `Authorization: Bearer <PIT>`
- Scope: bound to the Location it was created in. Cannot cross Locations.
- **Default to this for single-client work.** It is dramatically simpler than OAuth and is the right call ~80% of the time.

### OAuth 2.0 — marketplace apps / multi-tenant
- Use when: many different HighLevel customers will install / connect your app to their own accounts.
- Flow: standard OAuth 2.0 authorization code grant. Tokens expire — you **must** implement refresh.
- Header: `Authorization: Bearer <access_token>`
- Token types: Location-level token (acts on one sub-account) vs Company-level token (Agency level, can mint Location tokens).
- Reference: `docs/oauth/` in this repo, plus https://marketplace.gohighlevel.com/docs/oauth/Overview and https://marketplace.gohighlevel.com/docs/oauth/Faqs/index.html

**Up-front question for the AI to ask the user if it isn't told:** "Is this for one HighLevel account (PIT) or a marketplace app many customers will install (OAuth)?" Do not start coding until this is answered.

---

## 3. Rate limits — bake these in from line one

Per app, per resource (a "resource" = one Location, or one Company for Agency-level calls):

| Limit | Value |
|---|---|
| Burst | **100 requests / 10 seconds** |
| Daily | **200,000 requests / day** |

**Response headers to read on every request:**
- `X-RateLimit-Limit-Daily`
- `X-RateLimit-Daily-Remaining`
- `X-RateLimit-Interval-Milliseconds`
- `X-RateLimit-Max`
- `X-RateLimit-Remaining`

**Rules for AI-generated code:**
- Never write a naive `for contact in contacts: requests.post(...)` loop. Add a token-bucket / sliding-window limiter sized to the burst limit.
- On `429`, back off exponentially and respect any `Retry-After` header.
- For bulk operations, prefer batch endpoints where they exist (see `docs/` for which resources support bulk).
- Log the remaining-quota headers when getting close to the daily cap so the human operator sees it before it hits zero.

---

## 4. Where to find what — repo map

| Path | What's in it |
|---|---|
| `docs/marketplace modules/` | Per-resource markdown reference (Contacts, Conversations, Calendars, Opportunities, Locations, Custom Fields, Workflows, Payments, Users, Forms/Surveys, etc.). **Start here for any endpoint question.** |
| `docs/oauth/` | OAuth flow, scopes, token refresh, FAQ |
| `docs/webhook events/` | Webhook event payloads. **Critical:** event payloads do **not** include the full contact object — only the fields documented here. |
| `docs/country list/` | Country/locale codes used by location & contact endpoints |
| `models/` | OpenAPI schemas (machine-readable). Prefer these for type generation. |
| `apps/` | Per-product API specs |
| `common/` | Shared schema fragments referenced by multiple endpoints |
| `toc.json` | Table of contents — useful for finding the right page programmatically |

**When asked about an endpoint:** read the markdown in `docs/marketplace modules/<resource>/` AND cross-check the OpenAPI in `models/` or `apps/`. The markdown is human-readable; the OpenAPI is authoritative for exact field names and types.

---

## 5. Resource quick-reference (what each major area does)

- **Contacts** — CRUD on people. Custom fields live here. Tag operations are sub-endpoints of a contact.
- **Conversations** — SMS, email, IG/FB DM, WhatsApp threads. One Conversation = one channel with one contact.
- **Calendars** — appointments, calendar groups, free-slot lookup, calendar resources.
- **Opportunities** — pipeline stages, deal records. Opportunity custom fields are **separate** from contact custom fields.
- **Locations** — sub-accounts (one HighLevel customer = one Location). Most endpoints scope to a `locationId`.
- **Custom Fields** — schema definitions. Read these before writing any contact/opportunity that sets custom field values; you need the field's `id`, not its display name.
- **Workflows** — trigger workflows externally, list workflows. Cannot edit workflow definitions via API.
- **Payments** — invoices, orders, transactions, subscriptions, products.
- **Users** — Agency users and sub-account users. Different endpoints.
- **Forms / Surveys** — submissions and definitions.
- **Webhooks** — register subscriptions, view events. See section 7.

---

## 6. Common gotchas (each one has burned somebody)

1. **`Version` header missing → 401.** Not "invalid version" — just 401. Easy to mistake for an auth problem.
2. **Custom field IDs, not names.** When writing a contact, `customFields` takes `[{id, field_value}]`, not `{fieldName: value}`.
3. **Contact custom fields ≠ Opportunity custom fields.** Different endpoints, different field IDs, different scopes. Do not pass one where the other is expected.
4. **Phone numbers must be E.164.** `+15555551234`. Anything else gets silently rejected or normalised in surprising ways.
5. **`locationId` is mandatory on most writes.** Even when the token is Location-scoped. Pass it explicitly.
6. **Pagination is cursor-based on most list endpoints**, not offset/limit. Read the response's `meta` block; do not assume `?page=2` works.
7. **Webhook payloads are minimal.** A `ContactCreate` event gives you an ID and a few key fields, not the full contact. If you need more, follow up with `GET /contacts/{id}`.
8. **OAuth Location tokens vs Company tokens** — calling a Location-scoped endpoint with a Company token (or vice versa) gives confusing 403s. Mint the right token type for the call.
9. **Tag operations are not idempotent in the way you'd expect.** Adding an existing tag is a no-op (good); removing a non-existent tag returns success (also good); but bulk tag writes don't dedupe — sending the same tag twice creates two attempts.
10. **`startDate` / `endDate` are Unix ms timestamps in calendar endpoints, ISO-8601 strings elsewhere.** Always check the schema; do not assume.

---

## 7. Webhooks — read before generating any handler

- **Subscribe** via the API (see `docs/marketplace modules/` webhook section) or via the marketplace app config.
- **Verify the signature.** HighLevel signs webhook requests; reject any request whose signature doesn't validate. The signing secret is the app's secret (OAuth) or a webhook-specific secret (PIT-scoped webhooks).
- **Payload shape:** event-specific. **Do not assume a contact-create webhook contains the full contact.** Look up the exact event in `docs/webhook events/` before parsing.
- **Idempotency:** events can be redelivered. Use the event ID (or a `(type, resourceId, timestamp)` tuple) as a dedupe key.
- **Respond fast.** 200 quickly, then process async. HighLevel will retry on timeouts and that compounds your rate-limit usage.

---

## 8. When generating code

- Use a real HTTP client with retry + backoff (e.g. `httpx` + `tenacity` in Python, `got` + `p-retry` in Node). Never raw `fetch`/`requests` in a loop.
- Centralise the auth header, base URL, and `Version` header in one client wrapper so they cannot be forgotten on individual calls.
- Type the responses against the OpenAPI in `models/`. Generated types beat hand-typed dicts.
- Log the `traceId` / `requestId` returned by the API on every error response — HighLevel support cannot help without it.
- Pin the API version (`2021-07-28`). Do not switch versions mid-project without the user's explicit approval.

---

## 9. When in doubt

- **Look at the markdown in `docs/`** — it is the rendered marketplace docs, frozen as files. Authoritative.
- **Look at the OpenAPI in `models/` and `apps/`** — authoritative for schemas.
- If neither has the answer, the live docs are at https://marketplace.gohighlevel.com/docs/ and the developer hub at https://developers.gohighlevel.com/. Treat the *current* live docs as more recent than this repo if there is a conflict — but only if the human operator confirms.
- For ambiguous behaviour, ask the human. Do not invent.

---

## 9b. Automation Workflows (the in-app builder, not the API)

Building **workflow automations** (triggers/actions in the GHL builder UI, the Workflow AI "Build" mode, Custom Webhook retry patterns)? Read **`docs/automations/WORKFLOW-BUILDER-PLAYBOOK.md`** in this repo first. Highlights: drive the Workflow AI one node per message; its "Complete these steps" checklist is the designed human handoff; Custom Webhook saves `status` even on 4xx/5xx (branch `status Is 200` → success, everything else → bounded retry with a Counter field + native Math Operation); manual enrolls SKIP Custom Webhook actions; never ship an unbounded Go-To loop.

---

## 10. External references (for the human, not for AI ingestion)

- Developer Hub: https://developers.gohighlevel.com/
- API V2 marketplace docs: https://marketplace.gohighlevel.com/docs/
- Source repo (this is a copy of): https://github.com/GoHighLevel/highlevel-api-docs
- OAuth overview: https://marketplace.gohighlevel.com/docs/oauth/Overview
- OAuth FAQ: https://marketplace.gohighlevel.com/docs/oauth/Faqs/index.html
- Dev Slack invite: https://developers.gohighlevel.com/join-dev-community
- Support FAQ: https://help.gohighlevel.com/support/solutions/articles/48001060529-highlevel-api

---

*This file exists so an AI agent can load one short document and know how to behave correctly against the HighLevel API V2 without parsing the entire docs tree. If you change rules, change them here too.*
