# Evals — fixture-based regression tests for inbox-watch

This is the substitute for classical TDD. The "code" is a prompt + YAML
config; LLM classification doesn't have a function to red/green. So
instead: a fixed corpus of synthetic emails with expected classifications,
re-run before any prompt change.

## When to run

**Always before pushing changes to:**
- `inbox-watch/triage-heuristics.md`
- `inbox-watch/rules.yaml`
- `inbox-watch/prompt.md` (if you touched the classification logic)

Optional but useful: every Sunday before reviewing the weekly audit.

## How to run

From the repo root:

```bash
claude -p "Read inbox-watch/evals/eval-prompt.md and follow it"
```

The prompt classifies every fixture in `fixtures.yaml` and diffs against
`expected.yaml`. Final line is either `EVAL PASSED` or `EVAL FAILED — N
mismatch(es)`. Don't push on red.

## Discipline

- A failing fixture is a failing test. Fix the prompt, OR fix the fixture
  (if the expectation was wrong). Don't push past red.
- Add a new fixture for every real misclassification the user reports. The
  bug you fix today becomes a regression test forever.
- Cap the suite at ~50 fixtures. Beyond that the LLM run cost outweighs
  the value.

## Files

- `fixtures.yaml` — synthetic emails (no real PII)
- `expected.yaml` — expected `class` + `ping` + rationale per fixture id
- `eval-prompt.md` — the runner (Claude reads this and follows it)

## Required vips.yaml entries

The fixtures assume `vips.yaml` contains:

- `sister@example.com`
- `*@bigclient.example.com`
- `family.member@example.com`
- `fraud-alerts@yourbank.com`

If your real `vips.yaml` doesn't include these, the eval will fall back to
`vips.example.yaml` and report the substitution in the header. The user's
real `vips.yaml` won't have these synthetic addresses — that's fine; the
fixtures are designed to test the rule engine against the example file,
not against the real list.

To run the eval against your real `vips.yaml`, temporarily symlink:

```bash
mv inbox-watch/vips.yaml inbox-watch/vips.yaml.bak
ln -s vips.example.yaml inbox-watch/vips.yaml
claude -p "Read inbox-watch/evals/eval-prompt.md and follow it"
mv inbox-watch/vips.yaml.bak inbox-watch/vips.yaml
```

(There's no need to do this most of the time. The eval-prompt handles the
fallback automatically.)
