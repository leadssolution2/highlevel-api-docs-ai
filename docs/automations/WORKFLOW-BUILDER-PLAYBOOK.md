# GHL Workflow Builder Playbook — AI Builder, Custom Webhooks, Bounded Retry

Generic, client-agnostic field knowledge for building GoHighLevel **Automation Workflows**, verified empirically 2026-06-10 on live workflows. Applies to any sub-account.

---

## 1. The Workflow AI Builder ("Build" mode) — what it can and cannot do

The purple-sparkle Workflow AI panel has two modes: **Chat** (Q&A) and **Build** (edits the workflow). Build mode is genuinely useful **if driven correctly**.

### Hard-won rules

1. **Feed it ONE node per message** (2–3 max, and only for non-branching chains). Large multi-node prompts produce hollow scaffolds: actions created with empty pickers, wrong condition fields (it will grab a similarly-named contact field), and junk substitutions (an Add Notes step instead of a Go To). One-node-at-a-time produces correctly configured nodes.
2. **Put the node's settings and its connections in the same sentence/paragraph as the node.** Narrative paragraphs describing the whole flow perform far worse.
3. **Webhook-response conditions CAN be bound by the AI** — but only if you name the source explicitly. Phrasing that works:
   > "…whose condition checks the saved webhook response of "<Webhook Action Name>": the response field "status" Is 200."
   The AI then sets condition type `webhook_response`, subtype `status_code`, operator `==`, value `200` — fully configured. Without that phrasing it picks a contact field and leaves the value empty.
4. **Contact-custom-field pickers are NOT reliably set by the AI** (Update Contact Field values, Math Operation field selection, contact-field If/Else conditions). It creates the action and tells you to finish it.
5. **The "Complete these steps before executing your workflow" checklist is the designed handoff, not an error.** Each item has a pencil icon that opens the exact action panel. Workflow = AI scaffolds → operator clicks each pencil and completes the config → next increment. Complete each node's pending items **before** sending the next increment.
6. **Check the TRIGGERS after every AI pass.** Build mode has been observed adding unrelated triggers (e.g. "Payment Received", "Invoice") that were never requested. Delete strays immediately.
7. **The AI's own success claims lie.** It will report "the correct custom field is selected" while the checklist says "Select field is required." Trust the checklist + the actual node panels, never the chat summary.
8. The AI cannot fill: workflow selectors (Remove From Workflow — though it usually defaults to "Current workflow", needing only an open+Save), field pickers, and exact field mappings. It CAN fill: action names, wait durations/units, operators, plain condition values, Go To targets, branch structure.

### Iterating / recovering

- "Delete every action you added; keep only X" works as a chat instruction for resetting a botched pass.
- A correction pass on a misconfigured multi-node build tends to make things WORSE (duplicate nodes). Prefer reset + smaller increments over in-place repair.

---

## 2. Custom Webhook action — response handling facts (verified)

- Enable **"Save response from this Webhook"**. This exposes `response` (body), `status` (HTTP code), and `headers` as branchable fields on downstream If/Else conditions, referenced as `<Action Name> > status` etc.
- **`status` is captured even on failures** — a 404 from the receiver stores `404` (builder shows "Request failed with status: 404" with headers+body). So `If status Is 200` is a reliable success gate.
- **A non-2xx webhook does NOT halt the contact** when an If/Else follows: the contact proceeds into the condition and lands in the `None` branch. (Without a following condition, the contact simply stops at the failed action and the event is lost — GHL Custom Webhook has **no native retry**.)
- **If the receiving server is completely down, there is NO response at all** — no status value. Therefore branch on success (`status Is 200` → done) and treat *everything else* (404, 500, empty/timeout) as the retry path. Never enumerate failure codes.
- Condition on `status` (the code), NOT `response` (the body). `response Is 200` compares the body string and will never match.
- The builder's **"Test again"** button on the webhook action fires a real request and shows status/headers/body — use it to verify the endpoint and see exactly what the condition fields will contain.
- **Manual enrollment / "Test workflow" / Workflow Status Page enrollment SKIPS Custom Webhook actions** (Event Status: "Skipped", Action Executed From: "Workflow Status Page"). You CANNOT validate webhook behavior with manual enrolls — only a real trigger event (or the Test-again button for the call itself).

## 3. Bounded retry pattern for Custom Webhook (copy this)

Goal: survive transient receiver outages without infinite loops. Requires the receiving endpoint to be **idempotent** (a re-send must no-op if the first attempt succeeded).

```
Trigger
  → Update Contact Field  "Set Counter to 0"      (number custom field Counter = 0)
  → Custom Webhook        "#1 Send To X"           (Save response ON)
  → If/Else               "If Response is 200"     (#1 Send To X > status  Is  200)
      ├─ 200  → Remove From Workflow → END                       (success)
      └─ None → If/Else   "If Counter > 3"         (Counter  Greater Than  3)
            ├─ >3   → Remove From Workflow → END                 (give up; optionally notify first)
            └─ None → Wait 1 minute
                     → Math Operation              (field Counter, Add, 1, Save Result To Field = Counter)
                     → Go To  →  "#1 Send To X"                  (the loop)
```

Notes:
- **The counter bound is mandatory.** A bare `None → Wait → Go To` loop retries FOREVER on persistent failure (observed live). GHL loop protection will not save you.
- **Math Operation is a native action** (search "math" in the action picker — searching "increment" finds nothing). Field / Operator / Value / **Save Result To Field** — the save-back is required or the math is discarded. AI chat claims that GHL "can't do field math" are wrong.
- Reset the counter to 0 at the START of each enrollment, before the webhook.
- A "Counter" number custom field must exist in the sub-account (create once, reuse in every workflow).
- Add an Internal Notification before the give-up Remove if a human should hear about exhausted retries.
- Editing/republishing changes nothing for already-failed past events — they were dropped; re-deliver them manually if the receiver matters.

## 4. Builder UI facts that bite

- **"Copy action" / "Copy all actions from here" pastes only within the SAME workflow.** There is no action-level copy between workflows. "Duplicate workflow" clones the whole thing (new workflow, no enrollment history).
- Workflows save as **Draft vs Published** — the Publish toggle, not Save, controls what runs live. A tested draft change does nothing until published.
- Manual value steppers: numeric inputs in action panels sometimes reject programmatic/pasted text; the +/− steppers always work.
- The list search box matches loosely — search short substrings ("Dr Chrono"), not full names with punctuation.

## 5. Browser-automation notes (only relevant when an AI drives the builder)

- The workflow canvas/panels live in a **cross-origin iframe**: DOM queries and accessibility trees can't reach it; only real coordinate clicks work.
- **Browser zoom MUST be 100% (devicePixelRatio = 1)** or every click lands offset. Check `window.devicePixelRatio` FIRST when clicks misbehave; fix with Ctrl+0.
- Keystrokes that miss an input land on the canvas as **hotkeys** (opens workflow switcher, shortcuts overlay, notes palette). Verify the caret is in the field, and verify text visibly arrived, before submitting anything.
- Navigate by in-app clicks; deep URL navigation blanks the SPA. When a page wedges: refresh (F5), wait 10–20s, refresh again if needed.
- Category drill-down dropdowns (field pickers) are flaky under synthetic clicks; the search box inside the picker, when present, is the reliable path.
- **No DOM/text shortcut for the canvas:** `get_page_text` sees only the top document and `read_page` (accessibility tree) returns nothing from the builder iframe — there is no text-extraction path for a large branching workflow; visual zoom-region reads of screenshots are the only way. Don't waste time trying them on the canvas.
- **The select-all trap (cause of "stuck panel" loops):** a page-wide `Ctrl+A` selects the ENTIRE page, and the lingering select-all state breaks the next click/scroll so the node panel won't switch or close. Read a node from the screenshot (no Ctrl+A); to grab one truncated field verbatim, `triple_click` that single field → `Ctrl+C` → read clipboard.
- **Read-only safety:** merely OPENING an email node sets the unsaved-changes flag → a discard-on-close dialog (never Save on a reference/pristine account). **SMS nodes don't dirty** — open them freely.

## 6. Reading GHL data without the browser — client-ops MCP (verified 2026-06-18)

When you have the `mcp__client-ops__ghl_<account>` proxy, read live data instantly — no token dance, no iframe. Pass `{ghl_tool, arguments}` (arguments = a JSON string). Highest-value calls:
- `opportunities_get-pipelines` → every pipeline + stages + IDs. (Answers "which pipeline do bookings land in," "why are there extra pipelines.")
- `opportunities_search-opportunity {"location_id":"<id>","limit":10}` → opps with `pipelineId`, `pipelineStageId`, `source`, and `customFields`. Empty `customFields:[]` on a booking-created opp = the **blank-card problem** (fix in §7).
- `locations_get-custom-fields {"model":"opportunity"}` → ⚠️ **the default returns model=contact**; you MUST pass `model:"opportunity"` to see opportunity fields + their `fieldKey`/`id`.
- Also `opportunities_update-opportunity`, `contacts_*`, `calendars_get-calendar-events`. Responses can be huge and spill to a tool-results .txt with lines too long for line-based reads — `grep` it or python-slice `read()[A:B]`.
- The proxy is READ + update for opps/contacts. It **cannot** create custom fields or edit workflow/pipeline internals — those stay UI-only.

## 7. Map contact fields onto the opportunity card — the "blank booking" fix (verified 2026-06-18)

A calendar booking auto-creates an opportunity, but it lands with **empty custom fields** — the card shows no service, no detail. Fix with a tiny published workflow:
- **Trigger:** `Opportunity created` → add filter **In pipeline = <your pipeline>** (scopes it; harmless even if it also catches synced opps that lack the source fields — they just get blanks).
- **Action:** `Update opportunity` → add a field row per opp custom field, value = `{{contact.<field>}}` (the value box takes literal merge syntax). **No Find Opportunity needed** — when an opportunity-type trigger is present, Update Opportunity updates the *triggering* opp (the action panel states this).
- Publish (toggle Draft→Publish, then Save). Going forward every new opp in that pipeline carries the captured detail on its card. Pre-existing blank cards aren't backfilled (their contacts usually lack the values anyway).

## 8. Calendar DEFAULT notifications live in a SECOND cross-origin iframe (verified 2026-06-18)

Settings → Calendars → edit calendar → Advanced settings → **Notifications & policies** is a SEPARATE GHL surface from workflows, and it renders inside `calendar-app.leadconnectorhq.com` — cross-origin from the white-label parent. So parent-window JS / `find` / `read_page` see nothing and token-capture can't reach it: **coordinate clicks only.**
- Notifications: Appointment booked (Unconfirmed / Confirmed), Cancellation, Reschedule, Reminder, Follow-Up. Each has **Email / In-app / SMS / WhatsApp** tabs and per-recipient bodies (Contact, Assigned user, Additional emails). Green channel badge = on.
- **Edit an email body SAFELY (without shattering merge tags):** open the body toolbar's **`</>` source-code** button → in the plain-textarea, **double-click a unique word in your anchor line** (e.g. `meeting_location`), confirm the highlight with a `zoom`, press **Right ×3** (collapse selection + step past the `}}`), type your HTML (`<br><strong>Label:</strong> {{contact.field}}`), then the source modal **Save changes**. Append-only → all existing links/custom-values (`{{reschedule_link}}` etc.) are preserved. NEVER click mid-tag in the WYSIWYG — it breaks the tag.
- Expand a recipient via its **chevron on the right** (clicking the label/row toggles the recipient checkbox instead). The **SMS tab opens Disabled** — toggle it on before the body is editable. Save the notification modal **and** the calendar's **top-right Save changes** (both persist).
- 150% browser zoom materially improves coordinate-click precision on this surface. This is distinct from the Extendly **B-007 reminder workflows** (which fire off Appointment-status triggers and use HTML snippets) — a client may use either or both.

## 9. GHL REST API (PIT) — the precise read/write boundary (verified 2026-06-18)

§6 covers the `client-ops` *proxy*. With a **Private Integration Token** (`Authorization: Bearer pit-…`, `services.leadconnectorhq.com`, `Version: 2021-07-28`) you call REST directly from a shell. Reads are reliable (workflows, custom values, pipelines, calendars, tags, location config). Writes are mixed — exact boundary:

| Target | API write? | Notes |
|---|---|---|
| **Custom Values** | ✅ YES | `PUT /locations/{id}/customValues/{cvId}` — read-back confirms. |
| **Custom Fields** | ✅ YES (create) | created intake fields via API. |
| **Contacts** | ✅ write, ❌ search | `POST /contacts/` works (tags inline → fires tag triggers); `GET …/search` → 401 "not authorized for this scope". |
| **Calendar `alertEmail`** | ✅ persists | — |
| **Calendar owner / slug** | ❌ NO | only `alertEmail` takes; GHL "requires the calendar to have an owner first" → set in UI. |
| **Email template BODY** | ❌ NO | API only lists/creates templates — content edits return *"this route is not yet supported by the IAM Service."* Edit body in the WYSIWYG builder. |
| **Business Profile** | ❌ NO | PIT/agency token `GET` only; `PUT` → 401. UI-only. |
| **Pipelines / workflow internals** | ❌ NO | UI-only. |

- **⚠️ PIT is LOCATION-SCOPED → 403 cross-location.** A PIT for one sub-account can't read another — reference accounts must be read via the browser UI.
- **⚠️ Image/file-type Custom Values silently no-op a plain-text PUT** — returns **200 but stays empty** on read-back (a `200 + empty read-back` = wrong value type, e.g. a logo CV is image-type).

### Reading the live COPY via API (when you can't open every node)
- **Email bodies — `previewUrl`:** `GET …/emails/builder` lists templates, each with a firebase-hosted **`previewUrl`** whose HTML is the verbatim rendered body (merge fields intact), curl-able with no browser. **Limits:** graphic-heavy templates were never rendered to firebase → **no `previewUrl`**; individual sub-templates aren't in the list (only a parent builder); the per-builder detail endpoint is out-of-scope for the PIT. For those, read the node panel.
- **SMS bodies — the templates endpoint returns EMPTY:** `GET …/locations/{id}/templates?type=sms` comes back empty, yet every SMS send-node references a **named HTML-snippet template**. The snippet body is only readable in the node's panel preview (canvas-only); editing the snippet once updates every node that references it.
