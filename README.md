# Birthday Reminders via Claude

Replace paid birthday reminder apps with a free, self-hosted solution using [Claude](https://claude.ai) desktop app's scheduled tasks and iMessage.

You get two notifications per birthday:
- **7-day warning**: a heads-up text a week before
- **Day-of reminder**: a "Happy birthday!" text on the actual day

All birthday data lives in a single JSON file you can edit anytime.

---

## Prerequisites

- **macOS** (iMessage is required for sending texts)
- **Claude desktop app** with [Cowork mode](https://claude.ai) enabled
- **Read and Send iMessages** MCP connector installed in Claude
- **Full Disk Access** granted to `node` (System Settings > Privacy & Security > Full Disk Access) — required so Claude can read and send iMessages

## Setup

### Step 1: Create your birthday list

Copy `birthdays.template.json` to a permanent location on your Mac and rename it `birthdays.json`:

```bash
cp birthdays.template.json ~/Documents/birthdays.json
```

Edit it with your actual birthdays. Each entry is simple:

```json
{
  "birthdays": [
    { "name": "Jane Doe", "month": 3, "day": 15 },
    { "name": "John Smith", "month": 7, "day": 22 }
  ]
}
```

- `name`: however you want to see the person's name in your reminder text
- `month`: 1-12
- `day`: 1-31

### Step 2: Create the scheduled task

Open a Cowork session in Claude desktop and paste the following prompt:

> Create a scheduled task called "birthday-reminders" that runs daily at 9 AM. Every day it should:
>
> 1. Read my birthday list from `~/Documents/birthdays.json`
> 2. Check if anyone's birthday is **today** or **exactly 7 days from now**
> 3. If today: text me "Happy birthday to [Name]!" via iMessage
> 4. If in 7 days: text me "Heads up — [Name]'s birthday is in 7 days ([Month Day])!" via iMessage
> 5. If nothing matches, send no message
>
> My phone number is [YOUR_PHONE_NUMBER].

Claude will create the scheduled task for you. Alternatively, you can copy the task template from `scheduled-task-template.md` in this repo and create the task manually.

### Step 3: Approve permissions

After creating the task, click **"Run now"** in the Scheduled section of Claude's sidebar. This triggers a test run and lets you approve the iMessage tool permissions so future automated runs don't get stuck.

### Step 4: You're done

The task runs every morning at 9 AM. Cancel your old birthday reminder subscription.

## Managing your birthday list

Your `birthdays.json` file is the single source of truth. The scheduled task reads it fresh every morning.

**Add someone:**
```json
{ "name": "New Friend", "month": 8, "day": 14 }
```
Add a new entry to the `birthdays` array. Don't forget the comma after the previous entry.

**Remove someone:**
Delete their line from the array.

**Change a date:**
Edit the `month` and `day` values.

You can also ask Claude in any Cowork session: *"Add Sarah Chen, September 12th to my birthdays.json"* and it will edit the file for you.

## Importing from an existing birthday app

If you already have birthdays stored in a text-based service (like a text chain with a birthday bot), you can ask Claude to extract them automatically. Here's how this project was originally built:

1. Opened a Cowork session and gave Claude access to iMessages via the Read and Send iMessages connector
2. Asked Claude to read the full message history with the birthday app's phone number
3. Claude parsed all the messages, extracted names and dates, and built the `birthdays.json` file automatically
4. Then created the scheduled task to replace the paid service entirely

This same approach works for any text-based birthday data source.

## Importing from Facebook

You can bulk-import your Facebook friends' birthdays using Claude Code with a browser automation tool (like the Chrome DevTools MCP server). Facebook's birthdays page (`facebook.com/friends/birthdays`) only shows individual dates for the current month — but the underlying GraphQL API has all 12 months.

### How it works

1. Log into Facebook in Chrome and navigate to `facebook.com/friends/birthdays`
2. Claude extracts the `fb_dtsg` token and the `BirthdayCometContentQuery` query ID from the page's embedded JSON
3. For each month offset (0–11), Claude makes a GraphQL request to `/api/graphql/` with:
   ```json
   {
     "offset_month": 0,
     "scale": 2,
     "stream_birthday_months": true
   }
   ```
4. Each response contains friend nodes with `"name"` and `"birthdate": {"day": N, "month": N}` fields
5. Claude splits on `"node":{` boundaries and pairs each name with its birthdate
6. Results are deduplicated across all 12 requests and sorted by date

### Deduplication

When merging Facebook birthdays into an existing `birthdays.json`, Claude:

- **Exact match** (same name + date): skips the entry
- **Same date, similar name** (e.g., "Kieran" vs "Kieran Bondy"): updates the existing entry with the full Facebook name
- **Similar name, different date**: flags as a potential conflict for you to resolve
- **No match**: adds as a new entry

### Steps

1. Open Facebook's birthdays page in Chrome while logged in
2. Ask Claude Code (with Chrome DevTools MCP access) to pull down your Facebook birthdays
3. Claude exports a CSV you can review — mark entries with "x" in the `Add?` column
4. Claude reads back your selections, deduplicates against your existing list, and merges

## How it works under the hood

The scheduled task is a Claude skill file stored at:
```
~/Documents/Claude/Scheduled/birthday-reminders/SKILL.md
```

Every day at 9 AM (with a backup run at noon), Claude:
1. Checks `.birthday-log.json` to see if today has already been processed (prevents duplicate texts)
2. Reads `birthdays.json` from disk
3. Runs `date` commands in bash to get today's date and the date 7 days out
4. Compares each birthday against both dates
5. Sends iMessages via the Read and Send iMessages MCP connector (only if there's a match)
6. Writes today's date to the log file
7. Checks if yesterday was missed — if so, sends catch-up reminders prefixed with "⚠️ Missed yesterday:"
8. Does nothing if no birthdays are coming up

## Reliability

Since the scheduled task runs locally on your Mac, it won't fire if your laptop is asleep or powered off. Two mitigations:

### Auto-wake schedule

Use `pmset` to wake your Mac one minute before the task:

```bash
sudo pmset repeat wakeorpoweron MTWRFSU 08:59:00
```

Verify with `pmset -g sched`. Also make sure your sleep timer gives the task enough time to complete:

```bash
sudo pmset -a sleep 10
```

### Dedup and catch-up

The task writes a `.birthday-log.json` file after each successful run. This provides:

- **Deduplication**: if the task runs multiple times in one day (e.g., you trigger it manually), it won't send duplicate texts
- **Catch-up**: if the task missed an entire day, the next run detects the gap and sends yesterday's reminders with a warning prefix

## Customization ideas

- **Change notification time**: Ask Claude to update the cron schedule (e.g., "change my birthday reminders to 8 AM")
- **Add email notifications**: Install the Gmail connector and ask Claude to also send an email
- **Change the warning window**: Modify the task to check 14 days ahead instead of 7
- **Add year tracking**: Add a `year` field to track ages if you want (e.g., "Jake turns 30!")

## License

MIT — do whatever you want with it.
