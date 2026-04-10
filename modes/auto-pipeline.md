# Mode: auto-pipeline — Full Pipeline (Prospect → Eval → Outreach Draft → Tracker)

When the user pastes a LinkedIn or Twitter/X URL (profile or post), run the full pipeline end-to-end.

## Step 0 — Navigate and extract context (Playwright)

1. `browser_navigate` to the URL
2. `browser_snapshot` to extract:
   - name (or handle)
   - title and company (if visible)
   - the post/tweet snippet that triggered the signal (if applicable)
   - date/recency hints
3. If Playwright can’t access content (login wall, deleted post), fall back to:
   - ask user to paste the visible snippet
   - and/or WebFetch/WebSearch for minimal context

## Step 1 — Run `eval` mode (Blocks A–E)

Execute the full evaluation flow as defined in `modes/eval.md`:
- Lead Summary
- ICP Fit Analysis
- Company Research
- Outreach Email Draft (email + LinkedIn DM)
- Follow-up Sequence

## Step 2 — Save report

Save the full evaluation report to:

`reports/{###}-{name-slug}-{YYYY-MM-DD}.md`

The report **must** include `**URL:**` in the header.

## Step 3 — Write tracker addition TSV

Write one TSV line to:

`batch/tracker-additions/{num}-{name-slug}.tsv`

Format (single line, 9 tab-separated columns):

```
{num}\t{date}\t{name}\t{title} at {company}\tEvaluated\t{grade}\t✅\t[{num}](reports/{num}-{name-slug}-{date}.md)\t{signal_snippet}
```

## Step 4 — Merge tracker

Run:

```bash
node merge-tracker.mjs
```

## Step 5 — Final user checkpoint

Ask:

> "Grade is {X}. Want me to send the outreach email? I’ll find their contact info."
