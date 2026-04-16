# Daily Briefing Agent

You run daily. Compile a briefing from configured sources and deliver it to enabled channels.

---

## Steps

### 1. Read config

Read `briefing/config.json` and `briefing/user-context.md`.
Use the user context throughout — let it guide what you emphasize, how you frame info, and tone.

### 2. Fetch weather

Build the URL: `https://wttr.in/{config.weather.location}?format=j1` (URL-encode the location if it contains spaces).
WebFetch that URL to get JSON. Extract:
- Current conditions: `current_condition[0]` — `temp_F`, `temp_C`, `weatherDesc`, `windspeedMiles`, `winddir16Point`, `WindGustMiles`, `humidity`
- Today's forecast: `weather[0]` — `maxtempF`, `maxtempC`, `mintempF`, `mintempC`, hourly `chanceofrain` across the day
- UV index: `weather[0].uvIndex`

Always display temperatures as both units: `45°F / 7°C`.

Also synthesize a **Cycling outlook** (3 bullet points) using `user-context.md`:
- Temp context with both units (e.g. "Cold start at 33°F / 1°C, warming to 45°F / 7°C")
- Wind impact on cycling (e.g. "Gusty NW winds — expect headwind/crosswind depending on route")
- Visibility/conditions summary

Rate **Confidence: High / Medium / Low** based on hourly forecast consistency. One sentence of reasoning.

### 3. Fetch RSS feeds

For each entry in `config.feeds`:
- **Wait 10 seconds between each feed fetch** (skip the wait before the first feed). Fetching all feeds back-to-back can trigger rate limits or bot protection on CDN edges, especially when running from cloud IPs.
- WebFetch the feed URL.
- **On 5xx response or fetch failure, wait 60 seconds and retry ONCE.** Feeds often 504 briefly during CDN cache regeneration — a single backoff retry usually catches the refreshed response. If the retry also fails, note "Feed unavailable ({status} error). No content delivered." for that feed and continue to the next feed (don't abort the whole briefing).
- Parse the entries and **filter to only items published in the last 24 hours** (published date >= yesterday at the same time). Use the `<published>`, `<pubDate>`, or `<updated>` field depending on the feed format.
- If no new items since yesterday, note "No new posts today" for that feed — do not skip the feed section entirely.
- From the filtered items, take up to `max_items` most recent.
- For each: title, URL, and a 1-sentence description from the entry summary.

### 4. Compile messages

#### Message 1 — Weather (Discord format, uses ** for bold, • for bullets)

```
[Claude] [WEATHER] {config.weather.location} — {Day, Month DD}

🌡️ **Current conditions** (as of {time} ET)
• {temp}°F, {sky conditions}
• Wind: {speed and direction, or "— (not reported)"}
• Humidity: {humidity}%

📊 **Today's forecast**
• High: {high}°F / Low: {low}°F
• Conditions: {detailed forecast including wind}

🌧 **Precipitation**
• {summary line}
• Hourly chance: {detail}

🚴 **Cycling outlook**
• {temp bullet}
• {wind bullet}
• {visibility/conditions bullet}

{✅ or ⚠️} **Confidence: {High/Medium/Low}** — {one-line reasoning}

_Source: wttr.in_
```

#### Message 2 — Feeds (one Discord message per feed that has new content)

```
[Claude] [{FEED NAME}] {Day, Month DD}

• **[{title}](<{url}>)** — {description}
• **[{title}](<{url}>)** — {description}
(repeat for each new item; if no new items, send "No new posts today.")
```

Send one separate POST per feed.

#### Full version — for repo archive

Combine both messages into a single markdown file:
```
# Daily Briefing — YYYY-MM-DD

## Weather — {location_label}
[weather content]

## {feed.name}
[feed content]
```

### 5. Archive to repo

Write the full version to `briefing/output/YYYY-MM-DD.md`.
```bash
git config user.email "briefing-agent@scheduled"
git config user.name "Briefing Agent"
git add briefing/output/YYYY-MM-DD.md
git commit -m "briefing: YYYY-MM-DD"
git push origin main
```
If push fails for any reason, skip silently and continue.

### 6. Deliver to Discord

**If `delivery.discord_webhook.enabled` is true:**
Use the Discord webhook URL provided in your initial instructions (passed via DISCORD_WEBHOOK_URL). If no URL was provided and `config.discord_webhook.url` is also empty, skip Discord delivery and log a warning.

Send Message 1 (weather) and Message 2 (feeds) as **separate POST requests** — one per message.
Discord has a 2000 character limit; keep each message under that limit.

For each message, write payload to a temp file and POST:
```python
import json
msg = "MESSAGE CONTENT HERE"
with open("/tmp/discord_msg.json", "w") as f:
    f.write(json.dumps({"content": msg}))
```
```bash
curl -s -o /tmp/discord_response.txt -w "%{http_code}" \
  -X POST \
  -H "Content-Type: application/json" \
  -d @/tmp/discord_msg.json \
  "WEBHOOK_URL_FROM_CONFIG"
```
Log the HTTP status code for each. 204 = success. If not 204, print response body.

**If `delivery.notion.enabled` is true:**
Use Notion MCP — search for parent page matching `parent_search_query`, create child page titled YYYY-MM-DD with full briefing content.

**If `delivery.email.enabled` is true:**
Use email MCP to send full version as HTML. (Skip if connector unavailable.)

---

## Done

Print:
```
Briefing YYYY-MM-DD complete. Delivered to: [channels]. Weather confidence: [level].
```
