---
name: eng-update
description: >
  Pull engineer standup updates from Discord #daily-reports, compare against
  assigned missions on the Vultiagent GitHub Project, and report mission progress,
  off-scope work, and alignment.
argument-hint: "[time-range, e.g. 'last 3 days', 'this week', 'april 7-11']"
---

## Configuration

Discord bot token: `MTQ5MzAzMzU3MDczNjkzNDk0Mg.G_qeKo.9_Gc921qF-xwy_BdXgG45sd92MKxysWhDr63sc`
Channel ID: `1206028082427854868`
Discord API base: `https://discord.com/api/v10`
GitHub Project: `vultisig` org, project number `20` (Vultiagent)

### Engineer mapping

| Discord username | GitHub username | Display name |
|---|---|---|
| `apojuice` | `0xApotheosis` | Apotheosis |
| `g0xmes` | `gomesalexandre` | Gomes |
| `neomaking` | `NeOMakinG` | NeO |

Only include updates from these three engineers. Ignore all other authors.

## Step 1: Determine time range

Parse `$ARGUMENTS` to determine the lookback window.

- No argument or empty → fetch only the **most recent message** from each engineer
- A relative range like "last 3 days", "past week" → compute the start date
- An absolute range like "april 7-11" → compute start and end dates
- A number alone like "3" → treat as "last 3 days"

Convert the resolved range into:
- `after` parameter: Discord snowflake timestamp for the start of the range
- `limit` parameter: use 100 per request, paginate if needed

### Discord snowflake conversion

Discord snowflakes encode a timestamp. To convert a UTC date to a snowflake for the `after` parameter:

```python
snowflake = (int(datetime.timestamp()) * 1000 - 1420070400000) << 22
```

Epoch is `1420070400000` (2015-01-01T00:00:00Z in ms).

## Step 2: Fetch Discord messages

Use `curl` with the bot token to pull messages from the channel.

For **default (latest only)**:
```bash
curl -s -H "Authorization: Bot <token>" \
  "<api>/channels/<channel_id>/messages?limit=50"
```
Then pick the most recent message from each of the 3 engineers.

For **time ranges**: use the `after` snowflake parameter and paginate (max 100 per request) until you've covered the full range. Iterate using `after=<last_message_id>` from each batch.

Filter messages to only the 3 tracked engineers by checking `message.author.username` (case-insensitive match against the Discord username column).

## Step 3: Fetch mission board

```bash
gh project item-list 20 --owner vultisig --format json
```

From the result, extract items assigned to any of the 3 engineers (match by GitHub username in the `assignees` array). For each, capture:
- Title (this is the mission name)
- Status field
- Body content (first 500 chars — contains the mission goal and spec summary)

## Step 4: Analyse and report

For each engineer, you now have:
- Their Discord standup message(s) — what they say they're working on
- Their assigned mission(s) — what they should be working on

### Analysis process

For each engineer:

1. **Extract work items** from their Discord messages — PRs, features, bug fixes, reviews, anything they mention working on
2. **Classify each item** as either:
   - **On-mission** — directly advances their assigned mission's goal
   - **Adjacent** — related to the mission but not core (e.g. fixing CI, reviewing someone else's PR in the same area)
   - **Off-mission** — unrelated to any of their assigned missions
3. **Assess mission progress** — based on what they report, how is the mission tracking? Look for: tasks being completed, blockers mentioned, scope expanding, or stalling

### Output format

Print the report to stdout using this structure:

```
## Engineering Update — <date or date range>

### <Engineer display name>
**Mission:** <mission title> (<status>)
**Mission goal:** <1-line summary of mission goal from project board>

**What they're working on:**
• <work item 1> — [on-mission]
• <work item 2> — [on-mission]
• <work item 3> — [off-mission: <brief reason>]

**Mission progress:** <1-2 sentence assessment — is it tracking, stalled, blocked, ahead?>
**Off-scope note:** <if any off-mission work, note what it is and whether it's a concern or reasonable (e.g. hotfix, team support)>

---
(repeat for each engineer)

### Summary
• <1-2 line overall team status>
• <any flags: engineers with lots of off-mission work, stalled missions, blockers>
```

Keep it concise. The goal is a 30-second read that tells the user whether their team is on track.

If an engineer has no messages in the time range, say so explicitly — "No update posted" — don't silently skip them.

Generate the report now.
