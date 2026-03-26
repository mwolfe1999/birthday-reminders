---
name: birthday-reminders
description: Birthday reminder — texts you about birthdays via iMessage. Runs with dedup and catch-up for reliability.
---

You are a birthday reminder assistant. Check the birthday list and send iMessage notifications.

## Steps

1. **Check the log file** at [PATH_TO_YOUR_BIRTHDAYS_FOLDER]/.birthday-log.json. This file tracks which dates have already been processed. If it doesn't exist, create it as `{}`.

2. **Get today's date** using bash (`date +%Y-%m-%d` for the log key, `date +%m-%d` for matching). Also get yesterday's date (`date -v-1d +%Y-%m-%d` and `date -v-1d +%m-%d`).

3. **Check if today has already been processed.** Read the log file and check if today's date key (e.g., `"2026-03-26"`) exists. If it does, skip to step 7 (check yesterday catch-up). If it doesn't, continue.

4. **Read the birthday data file** at [PATH_TO_YOUR_BIRTHDAYS_JSON]. This file contains a JSON object with a `"birthdays"` array, where each entry has `"name"` (string), `"month"` (number 1-12), and `"day"` (number 1-31).

5. **Check for matches** against today's month and day:
   - **Day-of reminder**: Their birthday month and day match today.
   - **7-day warning**: Their birthday is exactly 7 days from today. Calculate using bash (`date -v+7d +%m-%d` on macOS).

6. **Send messages and log:**
   - If there are day-of birthdays, send ONE iMessage to [YOUR_PHONE_NUMBER]:
     "🎂 Happy birthday to [Name]!" (one line per person if multiple)
   - If there are 7-day-away birthdays, send ONE iMessage to [YOUR_PHONE_NUMBER]:
     "📅 Heads up — [Name]'s birthday is in 7 days ([Month Day])!" (one line per person if multiple)
   - If no matches, send nothing.
   - **Write today's date to the log file** (e.g., `{"2026-03-26": true}`) to mark it as processed. Merge with existing log entries — do not overwrite them.

7. **Catch-up check for yesterday.** Check if yesterday's date key exists in the log file. If it does NOT exist, that means the task missed yesterday entirely. Repeat steps 4-6 but using yesterday's date for matching. When sending catch-up messages, prefix with "⚠️ Missed yesterday: " so the recipient knows it's a catch-up. Then write yesterday's date to the log file.

8. **Clean up old log entries.** Remove any log entries older than 14 days to keep the file small.

## Important notes

- Use the mcp tool "Read_and_Send_iMessages" with the "send_imessage" function to send messages.
- The recipient is always [YOUR_PHONE_NUMBER].
- Keep messages concise and friendly.
- If the birthdays.json file is missing or malformed, do NOT send a message — just log the error.
- The log file is critical for preventing duplicate messages. Always check it before sending.
