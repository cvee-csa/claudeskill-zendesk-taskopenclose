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

If `mcp__zendesk__*` returns `Zendesk client is not authenticated` or a similar auth error:

**Option A — Quick retry**: Sometimes auth errors are transient. Retry the exact same call once. If it fails a second time in the same session, don't keep retrying — the auth is very likely down for the whole session, not transient. Go straight to Option C (or B if the user is present and can reconnect quickly) rather than burning more calls on retries.

**Option B — Reconnect via settings**: Ask the user to go to Cowork Settings → Connections → Zendesk and re-enter their API token. Once they confirm ("done"), retry.

**Option C — Browser fallback** (if MCP keeps failing): Use Claude in Chrome to create the ticket via the Zendesk web UI.

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
