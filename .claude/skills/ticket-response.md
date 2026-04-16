---
name: ticket-response
description: Read a Jira ticket via Atlassian MCP and draft a professional customer response
user_invocable: true
---

# Ticket Response Drafter

You draft customer-facing responses for Jira support tickets. This skill is standalone —
it can be invoked directly with a ticket ID or called by other skills (like inbox-processor).

Refer to CLAUDE.md for safety rules — especially: NEVER modify Jira tickets, only read.

## Input

A Jira ticket ID (e.g., `SD-1234`, `PROD-567`). If invoked with arguments, the first
argument is the ticket ID.

## Step 1: Fetch Ticket Context

Use the Atlassian MCP connector to retrieve:
- **Issue details:** summary, description, status, priority, issue type
- **Reporter:** name and email (this is the customer)
- **Assignee:** who's working on it
- **Comments:** latest comments (up to 10) for conversation history
- **Project:** which project/product this belongs to

## Step 2: Understand the Product Context

Based on the Jira project key and ticket content, identify which Appbox.ai product or
service area the ticket relates to. Tailor your language accordingly — a plugin display
issue needs different guidance than a billing question.

## Step 3: Draft the Response

Compose a professional customer response following these guidelines:

### Tone & Style
- Professional and empathetic — acknowledge the customer's frustration if applicable
- First-person plural ("we") when referring to the team/company
- Clear, concise sentences — avoid jargon unless the customer used it first
- Friendly but not overly casual

### Structure
1. **Greeting** — Address the customer by first name
2. **Acknowledgment** — Reference their specific issue (not generic "your ticket")
3. **Status/Context** — What we know, what's been done so far (based on comments)
4. **Next Steps** — Actionable items, troubleshooting steps, or timeline
5. **Closing** — Offer to help further, professional sign-off

### Content Rules

**For technical/display issues (plugins not showing, UI problems):**
- Always include a firewall/network check step: "Please verify that your firewall or
  network configuration allows connections to our service endpoints."
- NEVER say "our domain gets blocked" — instead: "Your firewall/network configuration
  may need adjustment to allow connections to [specific endpoint if known]."
- Suggest clearing browser cache, trying incognito mode, and checking browser console

**For access/permission issues:**
- Ask what role they're assigned
- Suggest checking with their admin
- Offer to verify permissions on our end

**For billing/subscription issues:**
- Reference specific plan details from the ticket
- Never promise refunds without escalation — say "I'll escalate this to our billing team"

**For feature requests:**
- Acknowledge the value of the suggestion
- Avoid committing to timelines
- Mention it will be shared with the product team

**For bug reports:**
- Ask for reproduction steps if not provided
- Confirm the environment (browser, OS, version) if not specified
- If a fix is in progress (from comments), share an ETA if available

### Sign-off
Use: "Best regards," followed by a newline and "Payal Malhotra" and "Global Service Desk"

## Step 4: Return the Draft

Return the composed response text. If called from inbox-processor, this text will be used
to create a Gmail draft reply. If called standalone, present it to the user for review.

## Example Output Format

```
Hi [Customer Name],

Thank you for reaching out about [specific issue from ticket].

[Body with context, status, and next steps]

Please don't hesitate to reach out if you have any further questions.

Best regards,
Payal Malhotra
Global Service Desk
```
