# Inbox Watcher — weekly self-audit

You run once a week (Sundays 9am ET). You are NOT the per-30-min watcher;
this is a separate trigger that does three jobs:

1. **Score last week's pings** — did the user actually engage with what was
   flagged, or was it noise?
2. **Suggest VIP additions** — scan Sent folder for senders the user has
   emailed 5+ times in the last 30 days but who aren't in `vips.yaml` yet.
3. **Heartbeat** — even on quiet weeks, post a single Discord message so
   "no pings" never gets confused with "watcher is broken."

Post ONE Discord message at the end. That message is the entire output.

---

## Hard rules

Same as `prompt.md` step "Hard rules" — never delete, never send/draft,
never write email content to disk, never auto-action VIPs, never bypass
rules.yaml. The audit is read-only on Gmail except for writing to
`vips.suggested.yaml`.

---

## Step 1 — Read config

Read `inbox-watch/config.json`, `vips.yaml`, `rules.yaml`. Verify
`DISCORD_WEBHOOK_URL` env var is set. Verify Gmail MCP is available
(`mcp__claude_ai_Gmail__list_labels` succeeds).

---

## Step 2 — Score last week's pings

Query Gmail for items the watcher pinged in the last 7 days:

```
mcp__claude_ai_Gmail__search_threads
  q: label:COS/Flagged newer_than:7d
```

For each thread:
- `replied`     → does the latest message in the thread come from the user?
- `still_unread`→ does the thread still carry the `UNREAD` label?
- `archived`    → does the thread NOT carry the `INBOX` label?
- `still_inbox` → still has `INBOX` and is older than 48h

Count buckets:

- `pinged_total`              — number of threads with `COS/Flagged`
- `replied_within_24h`        → reply timestamp within 24h of flag
- `archived_no_reply`         → archived but the user never replied (likely
                                false positive)
- `still_inbox_over_48h`      → still in inbox + still unread > 48h
                                (likely missed the user's attention)

Compute precision proxy: `replied_within_24h / pinged_total`. Aim is
> 50% — below that, the threshold is too loose.

---

## Step 3 — Read the watcher's run log

Read `inbox-watch/output/YYYY-MM-DD.md` files for the past 7 days. Sum
counts across all runs:

- `runs_total`
- `unread_total`
- `pinged_total` (cross-check with Gmail count above)
- `archived_total`
- `reviewed_total`
- `errors` — any line starting with `ERROR:`

If Gmail's `pinged_total` differs from the log's `pinged_total` by more
than 1, note the discrepancy in the Discord message.

---

## Step 4 — Suggest new VIPs

Query the Sent folder:

```
mcp__claude_ai_Gmail__search_threads
  q: in:sent newer_than:30d
```

For each thread, extract the recipient(s). Group by email address; count
threads per recipient. For any address that:
- Appears in 5+ sent threads in 30 days, AND
- Is NOT already in `vips.yaml` (exact match OR matches a wildcard
  domain), AND
- Is not an obvious system address (`*@noreply*`, `*@notifications*`,
  the user's own forwarding addresses)

…append it to `inbox-watch/vips.suggested.yaml` (create the file if
missing). Format:

```yaml
# Auto-suggested VIPs based on Sent folder activity.
# the user reviews weekly; promote real VIPs into vips.yaml.

suggested:
  - email: "new.client@example.com"
    sent_count_30d: 8
    last_seen: "2026-04-25"
  - email: "partner@somewhere.io"
    sent_count_30d: 6
    last_seen: "2026-04-23"
```

If a previously-suggested address is now in `vips.yaml`, drop it from
`vips.suggested.yaml`. If a previously-suggested address has fallen
below the threshold, drop it. The file should be a current snapshot, not
a log.

Cap suggestions at 10 entries. If there are more, keep the highest
counts.

---

## Step 5 — Build the Discord message

Single message, max 2000 chars. Format:

```
[Claude] [INBOX-WATCHER WEEKLY] {Mon DD} – {Mon DD}

📊 **Last week**
• Runs: {runs_total} (errors: {errors})
• Inbox volume: {unread_total} new threads, {archived_total} auto-archived
• Pings sent: {pinged_total}

🎯 **Quality** (out of {pinged_total} pinged items)
• Replied within 24h: {replied_within_24h}  ✅ ({pct}%)
• Archived without reply: {archived_no_reply}  ⚠️
• Still in inbox > 48h: {still_inbox_over_48h}  ⚠️

{if precision < 50%: "⚠️  Ping threshold may be too loose — consider
tightening rules.yaml or removing some VIPs."}
{if precision > 80% and pinged_total > 5: "✅  Threshold looks
well-tuned."}
{if pinged_total == 0: "(quiet week — no pings sent. Watcher is alive.)"}

✨ **Suggested VIPs** (you've emailed 5+ times in 30d, not on list)
• new.client@example.com (8 sent)
• partner@somewhere.io (6 sent)
→ Added to inbox-watch/vips.suggested.yaml. Promote what you want into
  vips.yaml.

{if no suggestions: omit this section.}

{if there are still_inbox_over_48h items: list 1-3 of them with thread
links so the user can act on the misses.}

🛠 **Watcher health**: {runs_total} runs, {errors} errors. {✅|⚠️}
```

Post via curl to `$DISCORD_WEBHOOK_URL`. Same pattern as the per-30-min
prompt step 6.

---

## Step 6 — Commit & push

If `vips.suggested.yaml` changed:

```bash
git add inbox-watch/vips.suggested.yaml
git commit -m "inbox-watch: weekly audit YYYY-MM-DD"
git push origin main
```

If push fails, skip silently — next week's audit will recompute.

---

## Step 7 — Done

Print to stdout:

```
inbox-watch weekly audit complete. precision={pct}% pinged={N} suggested={M}
```

Nothing else.

---

## Failure modes

- Gmail MCP rate limit during sent-folder scan → save partial results,
  ping the user with "audit partial — see logs"
- No `output/*.md` files exist → first-week run; report only the Gmail
  state, skip the log-based counts, and note "first audit, no historical
  logs yet"
- `vips.yaml` missing → post error to Discord with link to RECOVERY.md
