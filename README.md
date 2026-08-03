# claudeskill-zendesk-taskopenclose

A Claude skill for tracking personal tasks as Zendesk tickets. Creating a ticket starts a task; solving it marks the task done.

## What it does

- **Create a ticket** from a title and an estimated-minutes value (e.g. "Title: [ISP] Fix the router / Estimated Minutes: 15"). The skill sets the requester/assignee to the configured user, tags it `personal_task`, fills in the required custom fields (Department, Estimated Minutes, Ticket Allocation), and posts an internal note plus a public reply automatically.
- **Solve a ticket** by referencing it (or the most recently created one in conversation) and re-supplying the required custom fields so Zendesk accepts the status change.
- **Falls back to the Zendesk web UI** via browser automation if the Zendesk MCP connector is unauthenticated, including the org-specific quirk that the "Estimated Minutes" and "Ticket Allocation" fields only appear on the **IT Requests** ticket form, not the Default Ticket Form.

See [`SKILL.md`](./SKILL.md) for the full instructions Claude follows, including the custom field ID reference and the step-by-step browser fallback.

## Setup

- Requires a Zendesk MCP connector authenticated against `cloudsecurityalliance.zendesk.com`, or browser access to the same instance for the fallback path.
- The requester/assignee identity and custom field IDs are hardcoded in `SKILL.md` for this organization; update them if reusing this skill elsewhere.

## Usage

Install the skill, then ask Claude things like:

- "Make a new ticket: Title: [Reporting] Fix the dashboard export / Estimated Minutes: 20"
- "Solve that ticket"
- "Close ticket #12345"
