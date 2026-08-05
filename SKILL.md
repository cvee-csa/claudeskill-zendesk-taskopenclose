# Zendesk Personal Task Tracker

This skill manages personal task tickets in Zendesk for Catherine Vee at Cloud Security Alliance. Creating a ticket = starting a task. Solving a ticket = finishing it.

## Who this is for

- **User / Requester / Assignee**: Catherine Vee (`id: 38942574549655`)
- **Zendesk instance**: `cloudsecurityalliance.zendesk.com`

---

## Creating a task ticket

The user will typically say something like:
> "Make a new ticket: Title: [ISP] Fix the router / Estimated Minutes: 15"

### What to ask if not provided
- **Title** (required) — the task description, often includes a category prefix like `[ISP]`, `[Reporting]`, etc.
- **Estimated minutes** (required) — how long the task is expected to take; remember this for when solving

### Step 1: Create the ticket

Tool names may be prefixed differently depending on environment (e.g. `mcp__remote-devices__zendesk__*` instead of `mcp__zendesk__*`). If the plain name below isn't found, search for it before assuming MCP is unavailable.

```
mcp__zendesk__create_ticket(
    subject="<title>",
    description="<title>",
    type="task",
    tags=["personal_task"],
    priority="low",
    assignee_id=38942574549655,
    requester_id=38942574549655,
    custom_fields=[
        {"id": 8510083340695,  "value": "it"},       # Department = IT
        {"id": 29450755118359, "value": <est_mins>}, # Estimated minutes (integer)
        {"id": 42288679067287, "value": "csa"}        # Ticket Allocation = CSA
    ]
)
```

### Step 2: Post internal note (immediately after creation)
```
mcp__zendesk__create_ticket_comment(
    ticket_id=<new_ticket_id>,
    comment="Ticket has been received",
    public=False
)
```

### Step 3: Post public reply
```
mcp__zendesk__create_ticket_comment(
    ticket_id=<new_ticket_id>,
    comment="Ticket has been received for processing.",
    public=True
)
```

All three steps happen automatically every time a ticket is created — no need to ask the user.

---

## Solving a ticket

The user will say "solve" (or "close", "done", "mark as complete") after a ticket has been created.

All three custom fields are **required** by Zendesk when solving — omitting any one will cause an API error. Reuse the estimated minutes from when the ticket was created.

```
mcp__zendesk__update_ticket(
    ticket_id=<id>,
    status="solved",
    custom_fields=[
        {"id": 8510083340695,  "value": "it"},
        {"id": 29450755118359, "value": <est_mins>},
        {"id": 42288679067287, "value": "csa"}
    ]
)
```

After solving, confirm with the ticket ID and title so the user knows it's done.

---

## Custom field reference

| Field | ID | Type | Value |
|---|---|---|---|
| Department | `8510083340695` | FieldTagger | `"it"` |
| Estimated minutes spent resolving | `29450755118359` | FieldInteger | integer (per ticket) |
| Ticket Allocation | `42288679067287` | FieldTagger | `"csa"` |

These values never change — Department is always IT, Ticket Allocation is always CSA. Only estimated minutes varies per ticket.

---

## Handling auth failures

There are two distinct failure signatures — tell them apart before picking a fix, since they need different responses:

- **`Zendesk client is not authenticated` / "Configure either legacy API token auth or complete the OAuth flow first."** — no credentials are configured at all. Use Options A–C below.
- **`{"error": "invalid_token", "error_description": "The access token provided is expired, revoked, malformed or invalid for other reasons."}`** — credentials exist but the OAuth access token has expired or been revoked.

**As of the `zendesk-mcp-server` fix that wires refresh-token support into `_ensure_client()`, the server auto-refreshes an expired access token on every call — you should no longer see `invalid_token` just because time passed (the token TTL observed was as short as 30 minutes).** If you still see it, it means the auto-refresh itself failed — most likely the stored `refresh_token` is also expired or was revoked server-side (e.g. after a Zendesk admin revoked the app's grant), not just an ordinary TTL expiry. In that case, Option D's manual flow is still the way to get a fresh grant — go straight there rather than retrying, since retries won't fix a revoked refresh token.

**Option A — Quick retry**: Sometimes auth errors are transient. Retry the exact same call once. If it fails a second time in the same session with the *same* error, don't keep retrying — go straight to Option D (if it's `invalid_token`) or Option C (if it's the "not authenticated" error and the browser extension is available), rather than burning more calls on retries.

**Option B — Reconnect via settings**: Ask the user to go to Cowork Settings → Connections → Zendesk and re-enter their API token. Once they confirm ("done"), retry.

**Option D — OAuth token refresh** (only needed if auto-refresh itself has failed, e.g. the refresh token was revoked): This connector supports a self-service OAuth re-auth flow via two tools — `mcp__zendesk__begin_oauth_authorization` and `mcp__zendesk__complete_oauth_authorization` (may be prefixed, e.g. `mcp__remote-devices__zendesk__*` — search for them if the plain names aren't found).

1. Call `begin_oauth_authorization` (no args needed unless a non-default scope is required). It returns an `authorization_url` and a `state` value.
2. Give the user the `authorization_url` and ask them to open it in their own browser and approve access.
3. Zendesk will redirect to `https://localhost/callback?code=...&state=...`. This page will fail to load (nothing runs on localhost) — that's expected. Tell the user to copy the `code` and `state` query parameters out of the browser's address bar on that failed page, and send them back.
4. Call `complete_oauth_authorization(code=<code>, state=<state>)`. A successful response confirms the tokens were saved and reports an `expires_in` (observed to be as short as 1800 seconds / 30 minutes) — this also refreshes the stored refresh_token, so auto-refresh should resume working after this.
5. Retry the original ticket operation.

**Option C — Browser fallback** (if MCP keeps failing and it's the "not authenticated" error, not `invalid_token`): Use Claude in Chrome to create the ticket via the Zendesk web UI. Note: this requires the Claude browser extension to be connected — if `tabs_context_mcp` / `navigate` report the extension isn't connected, this option isn't available either; fall back to Option B or D instead.

1. Navigate to `https://cloudsecurityalliance.zendesk.com/agent`
2. Click **+ Add → Ticket**
3. Set **Requester** to Catherine Vee, and type the **Subject**.
4. **Set Form = "IT Requests"** (not the "Default Ticket Form" — that form is missing the Estimated Minutes and Ticket Allocation fields entirely; only "IT Requests" exposes them). Changing the Form resets some fields, so set it early, before the rest.
5. Set tags: `personal_task`
6. Set custom fields on the IT Requests form: Department = IT (usually defaults), Estimated Minutes = `<est_mins>`, Ticket Allocation = CSA. Scroll down — Ticket Allocation is below the fold and easy to miss. Before submitting, re-scroll and visually confirm Department, Estimated Minutes, and Ticket Allocation are all filled in; Zendesk only surfaces a missing-required-field error after you click submit, so it's cheaper to check first.
7. Click **take it** next to Assignee to self-assign — this also auto-sets the IT-Operations group correctly, which the MCP tools can't do directly (no `group_id` support).
8. The composer at the bottom defaults to **Internal Note** mode. A new ticket also requires body text or Zendesk blocks submission with "Please provide a ticket description" — switch the composer to **Public reply** (click the "Internal note" dropdown → Public reply) and type something (the title works fine) before submitting.
9. Submit as New. Then add the internal note ("Ticket has been received") and the public reply ("Ticket has been received for processing.") the same way — via the composer toggle at the bottom of the now-created ticket.
10. When solving via the browser, re-check the same three custom fields (Department, Estimated Minutes, Ticket Allocation) are still populated before clicking Submit as Solved — Zendesk will reject the status change otherwise.

Automation note: fields with search/autocomplete (Requester search box, Tags) can drop keystrokes if you type immediately after clicking into them. Add a brief pause (or click, wait, then type) to avoid empty submissions.

---

## Keeping track of context

When a user creates a ticket and then later says "solve" without specifying a ticket ID or estimated minutes, use the most recently created ticket and its estimated minutes from the same conversation. If ambiguous, ask: "Solve ticket #XXXXXX ([title])?"
