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

**Action:** Apply the Gmail label `claude-delete` to this email. Do NOT delete it.

### Category C — Important / Other
Everything that doesn't fit Category A or B.

**Action:** Leave it in the inbox untouched. Add it to the summary as "flagged but not actioned."

## Step 3: Process Jira Notification Emails

For each Jira notification email (Category A):

1. **Extract the ticket ID** from the subject line or email body (e.g., `SD-1234`)

2. **Fetch ticket context** using the Atlassian MCP connector:
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
Labeled claude-delete: [count]
Jira drafts ready: [count]
  - [TICKET-ID]: [one-line subject summary]
  - [TICKET-ID]: [one-line subject summary]
Important but not actioned: [count]
  - From: [sender] — Subject: [subject]
```

If there were zero emails to process, still send a summary confirming that.

## Error Handling

- If Atlassian MCP fails to fetch a ticket, skip drafting for that email. Include it in the
  Slack summary as: "Could not fetch [TICKET-ID] — Atlassian MCP error. Needs manual review."
- If Gmail draft creation fails, note it in the summary.
- Never let one email's failure stop processing of the remaining emails.
