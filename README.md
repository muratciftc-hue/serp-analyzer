# SERP Analyzer — Claude Code Skill

AI-powered SERP analysis skill for Claude Code. Analyzes top 10 search results for a target keyword, compares on-page and off-page metrics, detects content gaps, and generates a prioritized action list.

## Installation

Copy the `serp-analyzer` folder into your Claude Code skills directory:

```bash
cp -r serp-analyzer ~/.claude/skills/
```

### Requirements

- Claude Code CLI or Claude Code Desktop
- **Ahrefs MCP server** connected (for SERP data, keyword metrics, backlinks)
- WebFetch tool access (for fetching competitor page HTML)

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

## What It Analyzes

### Per Competitor (Top 10 SERP results)

| Metric | Source | Description |
|--------|--------|-------------|
| Domain Rating (DR) | Ahrefs | Site authority score 0-100 |
| Word Count | WebFetch | Total body text word count |
| H1 Count | WebFetch | Number of H1 tags |
| H2 Count | WebFetch | Number of H2 tags |
| FAQ Yes/No | WebFetch | FAQ section or FAQPage schema detected |
| Schema Type | WebFetch | JSON-LD schema types found |
| Internal Link Count | WebFetch | Same-domain links count |
| Backlinks (DR>40) | Ahrefs | Referring domains with DR >= 40 |
| Traffic | Ahrefs | Estimated monthly organic traffic |

### Keyword Metrics

| Metric | Description |
|--------|-------------|
| Search Volume | Monthly search volume |
| Keyword Difficulty | KD score 0-100 |
| CPC | Cost per click (USD) |
| Parent Topic | Broader topic and its volume |
| Brand Volume | Brand-only search volume |
| Brand + Term Volume | Brand + keyword search volume |

### Content Gap Analysis

When you provide your own URL, the skill compares your page against SERP averages and identifies gaps in:
- Word count
- Heading structure
- FAQ presence
- Schema markup
- Internal linking
- Backlink profile
- Topical coverage (H2 theme analysis)

## Output Format

The skill outputs a structured report with:
1. Keyword metrics overview
2. SERP feature detection
3. Competitor comparison table
4. Content gap analysis (if your URL provided)
5. Prioritized action list
6. Thematic coverage analysis

## Data Sources

- [Ahrefs SERP Overview API](https://ahrefs.com/api)
- [Ahrefs Keywords Explorer API](https://ahrefs.com/api)
- [Ahrefs Site Explorer API](https://ahrefs.com/api)
- WebFetch for HTML analysis

## Related Skills

- [on-page-scorer](https://github.com/muratciftc-hue/on-page-scorer) — Score individual pages across 9 SEO categories

## License

MIT
