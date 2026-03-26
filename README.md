# Company Job Scraper Builder

A Claude Code skill that builds resume-aware company careers scrapers and job-match pipelines. Give it a company's careers site and your resume(s), and it will scrape listings, score job fit with an LLM, and export ranked results.

## What It Does

- Discovers and scrapes a company's job listings (APIs first, then web endpoints)
- Extracts structured candidate profiles from resume files (PDF, DOCX, TXT, MD)
- Scores each job against your profile using an LLM, with evidence-based reasoning
- Exports ranked results to XLSX, CSV, JSON, or Markdown

## Usage

Install as a Claude Code skill, then ask Claude to scrape jobs from a company. For example:

```
Scrape software engineering jobs from $COMPANY careers site and rank them against my resume at ./resume.pdf
```

The skill handles:

- **Source discovery** — finds the careers API or web endpoints
- **Resume parsing** — extracts a structured profile with skills, level, and preferences
- **Caching** — caches job lists, details, and match results separately for safe reruns
- **Rate limiting** — tunes concurrency with warm-up ramp and retry-after handling
- **Scoring** — conservative matching that penalizes level mismatch and thin JDs
- **Multi-resume support** — routes jobs to the best-matching profile via classification

## Output

Results include: job title, ID, location, date posted, URL, category, score, reasons, and missing skills — sorted by score and recency.

## Configuration

The generated `.env.example` documents all tuning knobs:

| Variable | Purpose |
|---|---|
| `JOB_LIMIT` | Debug limiter for small test runs |
| `MATCH_CONCURRENCY` | Parallel LLM scoring requests |
| `MAX_DESCRIPTION_CHARS` | Truncate JDs to control token cost |
| `MATCH_MAX_RETRIES` | Retry count for failed LLM calls |
| `MATCH_RETRY_BACKOFF_SECONDS` | Backoff between retries |
| `USE_BATCH_API` | Opt-in batch mode for large runs (default: false) |