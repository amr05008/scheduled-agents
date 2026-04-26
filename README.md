# scheduled-agents

A self-hosted daily briefing powered by Claude Code scheduled agents — an alternative to digest tools like OpenClaw.

Every morning at 6am, a Claude agent spins up in Anthropic's cloud, fetches weather and your RSS feeds, and posts a formatted briefing to Discord (or any channel you configure). No server to maintain, no subscription beyond Claude Code.

---

## What it looks like

Two Discord messages arrive each morning:

**Weather** — current conditions, today's forecast, cycling/activity outlook, confidence rating:
```
[Claude] [WEATHER] Brooklyn, NY — Saturday, March 28

🌡️ Current conditions (as of 6:02 AM ET)
• 33°F / 1°C, Clear skies
• Wind: NW 14 mph, gusts to 28 mph
• Humidity: 49%

📊 Today's forecast
• High: 45°F / 7°C — Low: 33°F / 1°C
• Conditions: Mostly sunny, NW winds 14–16 mph with gusts to 28 mph

🌧 Precipitation
• None expected today
• Next chance: 30% Monday night after 2 AM

🚴 Cycling outlook
• Cold start at 33°F / 1°C, warming to 45°F / 7°C — dress in layers
• Gusty NW winds 14–28 mph — significant headwind/crosswind depending on route
• Clear and sunny all day — good visibility, no precipitation risk

✅ Confidence: High — hourly and period forecasts are consistent

_Source: wttr.in_
```

**Feeds** — one message per source, new posts only, no link previews:
```
[Claude] [SIMON WILLISON] Saturday, March 28

• Vibe coding SwiftUI apps is a lot of fun — Willison built two macOS apps using Claude with minimal prompting
• We Rewrote JSONata with AI in a Day, Saved $500K/Year — Case study of AI-assisted port to Go
...
```

---

## How it works

Claude Code lets you schedule remote agents on a cron schedule. Each run:

1. Anthropic's cloud clones this repo into an isolated environment
2. The agent reads `briefing/prompt.md` for instructions
3. It reads `briefing/user-context.md` to personalize the output
4. It fetches weather from [wttr.in](https://wttr.in) (global coverage, °F and °C) and your configured RSS feeds
5. It filters RSS to only posts published in the last 24 hours
6. It posts to Discord (or other configured channels)
7. It commits a full markdown archive to `briefing/output/`

No server, no cron job, no infrastructure. Just a repo and a trigger.

---

## Setup

### Prerequisites

- [Claude Code](https://claude.ai/code) account — **Pro plan or above required** for scheduled triggers. The free tier does not support them.
- GitHub account — repo can be public or private

### 1. Fork this repo

Fork to your own GitHub account. The trigger will clone it fresh on every run.

### 2. Edit `briefing/user-context.md`

This is the most important file — it's what makes the briefing *yours*. Describe:
- Your interests and what to prioritize in feeds
- Your weather priorities (e.g. cyclist = wind first, runner = humidity, etc.)
- Tone and format preferences

### 3. Edit `briefing/config.json`

```json
{
  "weather": {
    "location": "Your City, Country"
  },
  "feeds": [
    { "name": "Feed Name", "url": "https://example.com/feed", "max_items": 5 }
  ],
  "delivery": {
    "discord_webhook": {
      "enabled": true,
      "url": "YOUR_DISCORD_WEBHOOK_URL"
    }
  }
}
```

**Finding RSS feeds:** Most blogs and newsletters have a feed at `/feed` or `/atom.xml`. Substack newsletters are always at `https://yourpublication.substack.com/feed`.

### 4. Set up Discord delivery

1. In Discord, go to your channel → Edit Channel → Integrations → Webhooks → New Webhook
2. Copy the webhook URL
3. Set `delivery.discord_webhook.enabled: true` in `config.json` — **leave `url` empty**
4. Instead, add the URL to your trigger's bootstrap prompt (keeps it out of your public repo):

```
The repo has been cloned into your working directory.
Get today's date by running: date +%Y-%m-%d
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/YOUR_WEBHOOK_URL
Then read briefing/prompt.md and follow its instructions exactly.
```

No bot setup, no OAuth — Discord webhooks are just HTTPS endpoints. Storing the URL in the trigger (not the repo) means you can keep your repo public without exposing it.

### 5. Create the scheduled trigger

Open Claude Code and run `/schedule`, or go to [claude.ai/code/scheduled](https://claude.ai/code/scheduled).

Create a new trigger with these settings:

| Setting | Value |
|---------|-------|
| Name | `Daily Briefing` |
| Schedule | `0 10 * * *` (6am ET — adjust for your timezone, cron is UTC) |
| Environment | Internet Access |
| Git source | Your fork's GitHub URL |
| Model | `claude-sonnet-4-6` |
| Tools | `Bash, Read, Write, Edit, Glob, Grep, WebFetch` |

**Bootstrap prompt** (paste this exactly):
```
The repo has been cloned into your working directory.
Get today's date by running: date +%Y-%m-%d
Then read briefing/prompt.md and follow its instructions exactly.
```

### 6. Test it

Hit "Run now" from [claude.ai/code/scheduled](https://claude.ai/code/scheduled). The agent takes 1–2 minutes to run. Check Discord and `briefing/output/` in your repo to verify.

---

## Customizing

### Switch weather location (e.g. when traveling)

Leave `weather.location` set to your home city and add a time-boxed entry to `weather.travel`. The agent picks the first active entry (where today falls between `start`, if set, and `end` inclusive) and auto-reverts to home once `end` passes — no second push needed.

```json
"weather": {
  "location": "Brooklyn, NY",
  "travel": [
    { "location": "Amsterdam, Netherlands", "end": "2026-04-24" },
    { "location": "Tokyo, Japan", "start": "2026-06-10", "end": "2026-06-18" }
  ]
}
```

Dates are `YYYY-MM-DD` and interpreted in the agent's timezone. `start` is optional (omit for "starts immediately"); `end` is required and inclusive. Expired entries are ignored, so you can delete them whenever.

Weather is powered by [wttr.in](https://wttr.in), which works for any city worldwide. Temperatures are shown in both °F and °C. Just use a city name — `Tokyo`, `London`, `Mexico City` — anything wttr.in recognizes. To verify a location works, visit `https://wttr.in/Your+City?format=j1` in your browser.

### Change the time
Update the cron expression on your trigger. Cron runs in UTC — use [crontab.guru](https://crontab.guru) to convert. Remember to adjust in March (EDT, UTC-4) and November (EST, UTC-5).

### Add or remove feeds
Edit the `feeds` array in `config.json`. Feeds are filtered to the last 24 hours automatically, so weekly newsletters will show "No new posts today" on quiet days.

### Change what the briefing focuses on
Edit `briefing/user-context.md` — the agent reads it on every run. No trigger changes needed.

### Change the format or add sections
Edit `briefing/prompt.md`. This is where the agent instructions live. You can add sections, change the tone, restructure the output — the agent will follow it.

---

## Adding delivery channels

| Channel | What's needed |
|---------|--------------|
| **Discord** | Webhook URL in `config.json` — no connector needed |
| **Slack** | Same as Discord — Slack incoming webhooks work identically |
| **Email** | Connect an email MCP (Resend, etc.) at [claude.ai/settings/connectors](https://claude.ai/settings/connectors) |
| **SMS** | Connect a Twilio MCP at [claude.ai/settings/connectors](https://claude.ai/settings/connectors) |

---

## Adding more agents

Each agent is just a directory with a `prompt.md` and `config.json`. Drop a new one (e.g. `digest/`, `alerts/`, `weekly-summary/`) and create a separate trigger pointing to it.

---

## Inbox Watcher (`inbox-watch/`)

A second scheduled agent that watches Gmail every 30 min during weekday work hours and posts a Discord ping ONLY when something needs your attention. The point is to **stop checking email constantly** — let Discord interrupt you when (and only when) it matters.

**How it differs from `briefing/`:**
- Runs every 30 min Mon-Fri (not once daily)
- Reads Gmail via the `mcp__claude_ai_Gmail__*` connector (not RSS/web)
- Posts to Discord only when ≥1 VIP or high-confidence needs-attention email arrived
- Silent runs send nothing — the only Discord chatter is "real" alerts plus a Sunday weekly self-audit

**Setup** (one-time):
1. Connect Gmail at [claude.ai/settings/connectors](https://claude.ai/settings/connectors)
2. Copy `inbox-watch/vips.example.yaml` → `inbox-watch/vips.yaml` and fill in real VIPs
3. Create a scheduled trigger pointing to `inbox-watch/prompt.md` with cron `*/30 12-22 * * 1-5` (UTC; = every 30 min Mon-Fri 8am-6pm EDT)
4. Create a second trigger for the Sunday self-audit pointing to `inbox-watch/weekly-audit-prompt.md` with cron `0 13 * * 0`
5. Both triggers need `DISCORD_WEBHOOK_URL` in their bootstrap (kept out of repo per the same pattern as the briefing agent)

**Key files:**
- `inbox-watch/prompt.md` — per-run agent instructions
- `inbox-watch/weekly-audit-prompt.md` — Sunday self-audit
- `inbox-watch/config.json` — mode (dry-run / apply), thresholds, label names
- `inbox-watch/triage-heuristics.md` — classification decision tree
- `inbox-watch/rules.yaml` — auto-archive patterns
- `inbox-watch/vips.yaml` — VIP list (gitignored)
- `inbox-watch/evals/` — fixture-based regression tests; run before any prompt change
- `inbox-watch/RECOVERY.md` — "if pings stop" 5-minute fix checklist

**Always start in dry-run mode.** `config.json` ships with `mode: "dry-run"` — the agent will report what it WOULD do but won't touch labels. After ~5 weekdays of clean Discord reports, edit to `"apply"` and push.

---

## Repo structure

```
briefing/
  prompt.md          ← agent instructions (the "how")
  user-context.md    ← your personal context (the "who")
  config.json        ← feeds, weather, delivery channels (the "what")
  output/
    YYYY-MM-DD.md    ← daily archive committed by the agent
```
