---
name: inbox-processor
description: Process Gmail inbox — classify emails, draft Jira ticket responses, label junk, and send Slack summary
user_invocable: true
---

# Gmail Inbox Processor

You are processing Payal Malhotra's Gmail inbox. Follow every step below precisely.
Refer to CLAUDE.md for safety rules — especially: NEVER send emails, NEVER delete emails,
NEVER modify Jira tickets.

## Step 1: Fetch Unread Emails

Use the Gmail MCP connector to search for unread emails in the inbox:
- Search query: `is:unread in:inbox`
- Read each email's subject, sender, body snippet, and thread ID

## Step 2: Classify Each Email

For every unread email, classify it into exactly one category:

### Category A — Jira Notification
An email is a Jira notification if ANY of these are true:
- Sender address contains `jira@` or `atlassian.net`
- Subject contains a Jira ticket ID pattern (e.g., `ABC-123`, `SD-456`, `PROD-78`)
- Body contains `(JIRA)` or links to a Jira instance

**Action:** Extract the ticket ID and process it (see Step 3).

### Category B — Junk / Marketing / Newsletter
An email is junk if ANY of these are true:
- Sender is a known marketing/newsletter address (unsubscribe link present, promotional content)
- Subject contains words like: "unsubscribe", "newsletter", "promotional", "sale", "offer", "deal"
- The email is an automated notification that requires no action (build notifications, automated reports, etc.)

**Action:** Add this email to the delete queue (see Step 3b). Do NOT delete it.

### Category C — Important / Other
Everything that doesn't fit Category A or B.

**Action:** Leave it in the inbox untouched. Add it to the summary as "flagged but not actioned."

## Step 3b: Queue & Label as Junk

For each email classified as Category B (junk), do TWO things:

### 3b.1 — Apply the `claude-delete` Gmail label

The label `claude-delete` exists in the account (id `Label_5400706671634396889`). Use the
Gmail MCP `label_thread` with this label ID on the thread. This is the primary trash
mechanism — once labeled, the owner does a one-click bulk-trash in Gmail (open the
`claude-delete` label → select all → trash).

If `list_labels` is called to discover IDs, use the label whose displayName is
`claude-delete`. If the label is missing, create it via `create_label` with displayName
`claude-delete` and use the returned ID.

If `label_thread` returns "Requested entity was not found", the thread is already gone —
skip it; do NOT append to the queue.

### 3b.2 — Append to `delete-queue.json` (with dedup)

Then append an entry to `delete-queue.json` in the repo root. Read the existing file first:

1. Build a `Set` of all existing `thread_id` values in `pending_delete`.
2. For each new junk thread, **skip if its thread_id is already in the set** — no
   duplicates across runs.
3. For each new (non-duplicate) thread, append:
   - `thread_id` — Gmail thread ID
   - `date` — email date (YYYY-MM-DD)
   - `sender` — sender email
   - `subject` — email subject
   - `category` — one of: "Marketing/Promo", "Newsletter/Event", "Onboarding/Newsletter", "Automated Notification", "Survey/No-action"
   - `reason` — brief explanation
   - `labeled_claude_delete: true` — confirms the Gmail label was applied

Update `last_updated` at the top of the file. Do NOT remove existing entries — the file
is an audit log; the `claude-delete` label is the operational state.

After appending, commit and push the updated file to the repository.

## Step 3c: Process Jira Notification Emails

For each Jira notification email (Category A):

1. **Extract the ticket ID** from the subject line or email body (e.g., `SDCHK-1234`,
   `SDAQ-567`). The project key prefix identifies the product — see the product mapping
   in the `/ticket-response` skill.

2. **Check if a response is actually needed** — fetch the ticket status and last comment:
   - If status is "Waiting for customer" AND the last comment is from your team (Payal or
     Nisha), no draft is needed. Skip to the next email.
   - Only draft when the customer has replied and the ticket is waiting on support.

3. **Invoke the `/ticket-response` skill** with the ticket ID. The skill will:
   - Fetch full ticket context (issue, comments, reporter, status)
   - Identify the product and check routing
   - Search BOTH the SD and Dev projects for **role-model tickets** (label `role-model`)
   - Search for **similar resolved tickets** and extract resolution patterns
   - Pull relevant docs (Confluence + docs.appbox.ai)
   - Apply product-specific knowledge (Checklist item-count limits, "not copied over"
     framing for migrations, Calendly link pattern, etc.)
   - Return the drafted reply body

   When called from this skill, ticket-response returns ONLY the reply body — no research
   summary wrapper.

4. **Save as Gmail draft** using the Gmail MCP — create a draft REPLY on the same thread
   ID as the incoming email. Do NOT send. Do NOT post anything to Jira (read-only — see
   CLAUDE.md).

5. **If ticket-response flags a misroute** at the end of its output, include that flag
   verbatim in the Slack summary (Step 4) so the human knows to consider moving the ticket.

## Step 4: Send Slack Summary

After processing ALL emails, send a Slack DM to Payal with this format:

```
Inbox Processing Complete

Emails processed: [total count]
Added to delete queue: [count]
Jira drafts ready: [count]
  - [TICKET-ID]: [one-line subject summary]
  - [TICKET-ID]: [one-line subject summary] ⚠️ possible misroute → suggested [PROJECT]
Important but not actioned: [count]
  - From: [sender] — Subject: [subject]

Delete queue: [total pending count] threads awaiting batch delete
```

If there were zero emails to process, still send a summary confirming that.

## Error Handling

- If Atlassian MCP fails to fetch a ticket, skip drafting for that email. Include it in the
  Slack summary as: "Could not fetch [TICKET-ID] — Atlassian MCP error. Needs manual review."
- If Gmail draft creation fails, note it in the summary.
- Never let one email's failure stop processing of the remaining emails.
