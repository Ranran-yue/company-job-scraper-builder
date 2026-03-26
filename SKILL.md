---
name: company-job-scraper-builder
description: Build or adapt resume-aware company careers scrapers and job-match pipelines. Use when a user wants an agent to inspect a company's jobs site or API, scrape listings and details, extract structured profiles from one or more resumes, score job fit with an LLM, tune rate limits, cache intermediate results, and export ranked results.
---

# Company Job Scraper Builder

Use this skill when the task is to create or adapt a scraper for a company's careers site and rank jobs against one or more resumes.

## Quick Start

1. Read local instructions first if they exist: `AGENTS.md`, `CLAUDE.md`, `README.md`, `PLAN.md`.
2. Identify the concrete inputs:
   - company careers source or target domain
   - role families, geography, and seniority filters
   - resume file paths (`.pdf`, `.docx`, `.txt`, `.md`)
   - desired output (`xlsx`, `csv`, `json`, `md`)
3. If some details are missing, choose reasonable defaults and state them after working.

## Defaults

- Create or reuse `.venv`, `requirements.txt`, `.env.example`, and `.gitignore`.
- Put secrets in `.env`. Never hardcode API keys.
- If resume files are provided, derive or refresh structured candidate profiles from those files before matching.
- Make the pipeline resume-safe with caches.
- Add a debug limiter such as `--job-limit` before running a full scrape.
- Prefer one main script first. Refactor only when the workflow is proven.

## Workflow

### 1. Discover the job source

- Use official APIs first when available.
- If the site is undocumented, inspect browser network traffic to find search/list and detail endpoints.
- Capture pagination, filters, public job URL shape, and fields needed for output.
- **Early-stop on multi-query pagination.** When running multiple search queries (e.g., "Software Engineer" + "AI Engineer"), later queries often overlap heavily. Track seen job IDs across queries and stop paginating a query after 2 consecutive pages with 0 new unique jobs.
- **Verify server-side filters actually work.** Some APIs accept filter/sort parameters silently but ignore them (e.g., a `sort_by=date` param that falls back to default sorting). Test with a small request and inspect the response order before relying on server-side filtering.
- **Sort parameters can also break search relevance.** Some APIs (e.g., Apple's `sort=newest`) silently disable keyword matching when a sort parameter is applied, returning all jobs site-wide by date instead of keyword-relevant results. Always compare result titles between sort modes to confirm keyword filtering is still active.

### 2. Normalize resume input

If resume files are provided, treat them as the source of truth.

Recommended default:

- Accept resume files, extract text, and derive structured profiles automatically.
- Cache derived profiles separately from scraped job data.
- If the user already has pre-extracted criteria, use them only as a cache artifact, manual override, or explicit fallback.

Either way, each profile should include:
  - target role family
  - years and level
  - tiered strengths (core differentiators weighted heavily, supporting skills lightly)
  - explicit non-matches or penalty areas (skills the candidate does NOT have)
  - notes on level fit (e.g., "Senior/IC4 sweet spot, not ready for Principal")

Default multi-resume behavior:

- Derive one structured profile per resume.
- **Prefer classification-based routing over brute-force scoring**: if profiles map to distinct role families (e.g., "SDE" vs "AI platform"), classify each job by keywords first and route it to the matching profile. This avoids multiplying LLM calls by the number of profiles.
- Fall back to scoring against every profile only when role families overlap significantly or classification is ambiguous. In that case, keep the best score and record which profile won.
- Add a `Category` or `Matched Profile` column in the output so the user can see how each job was routed/scored.
- Only switch to separate per-resume reports if the user explicitly prefers that format.

### 3. Build a resume-safe pipeline

- Cache the job list separately from job details.
- Cache per-job detail fetches so reruns do not repeat site traffic.
- **Fetch detail pages concurrently.** Use a thread pool (e.g., `ThreadPoolExecutor` with 5 workers) for detail page fetches. Sequential fetching at 1 req/sec for 200+ jobs takes 3+ minutes; 5 concurrent workers cut this to under a minute.
- Cache match results independently so prompt tuning can rerun matching without re-scraping.
- Keep generated artifacts out of git.

### 4. Match conservatively

- Score only from explicit JD evidence.
- Penalize level mismatch and domains the resume does not support.
- **Penalize years-of-experience gaps.** If the JD requires significantly more YoE than the candidate has, subtract points (e.g., −2 for 2+ year gap, −4 for 5+ year gap). Include `years_experience` in the candidate profile.
- Thin or vague JDs should not receive top scores — cap at 6 even if promising.
- Require strict machine-readable output from the LLM (minified JSON, no markdown fences).
- Keep reasons short and evidence-based (one sentence, ~30 words max).
- Truncate job descriptions to a configurable max character limit before sending to the LLM to control token costs.
- Split the prompt into a **system message** (candidate profile + scoring rules) and a **user message** (job title + date + description) when that improves clarity or enables provider-supported caching.
- Treat prompt caching as optional. Verify the current provider docs for model support, minimum cacheable length, and token-accounting behavior before designing around it.
- Parse LLM responses robustly: strip markdown code fences, then find the first complete `{...}` JSON object by tracking brace depth and string escapes. LLMs occasionally wrap JSON in fences or add preamble text.

### 5. Tune rate limits from real usage

- Start with a small run first.
- Tune for the real bottleneck: request rate, input tokens per minute, and output tokens per minute.
- Respect `retry-after` headers when the model API returns `429`. Parse the header value and use it as a minimum backoff floor.
- **Warm-up ramp**: start at a lower RPM (e.g., 1/3 of target) for the first N requests and linearly increase to the target. This helps avoid acceleration-based throttling even when caching is unavailable.
- Track average input tokens per request (use an exponential moving average) and dynamically adjust pacing: `safe_rpm = min(request_limit, (ITPM_limit * utilization) / avg_input_tokens)`.
- Only rely on prompt caching if the current docs confirm the model and prompt length are actually eligible.
- **Batch API option** (disabled by default): For large runs where latency isn't critical, offer an opt-in toggle (e.g., `USE_BATCH_API=true`) to use the model provider's batch API. Benefits: separate rate limits, typically 50% cheaper. Downside: no streaming progress — you wait minutes for all results at once. Default to real-time mode so the user sees matches completing one by one.

### 6. Export useful outputs

- Include job title, id, location, date posted, URL, category, score, reasons, and missing skills.
- When multiple resumes/profiles are used, include the matched profile identifier in the output.
- Sort by score and recency.
- Prefer spreadsheet output when the user wants manual review.

### 7. Validate before scaling up

- Run a small slice first.
- Inspect top-ranked and suspicious jobs manually against the live or cached JD.
- Tune the prompt or penalties before moving from debug mode to a full scrape.

## Deliverables

- Working scraper code
- `requirements.txt`
- `.env.example` with **stable config only** (things that rarely change between runs):
  - API key placeholder
  - `MATCH_CONCURRENCY` and provider-specific request-rate controls
  - `MAX_DESCRIPTION_CHARS` (token cost control)
  - `MATCH_MAX_RETRIES` and `MATCH_RETRY_BACKOFF_SECONDS`
  - `USE_BATCH_API` (opt-in, default false)
  - `DETAIL_FETCH_WORKERS` (concurrent detail page fetchers, default 5)
- **CLI arguments** for per-run knobs (things users tweak each run):
  - `--job-limit` (debug limiter)
  - `--max-pages` (pagination cap)
  - `--max-days-old` (recency filter)
  - Other run-specific overrides
- `.gitignore` (exclude `.env`, `.venv`, `cache/`, generated output files)
- README when the repo does not already have one
- clear debug and full-run commands

## References

- Read [references/pipeline.md](references/pipeline.md) when implementing the scraper, resume extraction flow, matching prompt, cache layout, or rate-limit tuning.
