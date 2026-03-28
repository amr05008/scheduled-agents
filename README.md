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
• 33°F, Clear skies
• Wind: NW 14 mph, gusts to 28 mph
• Humidity: 49%

📊 Today's forecast
• High: 45°F / Low: 33°F
• Conditions: Mostly sunny, NW winds 14–16 mph with gusts to 28 mph

🌧 Precipitation
• None expected today
• Next chance: 30% Monday night after 2 AM

🚴 Cycling outlook
• Cold start at 33°F, warming to 45°F by afternoon — dress in layers
• Gusty NW winds 14–28 mph — significant headwind/crosswind depending on route
• Clear and sunny all day — good visibility, no precipitation risk

✅ Confidence: High — hourly and period forecasts are consistent

_Source: NOAA/NWS_
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
4. It fetches weather from NOAA/NWS and your configured RSS feeds
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
    "url": "https://forecast.weather.gov/MapClick.php?textField1=YOUR_LAT&textField2=YOUR_LON",
    "location_label": "Your City"
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

**Finding your weather coordinates:** Go to [forecast.weather.gov](https://forecast.weather.gov), search your location, and copy the lat/lon from the URL.

**Finding RSS feeds:** Most blogs and newsletters have a feed at `/feed` or `/atom.xml`. Substack newsletters are always at `https://yourpublication.substack.com/feed`.

### 4. Set up Discord delivery

1. In Discord, go to your channel → Edit Channel → Integrations → Webhooks → New Webhook
2. Copy the webhook URL
3. Paste it into `config.json` under `delivery.discord_webhook.url`
4. Set `enabled: true`

No bot setup, no OAuth — Discord webhooks are just HTTPS endpoints.

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

## Repo structure

```
briefing/
  prompt.md          ← agent instructions (the "how")
  user-context.md    ← your personal context (the "who")
  config.json        ← feeds, weather, delivery channels (the "what")
  output/
    YYYY-MM-DD.md    ← daily archive committed by the agent
```
