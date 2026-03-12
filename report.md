# Assignment Report: Multi-Source Scraper & Trust Scoring System

---

## 1. Scraping Strategy

The pipeline is split into three source-specific scrapers that share common utility modules.

**Blog Scraper** uses the `requests` library to download HTML and `BeautifulSoup` to navigate the parse tree. Pages are fetched with a realistic browser User-Agent to reduce the chance of bot-blocking. Noise elements (navigation bars, footers, ads, scripts) are stripped before text extraction. Author and date metadata are sourced from JSON-LD structured data embedded in the page, then from standard `<meta>` tags, then from well-known CSS selectors as fallbacks — ensuring resilience across different CMS platforms (WordPress, Ghost, Medium, custom).

**YouTube Scraper** fetches the watch page HTML and extracts the `ytInitialData` JSON blob embedded by YouTube's server-side renderer. Channel name and upload date are read from itemprop attributes and Open Graph meta tags. Captions are downloaded using `youtube-transcript-api`, which interfaces with YouTube's transcript endpoint. If captions are unavailable the field is set to a placeholder so downstream processing continues uninterrupted.

**PubMed Scraper** leverages the NCBI E-utilities REST API rather than scraping HTML, which provides a stable, structured XML response. The `efetch` endpoint is called with `rettype=xml`, and Python's built-in `xml.etree.ElementTree` parses the response. MeSH (Medical Subject Headings) terms, author lists, abstract sections, and journal metadata are all available from the XML. Citation counts are estimated via the NCBI ELink API.

All three scrapers return a uniform JSON object matching the specified schema, with `error` fields populated rather than raising exceptions when a source is unreachable.

---

## 2. Topic Tagging Method

Topic tagging is a two-phase process implemented in `utils/tagging.py`.

**Phase 1 — Domain mapping**: A hand-curated dictionary maps 14 topic domains (e.g., Machine Learning, Healthcare, Data Scraping, NLP) to lists of trigger keywords. The full text is searched for each keyword set and matching domain labels are added as tags. This ensures that well-known, high-value domains are reliably tagged regardless of how they are phrased in the source.

**Phase 2 — Keyword extraction**: Remaining tag slots (up to `max_tags=8`) are filled by TF-IDF extraction using `sklearn.feature_extraction.text.TfidfVectorizer` with 1- and 2-gram features. The document is split into pseudo-sentences to give TF-IDF a corpus to work with. If `sklearn` is unavailable, a frequency-based fallback is used instead. Common English stop words are excluded from both approaches.

PubMed records additionally include MeSH terms, which are prepended to the tag list before keyword extraction runs, giving highly authoritative domain tags for medical articles.

---

## 3. Trust Score Algorithm

The trust score is a weighted linear combination of five normalised components:

| Component | Weight | Normalisation |
|---|---|---|
| Author credibility | 0.25 | Rule-based: known org → 0.9; two-word name → 0.55; single token → 0.3; unknown → 0.2 |
| Citation count | 0.20 | Logarithmic: `log10(n+1) / log10(1001)`, ceiling at 1.0 |
| Domain authority | 0.25 | Seeded lookup (0–100 DA); normalised to [0,1]; unknown domains default to 0.35 |
| Recency | 0.20 | Exponential decay with 2-year half-life: `exp(-ln(2)/2 × age_years)` |
| Medical disclaimer | 0.10 | Binary: 1.0 if present (or PubMed), 0.0 otherwise |

After the weighted sum is calculated, up to five penalty rules can reduce the score. This design makes the score interpretable and auditable: each component can be inspected independently, and penalties are explicitly documented.

---

## 4. Edge Case Handling

| Scenario | Strategy |
|---|---|
| Missing author | Default credibility 0.2 (not 0) to avoid over-penalising anonymous but legitimate technical content |
| Missing publish date | Recency defaults to 0.3 (moderate penalty for uncertainty) |
| Transcript unavailable | Graceful fallback text; scoring and tagging still run on description |
| Multiple authors | Credibility is the arithmetic mean of individual name scores |
| Non-English content | `langdetect` identifies language; stored in `language` field; no processing changes |
| Long articles | `chunk_text()` splits by paragraph → sentence → hard split at 800 chars; merges micro-chunks below 100 chars |
| API / network failure | `try/except` in every scraper returns a structured error record instead of crashing the pipeline |

**Abuse prevention** targets four vectors: fake authors (generic name detection), SEO spam (low-DA domain penalty), misleading health content (health keywords without disclaimer), and spam/keyword-stuffed content (phrase and density detection). The combined penalty ceiling is 0.50, ensuring manipulated content still gets a score but cannot exceed 0.50 even with otherwise strong signals.
