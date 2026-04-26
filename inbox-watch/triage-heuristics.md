# Triage heuristics — what "needs attention" means

You are triaging the user's inbox. For each message, classify it as exactly
one of: **vip**, **needs-attention**, **routine-rule-match**, **cold-sales**,
or **uncertain**.

The cost of a false negative (missing something important) is high — the user
loses trust and goes back to checking email constantly. The cost of a false
positive is also high — a noisy ping erodes the same trust. **Both errors
defeat the system.** Be calibrated, not cautious-by-default.

When you genuinely can't tell: **uncertain → label `COS/Reviewed`, do not
ping.** The user will catch any miss in the weekly audit and tune the rules.

---

## VIP

Sender's `From` address matches an entry in `vips.yaml` (exact email or
wildcard domain). No further judgment needed — always classify as `vip`.
VIPs always get `COS/Flagged` and a Discord ping. They are NEVER
auto-archived, NEVER auto-labeled with anything else.

## Needs attention

Any ONE of these signals (when sender is NOT in vips.yaml) makes a message
`needs-attention`:

- **A direct question is asked of the user.** "Can you review…", "What do you
  think about…", "When can we…", "Did you see…". Questions in CC'd or bulk
  mail don't count.
- **A decision is requested.** "Let me know which option", "Need your
  sign-off", "Approve or reject".
- **A deadline or time-sensitive commitment.** "By Friday", "EOD", "before
  the meeting", explicit dates within the next 7 days.
- **the user made a commitment that's being chased.** "Following up on…",
  "Any update on…", "Circling back on…"
- **Escalation signals from real humans** (not automated): "Urgent",
  "ASAP", "Important". Marketing emails using these words don't count.
- **Novel sender on a real topic.** An unfamiliar human writing a coherent
  non-templated message — could be a new client, recruiter, opportunity.
- **Personal/family context from a known human.** Even if not on the VIP
  list, a real message from a real person about a real life thing.
- **Financial anomalies.** Unusual charge alerts, fraud warnings, security
  alerts on important accounts (bank, broker, primary email).
- **Calendar conflicts or meeting changes** where the user is a required
  attendee (not optional / FYI).
- **Wami.io customer or prospect.** Any thread carrying the `wami` label
  whose sender is not a known automated source.

## Routine rule match

Messages that:
1. Don't meet any `needs-attention` signal, AND
2. Match a rule in `rules.yaml` (sender, sender-domain, subject pattern,
   or `List-Unsubscribe` + category combo)

These get the rule's category label (`Newsletters`, `Receipts`, `Marketing`,
`Notifications`) and have `INBOX` removed (= archived). Silent — no ping.

Examples:
- Stripe receipt → `Receipts`, archive
- Substack newsletter → `Newsletters`, archive
- GitHub notification → `Notifications`, archive

## Cold sales

A specific kind of junk that can't be pattern-matched (every sender,
subject, and domain is different). Classify as `cold-sales` only when ALL
FIVE signals are present:

1. **Sender is a real human, not a system address.** First-name/last-name
   in the From field, personal-looking email address.
2. **Sender is unknown.** No prior thread history, not in vips.yaml.
3. **Templated pitch structure.** "Hi {name}, my name is X. I help
   {business type} with {pain point}…" — clear sales opener with a CTA.
4. **Service or product directly offered to the user** — consulting, design,
   marketing services, office space, software trial, leads, etc.
5. **No `List-Unsubscribe` header.** Sales CRMs usually don't include it;
   that's what distinguishes this from mass marketing (which is caught by
   the rules).

If ANY of those five signals is missing, classify as `uncertain` instead
and leave the message alone. False positives erode trust fast — err toward
leaving things alone.

**Examples that ARE cold sales (auto-archive, `Marketing`):**
- "Hi the user, my name is Danny. I help run a design agency in Cambridge…"
- "Hi the user, would you be interested in chatting if we helped you land
  4-8 contracts for your side business?"
- "Hi the user, We have simple storage, storage warehouse, compact office…"

**Examples that are NOT cold sales (uncertain or needs-attention):**
- Real business inquiries from established brands (e.g., RFQ referencing
  prior work).
- Recruiting outreach from big-name companies.
- Inbound leads from your own business forms or website.
- Acquisition offers with real PE firm context (let the user judge).

## Uncertain

Use this when you genuinely can't tell. Examples:
- Templated-looking message from an unknown domain that might be personal
  outreach.
- A thread the user is CC'd on where the ask is ambiguous.
- Anything touching money/legal/security without obvious auto-action.

Uncertain messages get labeled `COS/Reviewed` (so they're skipped on
future runs), stay in the inbox, and do NOT trigger a Discord ping. The user
will spot them during the weekly audit if they accumulate.

---

## Precedence when multiple categories fire

A single message can match more than one signal (e.g., a `wami`-labeled
thread that's also a textbook cold-sales pitch). Apply this precedence,
top wins:

1. **VIP** — anything in `vips.yaml`. Always wins. No exceptions.
2. **Cold-sales** — if all 5 cold-sales signals fire, the more-specific
   pattern wins over broader needs-attention signals (including the
   wami signal). Auto-archive to `Marketing`.
3. **Needs-attention** — any of the listed signals.
4. **Routine rule-match** — pattern matches in `rules.yaml`.
5. **Uncertain** — fallback.

Rule of thumb: more-specific classifications beat broader ones, except
VIP which always wins. If two non-VIP signals tie, prefer the one that
keeps the message in the inbox (= the more-cautious choice).

---

## Anti-patterns — things that look urgent but usually aren't

- **"URGENT" in all caps from marketing.** Tone ≠ signal — almost always junk.
- **Automated "your action is required"** from platforms the user uses daily
  (GitHub, AWS, Stripe) — usually routine maintenance notices.
- **CC'd FYIs on long threads** where the user is not the primary actor and
  the thread is already resolving itself.
- **"Following up" from cold sales** — junk, not needs-attention.
- **Calendar invites from tools the user already uses** (e.g., Google Meet
  auto-invites for events already on the user's calendar).
- **Promotional mail without an `List-Unsubscribe` header.** Some senders
  (small SaaS, course platforms) skip it. Treat as `uncertain` (label
  `COS/Reviewed`, leave in inbox), NOT as cold sales.

---

## "Why" line for the Discord ping

When you queue an item for Discord, generate a short reason — one phrase,
≤8 words — that lets the user evaluate the call at a glance. Examples:

- `matches vips.yaml: peter@example.com`
- `direct question, deadline this Friday`
- `your side business prospect, asking for pricing`
- `bank fraud alert on primary account`
- `family, real human, real topic`
- `new sender, RFQ referencing prior work`

The "why" is what makes the system tunable without re-reading prompt
source. If a ping looks wrong, the user will see why in one glance and add
the sender to vips.yaml or rules.yaml accordingly.

---

## Calibration tips

- Optimize for the user's time. He has limited attention; you're earning each
  ping.
- A Discord ping is permission to interrupt. Treat it like a tap on the
  shoulder during a meeting.
- When writing the "why" line, lead with the ask/decision/action, not the
  sender name.
- Wami.io context: `wami` label means it came in via the side business
  forwarding. These are often customer/prospect emails — weight them
  slightly higher than equivalent personal mail.
