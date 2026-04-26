# User Context — your inbox

This file personalizes inbox triage. The agent reads it on every run.
Replace examples with your own context. Generic placeholders are used
here so this file is safe to commit publicly.

---

## About me

- I run side projects out of a separate business email (forwarded into
  this inbox, auto-labeled `wami` by a Gmail filter — rename the label
  in `config.json` if you use a different one)
- Personal mail comes to my main Gmail address
- I'm **trying to stop checking email constantly.** That's the whole point of
  this agent. Every Discord ping should pass the bar of "I would have
  wanted to be interrupted right now."

## What "needs attention" means for ME specifically

Higher bar than the generic `triage-heuristics.md`. Flag/ping when:

- A real human writes me a real question that has a useful answer < 1 day
  old (vs. "next week is fine")
- Someone I'm collaborating with mentions a deadline, blocker, or "waiting
  on you"
- A side-business customer or prospect reaches out (label = `wami` or
  whatever you set in `config.json`)
- Money, security, or legal: bank fraud alerts, account compromise warnings,
  contracts being countersigned
- Personal: anything from family or close friends, regardless of urgency
  signals

DO NOT flag/ping when:

- It's a calendar invite I can deal with at the next desk session
- It's a notification from a tool I use daily (GitHub, Linear, Notion, AWS,
  Stripe) where nothing is broken
- It's "for your awareness" or a CC on a thread someone else is driving
- It's a newsletter I subscribed to (these go silently to `Newsletters`
  label and out of the inbox)
- It's vendor outreach with no prior relationship (cold sales → silently
  archive to `Marketing`)

## Tone for Discord pings

- Lead with the ask/decision, not the sender. "Approval needed on Q2 budget
  (Jane, 2h)" beats "Jane sent an email about Q2 budget".
- One short sentence per "why" line. No padding, no preamble.
- If it's a side-business email (carries the `wami` label), prefix with `[wami]` so I know the context.
- If it's a known VIP, prefix with `[VIP]` so I know to drop everything.

## Calibration cadence

- Weekly audit (Sundays 9am ET) is when I'll review false positives /
  missed flags and update `vips.yaml` / `rules.yaml`.
- Don't second-guess yourself mid-week. If you're uncertain, label
  `COS/Reviewed` and let me find it during the audit.

## Things I'm explicitly OK with you doing silently

- Auto-archiving newsletters with the `Newsletters` label
- Auto-archiving receipts with the `Receipts` label
- Auto-archiving cold sales pitches that meet ALL 5 signals
- Auto-archiving marketing emails with `List-Unsubscribe` headers
- Labeling routine inbox items `COS/Reviewed` and leaving them in inbox

## Things I want you to NEVER do without asking

- Send or draft any email
- Delete anything
- Auto-archive a VIP, ever
- Change my Gmail filters or labels other than the ones in `config.json:labels`
- Touch my calendar (separate scope)
