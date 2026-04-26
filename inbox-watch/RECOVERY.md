# RECOVERY — when the inbox-watcher goes quiet

If you haven't seen any pings AND you suspect that's wrong, work through
this in order. The whole point is that you don't have to think hard
during recovery — each step is 1-2 minutes.

---

## Symptom 1: No Discord pings for an unusually long time

### Step 1.1 — Check the weekly audit message

The Sunday 9am audit posts even on quiet weeks (it's the heartbeat). If
the most recent Sunday audit is missing entirely, the trigger is the
problem (skip to symptom 3).

If the audit posted but `errors > 0`, look at the trigger logs (see
1.3) for the actual error.

### Step 1.2 — Test the watcher manually

Open [claude.ai/code/scheduled](https://claude.ai/code/scheduled), find
the `Inbox Watcher` trigger, hit "Run now". Watch the run output.

- If it succeeds and reports `pinged=0`, the inbox really is quiet —
  not a bug.
- If it fails, the error message tells you what's broken. Most common
  causes are 1.4 / 1.5 / 1.6 below.

### Step 1.3 — Check trigger run history

Same UI → click the trigger → view recent runs. If runs are failing
silently with no Discord error, copy the failure trace and proceed to
the matching cause below.

### Step 1.4 — Gmail MCP connector expired

Symptom: `mcp__claude_ai_Gmail__list_labels` fails with auth error.

Fix: go to [claude.ai/settings/connectors](https://claude.ai/settings/connectors),
find Gmail, hit "Reconnect", complete the OAuth flow. Then "Run now" the
trigger to confirm.

### Step 1.5 — Discord webhook URL invalid

Symptom: trigger output shows `curl ... 404` or `401` against Discord.

Fix: webhook may have been rotated by Discord (channel deleted,
permissions changed). Generate a new webhook in the Discord channel
(Edit Channel → Integrations → Webhooks → New Webhook), then update the
trigger's bootstrap prompt with the new URL. Save and "Run now".

### Step 1.6 — Git push failing

Symptom: trigger output shows `git push` rejected.

Fix: usually a concurrent run pushed first. Hit "Run now" again — it'll
re-pull and proceed. If the failure persists, your fork's `main` branch
may have diverged; pull manually and resolve.

---

## Symptom 2: Pings have become noisy

### Step 2.1 — Tighten vips.yaml

Open `inbox-watch/vips.yaml`. If you find an entry that's pinging too
often (e.g., a vendor automated address you mistakenly added), remove or
narrow it (e.g., from `*@vendor.com` to a specific human).

### Step 2.2 — Add a rules.yaml auto-archive entry

For repeated false-positive senders, add a sender / domain / subject
pattern to `inbox-watch/rules.yaml` so they get silently archived
instead of pinged.

### Step 2.3 — Run the eval suite

Before pushing, run:

```bash
cd ~/repos/scheduled-agents
claude -p "Read inbox-watch/evals/eval-prompt.md and follow it"
```

If `EVAL FAILED`, check whether your tweak broke an existing fixture.
Either fix the prompt/rules differently, or — if the fixture's
expectation was actually wrong — update `evals/expected.yaml` and
re-run.

### Step 2.4 — Push

`git add` the changed files, commit with a descriptive message
(`inbox-watch: tighten vendor false positives`), push. Next 30-min run
picks up the change.

---

## Symptom 3: Trigger itself isn't firing

### Step 3.1 — Check trigger schedule

Open [claude.ai/code/scheduled](https://claude.ai/code/scheduled). If
the watcher trigger shows `disabled` or has no recent run, re-enable it.

### Step 3.2 — Check Claude Code Pro subscription

Scheduled triggers require Pro plan. If billing lapsed, all triggers
pause. Renew and the schedule resumes from the next cron tick.

### Step 3.3 — Check trigger config drift

Run history shows what bootstrap prompt was used. If it doesn't include
`DISCORD_WEBHOOK_URL=...`, fix the bootstrap. If it doesn't reference
`inbox-watch/prompt.md`, fix that.

---

## Symptom 4: Pings missed something obviously important

This is the most painful failure mode and the most important to log. Do
this:

1. Forward the missed email to yourself with subject `[FIXTURE]
   <whatever>` so you can find it later.
2. Add an entry to `inbox-watch/evals/fixtures.yaml` mimicking the
   missed email (with all PII removed — invent a fake sender + subject
   that captures the structural pattern).
3. Add the corresponding entry to `evals/expected.yaml` showing what you
   wanted (probably `class: needs-attention, ping: true`).
4. Run the eval — confirm it FAILS on your new fixture (it should).
5. Tune `triage-heuristics.md` or `vips.yaml` until the eval passes
   (and all other fixtures still pass).
6. Push.

This is how the agent gets smarter without you having to keep tuning
the same thing twice.

---

## When in doubt — escalate to a Claude Code session

Open this repo in Claude Code (interactive) and say:

> The inbox watcher hasn't pinged me in N hours. Look at the trigger
> logs and recent output/*.md files and tell me what's wrong.

Claude can read everything in the repo and the trigger output, and will
walk through the steps above with full context.
