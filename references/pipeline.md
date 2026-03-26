# Resume-Aware Job Scraper Pattern

Use this reference when building or adapting a company jobs scraper.

## Recommended repo scaffold

Create or confirm:

- `.venv`
- `requirements.txt`
- `.env.example`
- `.gitignore`
- main script or package entrypoint
- `cache/`

Recommended cache layout:

```text
cache/
  jobs_list.json          # search API results (paginated list)
  job_details/            # one JSON file per job ID
  resume_profiles/        # derived candidate profiles (if using runtime extraction)
  match_results.json      # LLM match scores keyed by job ID
```

Keep job-list, details, resume profiles, and match results separate so each stage can be rerun independently. Key design points:

- If prompt/matching logic changes, only `match_results.json` needs to be cleared — scrape caches stay intact.
- The match stage should be resume-safe: load existing results on startup, skip already-scored jobs, save incrementally during the run.
- Per-job detail caches (one file per job ID) make the detail fetch stage resume-safe for partial runs.

### Concurrent detail fetching

Sequential detail page fetching (1 req/sec) is the bottleneck for large scrapes — 200 jobs takes 3+ minutes. Use a thread pool to parallelize:

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

DETAIL_FETCH_WORKERS = int(os.getenv("DETAIL_FETCH_WORKERS", "5"))

with ThreadPoolExecutor(max_workers=DETAIL_FETCH_WORKERS) as pool:
    futures = {pool.submit(fetch_detail, job_id): job_id for job_id in uncached_ids}
    for future in as_completed(futures):
        job_id = futures[future]
        # save to cache/job_details/{job_id}.json
```

5 workers typically cut fetch time from 3+ minutes to under a minute. Respect the site's rate limits — if the careers API returns 429s, reduce workers or add per-request delays.

### CLI args vs .env separation

Put **stable config** in `.env` (things that rarely change between runs):
- API keys
- Rate-limit settings (`MATCH_CONCURRENCY`, `DETAIL_FETCH_WORKERS`)
- Token cost controls (`MAX_DESCRIPTION_CHARS`)
- Retry settings (`MATCH_MAX_RETRIES`, `MATCH_RETRY_BACKOFF_SECONDS`)
- `USE_BATCH_API`

Put **per-run knobs** as CLI arguments (things users tweak each run):
- `--job-limit` (debug limiter)
- `--max-pages` (pagination cap)
- `--max-days-old` (recency filter)
- `--queries` (search terms)
- `--output` (output file path)

Users shouldn't have to edit `.env` every time they want a quick test run vs a full scrape.

## Resume handling

If resume files are provided, they are the source of truth. Extract text and derive structured profiles automatically, then cache those profiles in `cache/resume_profiles/`.

Pre-extracted constants are still useful, but only as:

- a cached artifact generated from the resume
- a manual override the user explicitly requests
- a fallback when file parsing is unavailable

Either way, each profile should include:

- `target_role_family`
- `years_experience` — total years of relevant professional experience (integer). Used to derive YoE penalties during scoring:
  - If JD requires N+2 or more years than the candidate has: subtract 2 from score
  - If JD requires N+5 or more years: subtract 4 from score
  - Include this penalty table in the scoring prompt so the LLM applies it consistently
- `level_fit` (e.g., "Senior/IC4 sweet spot, not ready for Principal/IC5+")
- `tier_1_strengths` (core differentiators, weighted heavily)
- `tier_2_strengths` (strong skills, moderate weight)
- `supporting_skills` (light weight)
- `penalty_areas` (skills the candidate does NOT have — jobs requiring these get penalized)
- `notes`

Good extracted evidence includes:

- role level and years
- backend, infra, AI-platform, data, or frontend emphasis
- languages and frameworks
- scale metrics (e.g., "8M writes/min, sub-500ms p95")
- cloud, storage, streaming, security, or workflow ownership
- explicit gaps that should lower scores

Default multi-resume behavior:

- derive one profile per resume
- **prefer classification-based routing**: if profiles map to distinct role families, classify each job by keywords and route to the matching profile — this avoids multiplying LLM calls
- fall back to scoring against all profiles only when role families overlap or classification is ambiguous
- record the category or winning profile in the output

Only switch to separate rankings per resume if the user explicitly asks for that format.

## Keyword classification

When profiles map to distinct role families (e.g., "general SDE" vs "AI platform"), classify jobs before matching to route each job to the right scoring criteria without LLM calls.

Pattern:

```python
ROLE_KEYWORDS = {
    "ai": [r'\bAI\b', r'\bmachine learning\b', r'\bLLM\b', r'\bcopilot\b', ...],
    "frontend": [r'\bReact\b', r'\bfrontend\b', r'\bCSS\b', ...],
}
```

- Check job title + description text (case-insensitive) against each keyword list
- Require 2+ matches to classify into a category (avoids false positives from incidental mentions)
- Default to the broadest category (e.g., "sde") when no keyword threshold is met
- Log the classification distribution so the user can sanity-check it

## Discovery workflow

When a company site is unfamiliar:

1. find the public search flow
2. inspect network traffic
3. locate list/search endpoint
4. locate detail endpoint
5. identify pagination and filters
6. confirm public job URL format
7. inspect response fields before coding the full pipeline

Prefer official APIs over HTML scraping when both exist.

**Gotcha — verify server-side filters.** Some APIs accept filter/sort parameters silently but ignore them. For example, a `sort_by=date` param that falls back to distance-based sorting. Always test with a small request and inspect the actual response order. If server-side filtering is unreliable, scrape all results and filter/sort client-side.

**Gotcha — sort parameters can break keyword relevance.** Some APIs (e.g., Apple's `sort=newest`) silently disable keyword matching when a sort parameter is applied, returning all jobs site-wide by date instead of keyword-filtered results. Always compare result titles between sort modes — if adding a sort param causes unrelated jobs to appear, drop the sort and sort client-side.

### Early-stop on multi-query pagination

When running multiple search queries (e.g., "Software Engineer", "AI Engineer", "ML Engineer"), later queries often return jobs already seen in earlier queries. To avoid wasting requests:

- Track all seen job IDs in a set across queries.
- For each query, count consecutive pages that yield 0 new unique jobs.
- After 2 consecutive zero-new-job pages, stop paginating that query and move to the next.
- Log the early-stop event so the user can see how many pages were skipped.

## Matching prompt pattern

Keep the prompt evidence-based and conservative.

### Prompt structure

Split into **system message** and **user message** when that improves clarity or when the provider supports caching for repeated prompt prefixes:

- **System message**: candidate profile (tier 1/2/3 strengths, penalty areas) + scoring rules + output format. This is often identical for all jobs in the same category.
- **User message**: job title, date posted, and job description text (truncated to a configurable limit, e.g., 4500 chars).

Treat prompt caching as optional. Before relying on it, verify the current provider docs for:

- model support
- minimum cacheable prompt length
- how cached tokens are counted against limits
- any required headers or API mode

### Scoring rules

- score only from explicit JD evidence
- do not assume missing stack details
- apply level penalties for title and years mismatch (e.g., subtract 2-3 points for Principal/Staff/Distinguished)
- **apply YoE penalties when the JD requires significantly more experience than the candidate has:**
  - 2+ year gap (JD asks for N+2 or more): subtract 2
  - 5+ year gap (JD asks for N+5 or more): subtract 4
  - If the JD doesn't specify YoE, skip this penalty
- apply level bonuses for the candidate's sweet spot (e.g., Senior IC4)
- penalize domains the candidate clearly lacks
- cap thin or vague JDs at a moderate score (e.g., 6) even if they sound promising
- add a small recency bonus for recently posted jobs (e.g., +0.5 for last 2 weeks)
- return strict JSON only — minified, single line, no markdown fences, no preamble

### Output format

Instruct the LLM to return minified JSON on a single line:

```json
{
  "score": 1,
  "reasons": "One short evidence-based sentence.",
  "missing_skills": "comma-separated list or none"
}
```

Keep `reasons` to one sentence (~30 words max) and `missing_skills` to at most 5 items.

### JSON parsing robustness

LLMs may wrap JSON in markdown fences or add preamble. Parse defensively:

1. Strip leading/trailing whitespace
2. Remove markdown code fences (`` ```json ... ``` ``)
3. Find the first `{` and track brace depth + string escapes to find the matching `}`
4. Parse the extracted substring as JSON
5. Clamp the score to valid range (e.g., 1-10)

### Multiple scoring tracks

If you need multiple scoring tracks, split the criteria by role family, such as:

- general backend / SDE
- AI application / platform
- data / analytics
- frontend

### Description truncation

Truncate job description text to a configurable max character limit (e.g., `MAX_DESCRIPTION_CHARS=4500`) before sending to the LLM. Truncate at a newline boundary when possible, and append a note like `[Description truncated for token efficiency.]`.

## Rate-limit tuning

Do not tune only by concurrency. Measure actual model usage first.

### Real-time mode

Recommended process:

1. start with a small run (5-20 jobs)
2. inspect average `input_tokens` and `output_tokens` per request
3. use the real model limits for the user's tier
4. set a conservative utilization target (e.g., 85%)
5. derive safe request pacing from token usage

Safe request-rate formula:

```text
safe_rpm = min(
  request_limit_rpm,
  (input_tokens_limit_per_minute * utilization_target) / avg_input_tokens_per_request
)
```

Track average input tokens with an exponential moving average (e.g., `avg = avg * 0.7 + new * 0.3`) and re-derive pacing dynamically as requests complete.

### Warm-up ramp

Don't jump to max RPM immediately. Start at a lower rate (e.g., 1/3 of target) for the first N requests (e.g., 8) and linearly interpolate to the full target:

```text
if issued_requests <= warmup_count:
  progress = (issued_requests - 1) / (warmup_count - 1)
  effective_rpm = warmup_rpm + (target_rpm - warmup_rpm) * progress
else:
  effective_rpm = target_rpm
```

This helps avoid acceleration-based throttling and burst limits. If prompt caching is available, it is an added benefit rather than an assumption.

### Retry and backoff

- On `429`, parse the `retry-after` response header and use it as a minimum backoff floor
- Use exponential backoff: `backoff = base_seconds * attempt_number`
- Take the max of the exponential backoff and the retry-after value
- Log each retry with attempt number and backoff duration

### Prompt caching interaction

If prompt caching is confirmed for the current model and prompt size, account for it after observing real usage rather than assuming it will help from the start.

### Batch API mode (opt-in, disabled by default)

Default to **real-time mode** so the user sees matches completing one by one with live progress logs. Batch mode should be opt-in (e.g., `USE_BATCH_API=true`) because it returns all results at once after a multi-minute wait with no per-job feedback.

When enabled:

- Submit all pending requests as a single batch
- Poll for completion (typically 2-10 minutes, up to 24 hours)
- Separate rate limits from real-time API — no RPM/ITPM pressure
- Typically 50% cheaper than real-time
- Fall back to real-time API for any requests that fail in the batch

### General guidance

- log per-request token usage during tuning
- only expect prompt caching to help if the model and prompt length are eligible
- save match results every N completions (e.g., every 10) for crash resilience

## Validation loop

Always do this before a full run:

1. run a small slice such as 5-20 jobs
2. confirm the parser is stable
3. inspect top matches manually
4. inspect obvious false positives
5. adjust the prompt or penalties
6. rerun the small slice
7. scale up only after the ranking looks believable

## Output expectations

For spreadsheet output, include:

- job title
- job id
- location
- work site
- date posted
- days ago
- public URL
- category
- score
- reasons
- missing skills

Sort by score descending, then by posted date descending (most recent first within same score).

For `xlsx` output, apply these formatting details:

- **Header row**: bold white text on dark fill, centered alignment
- **Freeze panes** at row 2 so headers stay visible while scrolling
- **Auto-filter** on the full data range
- **Score conditional fill**: green (score >= 8), yellow (5-7), red (<= 4)
- **URL column**: clickable hyperlinks with the Hyperlink cell style
- **Column widths**: auto-fit based on content, capped at a reasonable max (e.g., 80 chars)
- **Text wrapping**: enable wrap_text on content cells, vertical alignment top
