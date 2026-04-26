# Eval-runner — classify all fixtures and diff vs expected

Run before pushing any change to `triage-heuristics.md`, `rules.yaml`, or
`prompt.md`. The discipline: a fixture mismatch is a failing test. Either
fix the prompt/rules until all 30 pass, OR update the fixture (if the
expected outcome was wrong) — then re-run. Never push past red.

This prompt is invoked locally via:

```bash
cd ~/repos/scheduled-agents
claude -p "Read inbox-watch/evals/eval-prompt.md and follow it"
```

You should NOT make any Gmail API calls or post to Discord. This is a
pure-classification exercise.

---

## Steps

### 1. Load context

Read these files into context:

1. `inbox-watch/triage-heuristics.md`
2. `inbox-watch/rules.yaml`
3. `inbox-watch/vips.yaml` (or `vips.example.yaml` if real one missing —
   note the substitution in the report header)
4. `inbox-watch/evals/fixtures.yaml`
5. `inbox-watch/evals/expected.yaml`

### 2. For each fixture, classify

For each entry under `fixtures:` in `fixtures.yaml`, decide its
classification using ONLY the heuristics + rules just loaded. Output:

- `class` — one of `vip`, `needs-attention`, `routine-rule-match`,
  `cold-sales`, `uncertain`
- `ping` — true if the agent would queue it for Discord, false otherwise
- `actual_rationale` — your reasoning in one phrase

Important: be a strict reader. Do NOT use general knowledge about email
patterns — use only what's in the loaded files. The eval is testing
whether the prompt+rules+heuristics, as written, would produce the
expected behavior.

### 3. Compare to expected

For each fixture, compare your `class` and `ping` to the entry in
`expected.yaml`.

### 4. Report

Output in this format:

```
# Inbox-watch eval — {timestamp}
# vips.yaml source: {real | example fallback}

## Pass/fail summary
Total:   30
Passed:  N
Failed:  N

## Failures (if any)
Fixture 023:
  expected:  uncertain  (rationale: "templated greeting but no clear pitch")
  actual:    cold-sales (rationale: "looks like a sales pitch")
  HINT:      Cold-sales requires ALL 5 signals — fixture 023 is missing
             signal #3 (no clear sales structure / CTA).

Fixture 028:
  expected:  routine-rule-match
  actual:    needs-attention
  HINT:      "URGENT" tone is an anti-pattern, not a needs-attention signal.
             See triage-heuristics.md "Anti-patterns" section.

## Per-class breakdown
vip:                  X expected,  Y actual,  Z mismatched
needs-attention:      X expected,  Y actual,  Z mismatched
routine-rule-match:   X expected,  Y actual,  Z mismatched
cold-sales:           X expected,  Y actual,  Z mismatched
uncertain:            X expected,  Y actual,  Z mismatched
```

### 5. Exit signal

If all 30 passed: print final line `EVAL PASSED`.

If any failed: print final line `EVAL FAILED — N mismatch(es)`.

The user will read the report and decide whether to:
- Tune the prompt/rules and re-run (most common)
- Update an `expected.yaml` entry (if the original expectation was wrong)
- Add new fixtures for edge cases the failures revealed

---

## What NOT to do in this run

- Do NOT call any Gmail MCP tool
- Do NOT post to Discord
- Do NOT modify any file
- Do NOT update fixtures or expected.yaml — that's a separate human decision
- Do NOT skip fixtures because they "feel obvious" — run all 30 every time

---

## When to add a new fixture

Whenever the user reports a real misclassification during the soak period:
1. Add a new fixture to `fixtures.yaml` with the next id
2. Add the corresponding expected entry to `expected.yaml`
3. Re-run the eval — confirm the new fixture either passes (great, just
   coverage growth) or fails (now you have a regression test for the bug
   you're about to fix)
4. Tune `triage-heuristics.md` / `rules.yaml` until all fixtures pass
5. Push

Cap the suite at ~50 fixtures total. Beyond that the LLM run cost
outweighs the value.
