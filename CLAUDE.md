# Routines — Cloud Automation for Service Desk

## Purpose

This repo contains cloud-based automation routines that run on Anthropic's infrastructure
via [claude.ai/code/routines](https://claude.ai/code/routines). Routines are scheduled
Claude Code agents that execute tasks using MCP connectors — no local CLI tools.

## Owner

**Payal Malhotra** — Service Desk Agent (Global Service Desk) & Product Manager at Appbox.ai

## Architecture

All routines operate exclusively through MCP connectors:

- **Gmail** — read emails, create drafts, apply labels
- **Atlassian** — read Jira tickets and Confluence pages
- **Slack** — send messages, read channels

No local tools (gws, jira-cli, curl, etc.) are available in the routine execution
environment. Every external interaction must go through an MCP connector.

## Safety Rules — MUST FOLLOW

These rules apply to ALL routines in this repo. Violating them could cause real harm.

### Email (Gmail)

- **NEVER send emails.** Only create drafts. The human reviews and sends.
- **NEVER delete emails.** Only apply labels (e.g., `claude-delete` for junk).
- **NEVER modify email content** in existing threads beyond drafting replies.

### Jira (Atlassian)

- **NEVER modify Jira tickets.** Read-only access: fetch issue details, comments, status.
- **NEVER transition tickets**, change assignees, or add comments via automation.
- **NEVER create new Jira issues** from routines.

### Slack

- Slack messages are used for summaries and notifications only.
- Always send to the owner (Payal) via DM unless explicitly configured otherwise.

### General

- When in doubt, do less. It is always safe to skip an action and flag it for human review.
- Log decisions: if a routine skips an email or flags something unusual, include it in the
  Slack summary so the owner has full visibility.

## Routine Inventory

| Routine | Skill | Schedule | Description |
|---------|-------|----------|-------------|
| Inbox Processor | `/inbox-processor` | 10 AM & 8 PM IST daily | Classifies inbox, drafts Jira responses, labels junk |

## Conventions

- Skills live in `.claude/skills/` and are invoked by routines.
- Each skill should be self-contained and reusable where possible.
- Skill files use the standard Claude Code skill frontmatter format.
- Draft responses should be professional, empathetic, and actionable.
- Never say "our domain gets blocked" — frame network issues as "firewall/network
  configuration on your end may need adjustment."
