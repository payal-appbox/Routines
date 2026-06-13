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

**Action:** Apply the `claude-delete` Gmail label (see Step 3b). Do NOT delete it.

### Category C — Important / Other
Everything that doesn't fit Category A or B.

**Action:** Leave it in the inbox untouched. Add it to the summary as "flagged but not actioned."

## Step 3b: Label as Junk

For each email classified as Category B (junk), apply the `claude-delete` Gmail label.
The label is the **single source of truth** — once labeled, the owner does a one-click
bulk-trash in Gmail (open the `claude-delete` label → select all → trash). No JSON queue
or repo commit is needed — Gmail's label view IS the audit log.

Use Gmail MCP `label_thread` with this label ID:
- `Label_5400706671634396889` (displayName: `claude-delete`)

If you need to discover the ID, call `list_labels` and find the one whose displayName is
`claude-delete`. If the label is missing entirely, create it via `create_label` with
displayName `claude-delete`.

`label_thread` is idempotent — re-labeling a thread is a safe no-op, so no dedup logic
is required. If it returns "Requested entity was not found", the thread is already gone;
skip silently.

Track count of newly-labeled threads in this run for the Slack summary (Step 4).

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
Labeled claude-delete: [count]
Jira drafts ready: [count]
  - [TICKET-ID]: [one-line subject summary]
  - [TICKET-ID]: [one-line subject summary] ⚠️ possible misroute → suggested [PROJECT]
Important but not actioned: [count]
  - From: [sender] — Subject: [subject]
```

After processing, the owner can open Gmail's `claude-delete` label view and bulk-trash
whenever they want — no queue to manage.

If there were zero emails to process, still send a summary confirming that.

## Error Handling

- If Atlassian MCP fails to fetch a ticket, skip drafting for that email. Include it in the
  Slack summary as: "Could not fetch [TICKET-ID] — Atlassian MCP error. Needs manual review."
- If Gmail draft creation fails, note it in the summary.
- Never let one email's failure stop processing of the remaining emails.
