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

## Step 3b: Append to Delete Queue

For each email classified as Category B (junk), append an entry to `delete-queue.json` in the
repository root. Read the existing file first, then add new entries to the `pending_delete` array.

Each entry must include:
- `thread_id` — the Gmail thread ID
- `date` — the email date (YYYY-MM-DD)
- `sender` — sender email address
- `subject` — email subject line
- `category` — one of: "Marketing/Promo", "Newsletter/Event", "Onboarding/Newsletter", "Automated Notification", "Survey/No-action"
- `reason` — brief explanation of why it's junk (e.g., "Hosting provider upsell", "Duplicate event email to test account")

Update the `last_updated` timestamp at the top of the file. Do NOT remove existing entries —
the file accumulates across runs until the owner batch-deletes them locally.

After appending, commit and push the updated file to the repository.

## Step 3c: Process Jira Notification Emails

For each Jira notification email (Category A):

1. **Extract the ticket ID** from the subject line or email body (e.g., `SD-1234`)

2. **Check if a response is needed** — if the ticket is already in "Waiting for customer" status
   and the last comment is from your team (Payal or Nisha), no draft is needed. Only draft a
   response when the customer has replied and is waiting for support.

3. **Fetch ticket context** using the Atlassian MCP connector:
   - Get the full issue: description, status, priority, reporter, assignee
   - Get the latest comments (up to 10)
   - Note the reporter's name — this is the customer you're drafting a response to

3. **Draft a customer response** using the `/ticket-response` skill logic:
   - Use the ticket-response skill guidelines to compose the reply
   - The response must be a Gmail draft reply on the SAME thread (use the thread ID)

4. **Save the draft** using Gmail MCP — create a draft reply, do NOT send it.

## Step 4: Send Slack Summary

After processing ALL emails, send a Slack DM to Payal with this format:

```
Inbox Processing Complete

Emails processed: [total count]
Added to delete queue: [count]
Jira drafts ready: [count]
  - [TICKET-ID]: [one-line subject summary]
  - [TICKET-ID]: [one-line subject summary]
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
