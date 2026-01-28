# AGENTS.md

## Project Overview

This is a **LLM pricing tracker** that scrapes pricing data from OpenRouter and LiteLLM, normalizes it into a unified schema, detects price changes, and generates a static website. Key differentiator from upstream: **cache pricing support** (`cache_read_per_million`, `cache_creation_per_million`).

## Architecture & Data Pipeline

The system follows a **sequential pipeline** executed by GitHub Actions every 6 hours:

```
scrape.py → normalize.py → detect_changes.py → generate_site.py → send_alerts.py
```

### Data Flow
1. **Scrape** → Raw JSON from APIs saved to `data/current/{openrouter,litellm}.json`
2. **Normalize** → Merged into unified schema at `data/current/prices.json`
3. **Detect Changes** → Compare with `data/history/YYYY/MM/DD.json`, output to `data/changelog/`
4. **Generate Site** → Static HTML pages written to `website/`
5. **Alerts** → Discord/Slack/Email notifications via webhooks

### Key Directories
- `scripts/` - Pipeline scripts (Python 3.12+, Pydantic models)
- `data/current/` - Latest scraped/normalized data
- `data/history/` - Historical snapshots (`YYYY/MM/DD.json` structure)
- `data/changelog/` - Change logs per day + `latest.json`
- `website/` - Generated static site (Tailwind CSS, no build step)

## Development Commands

```bash
# Setup
uv sync                              # Install dependencies

# Run pipeline manually
uv run python scripts/scrape.py      # Fetch from APIs
uv run python scripts/normalize.py   # Merge to prices.json
uv run python scripts/detect_changes.py  # Compare with history
uv run python scripts/generate_site.py   # Build website

# Testing & linting
uv run pytest                        # Run tests
uv run ruff check scripts/           # Lint
uv run ruff format scripts/          # Format
```

## Code Patterns

### Pydantic Models for Data Validation
All scripts use Pydantic models for type safety. See [normalize.py](scripts/normalize.py) for schema definitions:
- `PricingInfo` - Core pricing with cache fields
- `ModelInfo` - Complete model metadata
- `PricesSchema` - Root schema for `prices.json`

### Price Conversion
- OpenRouter/LiteLLM use **per-token** pricing → multiply by `1_000_000` for per-million
- Cache pricing fields: `cache_read_per_million` (hits), `cache_creation_per_million` (writes)

### Provider Extraction
Model IDs follow `provider/model-name` convention. The `extract_provider()` function in [normalize.py](scripts/normalize.py#L130) handles edge cases.

## Adding New Data Sources

1. Add scraper function in `scrape.py` following `scrape_openrouter()` pattern
2. Add normalization logic in `normalize.py` under `normalize_{source}()`
3. Map source-specific fields to unified `PricingInfo` schema
4. Ensure cache pricing fields are extracted if available

## Website Generation

[generate_site.py](scripts/generate_site.py) generates pure HTML with inline Tailwind CSS (via CDN). No bundler or build step required. Each page is a function returning HTML string.

## Environment Variables (for alerts)

- `DISCORD_WEBHOOK_URL` - Discord notifications
- `SLACK_WEBHOOK_URL` - Slack notifications
- `BUTTONDOWN_API_KEY` - Email alerts via Buttondown

## Testing Locally

To test the full pipeline without affecting production:
```bash
# Backup existing data
cp -r data/current data/current.bak

# Run pipeline
uv run python scripts/scrape.py && \
uv run python scripts/normalize.py && \
uv run python scripts/detect_changes.py && \
uv run python scripts/generate_site.py

# Preview website
python -m http.server -d website 8000
```
