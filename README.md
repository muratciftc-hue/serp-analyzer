# SERP Analyzer — Claude Code Skill

AI-powered SERP analysis skill for Claude Code. Analyzes top 10 search results for a target keyword, compares on-page and off-page metrics, detects content gaps, and generates a prioritized action list.

## Installation

**Option 1 — Git clone (recommended):**
```bash
git clone https://github.com/muratciftc-hue/serp-analyzer.git ~/.claude/skills/serp-analyzer
```

**Option 2 — Manual copy:**
```bash
cp -r serp-analyzer ~/.claude/skills/
```

## Usage

```
/serp-analyzer <keyword> [your-url] [country:xx]
```

### Examples

```bash
# Basic — keyword only (default country: Turkey)
/serp-analyzer 5g

# With your own URL for content gap analysis
/serp-analyzer 5g https://turkcell.com.tr/5g/

# Different country
/serp-analyzer best crm software country:us

# Full — keyword + your URL + country
/serp-analyzer misir turlari https://coraltatil.com/misir-turlari country:tr
```

## How It Fetches Pages (Dual-Fetch)

This skill uses a two-layer fetch strategy because **WebFetch strips `<script>` tags** during markdown conversion, making JSON-LD schemas invisible:

| Layer | Method | What It Gets |
|-------|--------|-------------|
| **A — Raw HTML** | `curl` + python3 | JSON-LD schema types, meta description, microdata |
| **B — Content** | WebFetch (cascade) | Word count, headings, links, FAQ text |

### Layer B Cascade (for JS-rendered or bot-protected sites)

1. **WebFetch** — default, fastest
2. **Google Cache** — `webcache.googleusercontent.com/search?q=cache:[URL]`
3. **Ahrefs Cache** — crawled content via `site-audit-page-content`
4. **Chrome MCP** — full JS rendering (requires Claude in Chrome extension)

If all methods fail for a URL, it's marked as "N/A" in the table — the analysis continues.

## What It Analyzes

### Per Competitor (Top 10 SERP results)

| Metric | Source | Description |
|--------|--------|-------------|
| Domain Rating (DR) | Ahrefs | Site authority score 0-100 |
| Word Count | Dual-fetch Layer B | Total body text word count |
| H1 Count | Dual-fetch Layer B | Number of H1 tags |
| H2 Count | Dual-fetch Layer B | Number of H2 tags |
| FAQ Yes/No | Both layers | FAQPage schema (Layer A) + FAQ text (Layer B) |
| Schema Type | Dual-fetch Layer A | JSON-LD + microdata types from raw HTML |
| Internal Link Count | Dual-fetch Layer B | Same-domain links count |
| Ref Domains | Ahrefs | Total referring domains |
| Traffic | Ahrefs | Estimated monthly organic traffic |

### Keyword Metrics (Ahrefs)

| Metric | Description |
|--------|-------------|
| Search Volume | Monthly search volume |
| Keyword Difficulty | KD score 0-100 |
| CPC | Cost per click (USD) |
| SERP Features | AI Overview, PAA, Video, Image Pack, etc. |
| Parent Topic | Broader topic and its volume |
| Brand Volume | Brand-only search volume |
| Brand + Term Volume | Brand + keyword search volume |

### Content Gap Analysis

When you provide your own URL, the skill compares your page against SERP averages:
- Word count vs SERP average
- Heading structure (H2 count)
- FAQ presence (% of competitors with FAQ)
- Schema markup comparison
- Internal linking density
- Backlink profile gap
- Topical coverage (H2 theme analysis)

## Requirements

### Required
- Claude Code CLI or Claude Code Desktop
- `python3` (for raw HTML schema parsing — pre-installed on macOS/Linux)
- `curl` (for raw HTML fetching — pre-installed on macOS/Linux)

### Recommended
- **Ahrefs MCP server** — SERP data, keyword metrics, backlinks, DR
  - Without Ahrefs: off-page columns (DR, traffic, backlinks) show "N/A"
  - On-page analysis still works via WebFetch + curl

### Optional (better site coverage)
- **Claude in Chrome extension** — JS-rendered and bot-protected sites
  - Without it: sites like Trendyol, Teknosa may show "N/A"

## Output Format

The skill outputs a structured report with:
1. Keyword metrics overview
2. SERP feature detection (AI Overview, PAA, Video, etc.)
3. Competitor comparison table (all 10 results)
4. Content gap analysis (if your URL provided)
5. Prioritized action list with targets
6. Thematic coverage analysis (H2-based topic mapping)
7. Fetch status report (which method worked per URL)

## Data Sources

- **Off-page**: Ahrefs SERP Overview, Keywords Explorer, Site Explorer APIs
- **On-page**: `curl` raw HTML (schemas, meta tags) + WebFetch (content analysis)
- **Fallback**: Google Cache, Ahrefs crawled content, Chrome MCP

## Related Skills

- [on-page-scorer](https://github.com/muratciftc-hue/on-page-scorer) — Score individual pages across 9 SEO categories

## License

MIT
