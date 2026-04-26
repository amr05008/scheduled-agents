# Trigger Instructions — TEMPLATE

This is the source of truth for what you paste into the **Instructions**
field of the scheduled trigger at https://claude.ai/code/scheduled.

Copy this file to `trigger-instructions.local.md` (gitignored) and fill
in your real webhook URL + VIPs. When you ever need to re-create the
trigger, the `.local` file is what you copy-paste.

---

## Watcher trigger (every hour weekday work hours)

**Routine name:** `Inbox Watcher`

**Schedule:** Custom cron `30 12-22 * * 1-5`
(= every hour at :30, Mon-Fri 8:30am-6:30pm ET in EDT;
 update to `30 13-23 * * 1-5` when EST kicks in November)

**Connectors:** Gmail (only)

**Repo:** your fork of `scheduled-agents`

**Instructions field — paste exactly:**

```
You are the inbox-watcher agent. The scheduled-agents repo has been
cloned into your working directory.

CONFIG (private to this trigger, supersedes anything in the repo):

DISCORD_WEBHOOK_URL=<paste your webhook URL here>

VIPS (treat as if listed in inbox-watch/vips.yaml — exact email or
wildcard domain match; always flag, never auto-action):
- person@example.com           (replace with real VIPs, one per line)
- *@bigclient.example.com
- partner@example.com
- fraud-alerts@yourbank.com
- (add 3-7 more)

STEPS:

1. Run: date +%Y-%m-%d
2. Read inbox-watch/prompt.md and follow it exactly.
3. When the prompt tells you to read inbox-watch/vips.yaml, treat the
   VIPS list above as the authoritative source. Use vips.yaml only as
   a fallback if VIPS above is empty.
4. The DISCORD_WEBHOOK_URL above is what the prompt expects in the
   $DISCORD_WEBHOOK_URL env var.
```

---

## Weekly-audit trigger (Sundays 9am ET)

**Routine name:** `Inbox Watcher — Weekly Audit`

**Schedule:** Custom cron `0 13 * * 0`
(= Sundays 9am ET in EDT; update to `0 14 * * 0` when EST kicks in)

**Connectors:** Gmail (only)

**Repo:** your fork of `scheduled-agents`

**Instructions field — paste exactly:**

```
You are the inbox-watcher weekly-audit agent. The scheduled-agents
repo has been cloned into your working directory.

CONFIG (private to this trigger):

DISCORD_WEBHOOK_URL=<paste your webhook URL here>

STEPS:

1. Run: date +%Y-%m-%d
2. Read inbox-watch/weekly-audit-prompt.md and follow it exactly.
3. The DISCORD_WEBHOOK_URL above is what the prompt expects in the
   $DISCORD_WEBHOOK_URL env var.
```

---

## After both triggers exist

- Hit "Run now" on the watcher to smoke-test
- Don't run the weekly audit manually until the watcher has 1+ week of
  output logs (it has nothing to audit yet)
- If you ever change the webhook URL or VIPs: update them in BOTH the
  trigger UI AND in this `.local.md` file so they stay in sync
