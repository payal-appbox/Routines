# Routines

Cloud-based automation routines powered by Claude Code, running on
[claude.ai/code/routines](https://claude.ai/code/routines).

## Overview

This repo contains scheduled routines that automate repetitive service desk tasks using
MCP connectors (Gmail, Atlassian, Slack). No local CLI tools — everything runs in
Anthropic's cloud infrastructure.

**Owner:** Payal Malhotra — Service Desk Agent (Global Service Desk) & Product Manager at Appbox.ai

## Routines

### Inbox Processor

**Schedule:** 10:00 AM and 8:00 PM IST daily

Processes unread Gmail inbox:
1. **Classifies** each email (Jira notification, junk/marketing, or important)
2. **Labels junk** as `claude-delete` (never deletes)
3. **Drafts replies** for Jira ticket emails using full ticket context from Atlassian
4. **Sends a Slack DM** summary of everything processed

### Ticket Response (reusable skill)

Standalone skill that takes a Jira ticket ID, reads context via Atlassian MCP, and
composes a professional customer response. Used by Inbox Processor and available for
ad-hoc use.

## Safety Guardrails

- Emails are **drafted, never sent**
- Jira tickets are **read, never modified**
- Emails are **labeled, never deleted**

See [CLAUDE.md](CLAUDE.md) for the full safety policy.
