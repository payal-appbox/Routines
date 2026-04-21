---
name: clean-inbox
description: Review and batch-delete emails from the delete queue, one category at a time
user_invocable: true
---

# Clean Inbox

Interactive skill for reviewing and deleting queued junk emails. Requires `gws` installed
locally (Google Workspace CLI).

## Step 1: Load and Group

Read `delete-queue.json` from the repository root. Group entries by the `category` field.
Present a summary table to the user:

```
Delete Queue Summary ([total] threads pending)

| # | Category              | Count | Example Subject                    |
|---|-----------------------|-------|------------------------------------|
| 1 | Marketing/Promo       | 4     | Meet AI All-Access Pack...         |
| 2 | Newsletter/Event      | 8     | Team '26 Founder Keynote...        |
| 3 | Automated Notification| 1     | Daily update for 7/26/2025         |
| ...                                                                       |

Pick a category number to review, or "all" to see everything.
```

## Step 2: Review Category

When the user picks a category, show all entries in that category:

```
Newsletter/Event (8 threads):

| # | Date       | Sender                          | Subject                          |
|---|------------|---------------------------------|----------------------------------|
| 1 | 2026-04-17 | servicedeskagent@appbox.ai      | Team '26 Founder Keynote...      |
| 2 | 2026-04-17 | khaldrogo@appbox.ai             | Team '26 Founder Keynote...      |
| ...

Reason: Duplicate Atlassian event newsletter sent to test account

Actions:
- "delete all" — trash all threads in this category
- "skip" — go back to category list
- "remove #3" — remove a specific entry (false positive) before deleting
```

Wait for user input before proceeding.

## Step 3: Execute Deletion

When the user confirms deletion for a category:

1. For each thread in the confirmed set, run:
   ```
   gws gmail users.threads trash --userId me --id [thread_id]
   ```
   Use `trash` not `delete` — this moves to trash (recoverable for 30 days).

2. If a thread fails to trash, note it and continue with the rest.

3. After processing, report results:
   ```
   Trashed 8/8 threads in "Newsletter/Event"
   ```

## Step 4: Update the Queue

After successful deletion:

1. Remove the trashed entries from `delete-queue.json`
2. Update the `last_updated` timestamp
3. If the user wants to continue, return to Step 1 with the updated queue
4. When done, commit the updated `delete-queue.json` with message:
   "chore: flush [N] threads from delete queue"

## Error Handling

- If `gws` is not installed or not authenticated, inform the user and stop.
- If a thread was already deleted/trashed (404), silently remove it from the queue.
- If network errors occur, retry once, then skip and report.
- Never modify `delete-queue.json` until deletion is confirmed by the user.

## Notes

- This skill runs locally only (requires `gws` CLI).
- Trashing is reversible — emails stay in Gmail trash for 30 days.
- The user can say "remove #N" to rescue a false positive before bulk-deleting.
