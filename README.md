# Multi-Source Data Scraper & Trust Scoring System

A Python pipeline that scrapes structured content from **blog posts**, **YouTube videos**, and **PubMed articles**, then evaluates each source's reliability using a weighted trust scoring algorithm.

---

## Project Structure

```
project/
├── main.py                  # Entry point – runs full pipeline
├── scraper/
│   ├── blog_scraper.py      # Blog post scraper (requests + BeautifulSoup)
│   ├── youtube_scraper.py   # YouTube metadata + transcript scraper
│   └── pubmed_scraper.py    # PubMed article scraper (NCBI E-utilities API)
├── scoring/
│   └── trust_score.py       # Trust scoring algorithm
├── utils/
│   ├── tagging.py           # Automatic topic tagging (TF-IDF + domain map)
│   └── chunking.py          # Text chunking utility
└── output/
    ├── blogs.json
    ├── youtube.json
    ├── pubmed.json
    └── scraped_data.json    # Combined dataset (all 6 sources)
```

---

## Tools & Libraries

| Library | Purpose |
|---|---|
| `requests` | HTTP requests for blogs and PubMed |
| `beautifulsoup4` | HTML parsing for blog and YouTube pages |
| `langdetect` | Automatic language detection |
| `scikit-learn` | TF-IDF keyword extraction for topic tagging |
| `youtube-transcript-api` | Fetch YouTube video transcripts/captions |
| `xml.etree.ElementTree` | Parse NCBI PubMed XML responses |

---

## Scraping Approach

### Blogs (`blog_scraper.py`)
- Sends HTTP GET with a browser-like User-Agent header
- Extracts author from JSON-LD schema, `<meta>` tags, or `.author` CSS selectors
- Extracts publication date from `<time>` elements and `article:published_time` meta
- Strips navigation, ads, scripts, and footers before extracting article body
- Detects language using `langdetect` on the first 500 characters

### YouTube (`youtube_scraper.py`)
- Fetches the page HTML and extracts channel name, upload date, and description from meta tags and the embedded `ytInitialData` JSON blob
- Uses `youtube-transcript-api` to download captions where available
- Falls back gracefully if transcripts are unavailable

### PubMed (`pubmed_scraper.py`)
- Uses the NCBI E-utilities REST API (`efetch.fcgi`) with `rettype=xml`
- Parses the XML response to extract title, authors, journal, abstract, and publication year
- Retrieves MeSH (Medical Subject Headings) terms as primary topic tags
- Estimates citation count via NCBI ELink

---

## Trust Score Design

```
Trust Score = w1·author_credibility
            + w2·citation_count
            + w3·domain_authority
            + w4·recency
            + w5·medical_disclaimer_presence
            - penalties
```

| Component | Weight | Description |
|---|---|---|
| author_credibility | 0.25 | Known org affiliation, multi-author average, fallback 0.2 for unknown |
| citation_count | 0.20 | Log-normalised; PubMed citations up to 1000 → 1.0 |
| domain_authority | 0.25 | Seeded DA lookup (0–100 scale); unknown domains default to 35 |
| recency | 0.20 | Exponential decay with 2-year half-life |
| medical_disclaimer | 0.10 | Binary; PubMed always scores 1.0 |

### Penalties (subtracted after weighting)
| Trigger | Penalty |
|---|---|
| Fake/generic author name | −0.10 |
| Blog domain DA < 30 (SEO spam) | −0.12 |
| Health keywords without disclaimer | −0.08 |
| Spam signal keywords in content | −0.05 per signal (max −0.20) |
| Keyword stuffing (token density > 3%) | −0.05 |

Final score is clamped to **[0.0, 1.0]**.

---

## Edge Cases

| Scenario | Handling |
|---|---|
| Missing author | Scored as 0.2 (not zero) to avoid over-penalising anonymous technical content |
| Missing publish date | Recency score defaults to 0.3 (moderate penalty) |
| Transcript unavailable | Returns `[Transcript unavailable]` placeholder; no crash |
| Multiple authors | Author credibility is the average of individual scores |
| Non-English content | `langdetect` auto-detects; stored in `language` field |
| Long articles | `chunk_text()` splits by paragraph then sentence; hard splits at 800 chars |
| Very short chunks | Merged into the previous chunk if below 100 characters |

---

## Abuse Prevention

- **Fake authors**: Names like `admin`, `staff`, `webmaster`, or single-token strings trigger a −0.10 penalty
- **SEO spam blogs**: Domains with DA < 30 receive a −0.12 penalty
- **Misleading medical content**: Health keywords without a medical disclaimer incur −0.08
- **Spam signals**: Phrases like "miracle cure" or "buy now" penalise each occurrence by −0.05
- **Keyword stuffing**: If the most frequent content word exceeds 3% of all tokens, a −0.05 penalty is applied
- **Outdated information**: Exponential recency decay ensures old content cannot achieve high scores

---

## How to Run

### 1. Install dependencies
```bash
pip install requests beautifulsoup4 langdetect scikit-learn youtube-transcript-api
```

### 2. Run the pipeline
```bash
python main.py
```

Output files are written to `output/`.

### 3. Use individual scrapers
```python
from scraper.blog_scraper import scrape_blog
result = scrape_blog("https://example.com/article")
```

---

## Limitations

- Blog scrapers rely on common HTML conventions; highly dynamic or JavaScript-rendered sites may require Playwright/Selenium
- YouTube transcript availability depends on the video having auto-generated or manual captions
- Domain authority values are a seeded static lookup; a live DA API (e.g., Moz, Ahrefs) would give more accurate scores
- PubMed citation counts are approximate (based on NCBI ELink, not citation databases like Scopus)
- `langdetect` is probabilistic and can misidentify language on very short texts
