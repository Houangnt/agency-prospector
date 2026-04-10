# Mode: scan — Signal Scan (Lead Discovery)

Scan public signals from LinkedIn and Twitter/X to discover prospects (founders, CTOs, VCs, senior PMs) who are expressing pain points your agency can solve — then score each lead **before** adding it to `data/pipeline.md`.

## Sources to scan

### 1) LinkedIn posts (via Playwright)

For each keyword in `config/profile.yml → icp.pain_signals`, navigate to:

`linkedin.com/search/results/content/?keywords={keyword}&origin=GLOBAL_SEARCH_HEADER`

Extract per post:
- author name
- author title (if visible)
- author profile URL
- post snippet
- post date
- post URL

### 2) LinkedIn profiles (via Playwright)

For each title in `config/profile.yml → icp.target_titles`, navigate to:

`linkedin.com/search/results/people/?keywords={title}&origin=GLOBAL_SEARCH_HEADER`

Extract per profile:
- name
- title
- company
- profile URL

### 3) Twitter/X (via WebSearch)

For each keyword in `config/profile.yml → icp.pain_signals`, query:

`site:x.com "{keyword}" (founder OR CEO OR CTO OR startup)`

(Fallback: `site:twitter.com ...`)

Extract per tweet:
- author handle
- tweet snippet
- tweet URL
- date (if visible)

## Signal scoring per lead (run AI eval before adding to pipeline)

For each discovered profile/post, score 1–5 on:
- **Pain signal strength (1–5)**: how explicitly are they expressing a problem your agency solves?
- **ICP fit (1–5)**: do their title, company stage, and industry match?
- **Timing (1–5)**: last 30 days = 5, 31–90 days = 3, older = 1

Compute total: \(total = pain + icp + timing\) (max 15).

**Only add leads with total score ≥ 9/15 to `data/pipeline.md`.**

## Output format per lead added to pipeline.md

```
- [ ] {profile_url} | {name} | {title} at {company} | signal: "{post_snippet_max_80_chars}" | score: {total}/15
```

## Scan history format (`data/scan-history.tsv`)

Header:

```
url	first_seen	source	name	company	signal_score	status
```

Status values: `added`, `skipped_score`, `skipped_dup`

## Dedup rules

Deduplicate against:
- `data/scan-history.tsv`: URL exact match already seen
- `data/pipeline.md`: URL already pending or processed
- `data/leads.md`: name+company already tracked

If duplicate, write scan-history row with `skipped_dup`.

## Scan summary output

```
Signal Scan — {YYYY-MM-DD}
━━━━━━━━━━━━━━━━━━━━━━━━━━
Sources scanned: LinkedIn posts, LinkedIn profiles, Twitter/X
Profiles found: N total
Below score threshold: N skipped
Duplicates: N skipped
New leads added to pipeline: N

  + {name} | {title} at {company} | score: {X}/15
  ...

→ Run /investor-ops pipeline to evaluate and draft outreach for new leads.
```
