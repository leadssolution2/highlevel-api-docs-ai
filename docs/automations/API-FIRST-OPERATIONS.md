# API-FIRST Operations — stop going to the browser for these

> Verified live 2026-07-03 (Hydra Clean, full-scope PIT). Companion to `WORKFLOW-BUILDER-PLAYBOOK.md`.
> PITs now carry **all scopes** — most reads AND many writes are API-doable. The browser is for the short UI-only list at the bottom, nothing else.

**Base:** `https://services.leadconnectorhq.com` · Headers: `Authorization: Bearer pit-…` + `Version: 2021-07-28` (+`Content-Type: application/json` on writes). PITs are **location-scoped** (403 cross-location).

## Verified API-doable

```bash
H=(-H "Authorization: Bearer $PIT" -H "Version: 2021-07-28" -H "Content-Type: application/json")

# Workflows — list + status (published/draft). METADATA ONLY, no node trees.
curl -s "${H[@]}" "https://services.leadconnectorhq.com/workflows/?locationId=$LOC"

# Enroll a contact in a workflow (RUNS it immediately) / remove
curl -s -X POST   "${H[@]}" -d '{}' "https://services.leadconnectorhq.com/contacts/$CID/workflow/$WF"
curl -s -X DELETE "${H[@]}"        "https://services.leadconnectorhq.com/contacts/$CID/workflow/$WF"

# Tags add/remove — FIRES Contact-Tag workflow triggers (verified)
curl -s -X POST   "${H[@]}" -d '{"tags":["my-tag"]}' "https://services.leadconnectorhq.com/contacts/$CID/tags"
curl -s -X DELETE "${H[@]}" -d '{"tags":["my-tag"]}' "https://services.leadconnectorhq.com/contacts/$CID/tags"

# Contact read/update (std fields, dnd, custom fields by id)
curl -s "${H[@]}" "https://services.leadconnectorhq.com/contacts/$CID"
curl -s -X PUT "${H[@]}" -d '{"customFields":[{"id":"<fieldId>","value":"..."}]}' ".../contacts/$CID"

# Custom fields CRUD — BOTH models (default is contact; pass model=opportunity explicitly)
curl -s "${H[@]}" ".../locations/$LOC/customFields?model=opportunity"
# create dropdown: {"name":"X","dataType":"SINGLE_OPTIONS","options":["A","B"],"model":"contact"}  ← options, NOT picklistOptions
# DELETE /locations/$LOC/customFields/$ID works (destructive — values clear on every record)

# Custom values
curl -s "${H[@]}" ".../locations/$LOC/customValues"          # + PUT /customValues/$ID

# Opportunities — search, stage-move, won/lost; pipelines w/ stage ids
curl -s "${H[@]}" ".../opportunities/search?location_id=$LOC&contact_id=$CID"
curl -s -X PUT "${H[@]}" -d '{"pipelineStageId":"...","status":"open"}' ".../opportunities/$OPP"
curl -s "${H[@]}" ".../opportunities/pipelines?locationId=$LOC"

# Appointments
curl -s "${H[@]}" ".../contacts/$CID/appointments"
# PUT /calendars/events/appointments/$ID needs ignoreDateRange:true + ignoreFreeSlotValidation:true + assignedUserId (else 400 "slot" / 422 "team member")

# Calendars — config incl. the attached booking formId
curl -s "${H[@]}" ".../calendars/$CAL_ID"

# Forms — list works; GET /forms/{id} = 401 "not supported by IAM" for ALL PITs (NOT a scope issue — do not retry).
# Read a form's fields via its SUBMISSIONS instead: fieldsOriSequance = every field id in order + sample values.
curl -s "${H[@]}" ".../forms/?locationId=$LOC&limit=50"
curl -s "${H[@]}" ".../forms/submissions?locationId=$LOC&formId=$FORM&limit=5"

# Conversations — find, inject inbound (see trigger caveat below)
curl -s "${H[@]}" ".../conversations/search?locationId=$LOC&contactId=$CID"
curl -s -X POST "${H[@]}" -d '{"type":"SMS","conversationId":"...","message":"..."}' ".../conversations/messages/inbound"

# Tasks on a contact (verify a workflow's Add-Task fired)
curl -s "${H[@]}" ".../contacts/$CID/tasks"
```

## ⚠️ Which API writes FIRE workflow triggers (the hours-saver)

| API write | Fires its trigger? |
|---|---|
| Tag added (`POST /contacts/{id}/tags`) | ✅ YES — Contact-Tag workflows fire |
| Add-to-workflow (`POST /contacts/{id}/workflow/{wf}`) | ✅ YES — workflow runs |
| Inbound message (`POST /conversations/messages/inbound`) | ❌ NO — "Customer Replied" does NOT fire (msg lands in convo only). Testing reply classifiers needs a REAL reply from a phone/inbox. |
| DND (`PUT /contacts/{id}` `{"dnd":true}`) | ❌ NO — "Contact DND" does NOT fire. Needs a real STOP reply or UI toggle. |
| Manual-enroll / Test workflow | ⚠️ Runs, but **skips Custom Webhook actions** ("Skipped" in exec log) |

→ Consequence: **tag-triggered campaigns can be driven AND tested end-to-end via API** (tag → enroll → force-step in UI → verify sends/tasks/opp-stage via API). Reply/DND branches are the only parts needing a human.

## ⚠️ Merge-field fallbacks are HANDLEBARS — `||` is a PARSE ERROR (verified live 2026-07-03)

GHL resolves merge fields with **Handlebars**. `{{contact.x || "default"}}` throws `Template Parser error … Expecting 'CLOSE_RAW_BLOCK'…` — the `||` OR-fallback does **not** exist in Handlebars. (Prior docs claiming `{{contact.field || "N/A"}}` works were never tested — it doesn't.)

**Correct fallback = if/else block:**
```
{{#if contact.service_requested}}{{contact.service_requested}}{{else}}cleaning{{/if}}
```
Verified end-to-end: populated contact rendered "…your last Carpet Cleaning…", blank contact rendered "…your last cleaning…".

**Free live template validator:** `POST /conversations/messages {type:"Email", contactId, subject, html:"<p>…{{merge}}…</p>"}` — GHL parses the Handlebars on send. Bad syntax → HTTP 400 with the exact parse error + caret; valid → sends, and you can read the *rendered* result in the recipient inbox. Use it to test any merge expression before committing it to a workflow/custom value.

**Field-or-field** (`{{a || b}}`) is the same parse error → use `{{#if a}}{{a}}{{else}}{{b}}{{/if}}`. Any GHL automation currently using `||` (e.g. opp-relay "prefer form field, else AI field") is silently broken — audit and convert.

## UI-only (go to the browser knowingly, once, for exactly these)

- Workflow **node trees / triggers / actions** — no public API (GET /workflows = metadata only)
- **Publish toggle** — Firestore/UI-only (no REST endpoint; page-token workaround does NOT cover it)
- **Enrollment History → "Force the Contact to the next step"** (person→ row icon) — fast-forwards a test contact past Wait nodes; sends fire for real; refresh the list between clicks
- Calendar **default-notification bodies** (cross-origin `calendar-app.leadconnectorhq.com` iframe)
- **Snippets** (double-submit rule), **email-template bodies** (IAM error), **Conversation-AI bot config**, **trigger links**

## Browser-connection protocol (when the UI trip is unavoidable)

- **deviceId ≠ machine; NOT stable across sessions** (observed flipping between PCs). Identify the machine by **viewport size fingerprint** (screenshot dimensions differ per PC) + its open work-tabs — or `switch_browser` (user clicks Connect on the machine they mean; the only certain method).
- `switch_browser` → "No other browsers available" = **the target machine is NOT connected**. Stop; have the user click the extension's Connect there. Do not proceed on whatever is connected.
- Tab groups drop between turns; after recreating one, **re-verify which machine you're on** (sessions silently rebind).
- Backgrounded tab ⇒ builder canvas paints blank. Heavy canvases wedge the renderer (30s screenshot timeouts) → act, wait 5–8s, retry capture once; reload the URL if clicks stop registering. "Create workflow" button can jam → open an existing Draft by URL instead. Single-click (never double) the "Add new trigger" node — double-click zooms the canvas.
