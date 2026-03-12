---
name: paper
description: Academic paper workflow — find, read/annotate (HTML), daily digest, cite. Use when user invokes /paper, needs paper annotation, literature search, citation generation, or daily paper discovery.
---
# /paper — Academic Paper Workflow

## Default Config
Edit this section to customize for your project.

```yaml
output_dir: "./papers"
# output_dir: "30_Research"                      # Obsidian vault users: set in project CLAUDE.md
# output_dir: "gdrive://My Drive/Papers"         # Google Drive (requires Drive MCP)
# output_dir: "onedrive://Documents/Papers"      # OneDrive (requires OneDrive MCP)

venues: [CHI, ACL, EMNLP, NAACL, NeurIPS, ICML, ICLR, EDM, LAK, AIED, ITS, CSCW, SIGIR, CIKM]

keywords:
  - cognitive load
  - adaptive learning
  - intelligent tutoring
  - conversational learning
  - LLM tutoring
  - learning analytics
  - behavioral signals
  - educational dialogue

min_citations: 5      # /paper find: filter out papers with fewer citations
daily_max: 10         # /paper digest: max papers per run

annotation_lang: zh   # zh = Chinese annotations (default) | en = English annotations
                      # Override per-command with --lang en
                      # Override in project CLAUDE.md with: paper_annotation_lang: en

openalex_email: ""    # Optional. Add your email to join OpenAlex Polite Pool (higher rate limits).
                      # e.g. openalex_email: "yourname@example.com"   No account needed.

semantic_scholar_api_key: ""  # Optional. Leave blank to skip Semantic Scholar entirely.
                               # Free key: semanticscholar.org/product/api
                               # When set, used as secondary source after OpenAlex.
```

Override any config value by adding `paper_output_dir`, `paper_venues`, `paper_keywords`, or `paper_annotation_lang` to your project's `CLAUDE.md`.

---

## Subcommands

| Command | Usage | Description |
|---------|-------|-------------|
| `find` | `/paper find "cognitive load LLM"` | Search OpenAlex, annotate top N with quality metrics |
| `read` | `/paper read /path/to/file.pdf` | Annotate a single PDF |
| `digest` | `/paper digest` | Daily new papers from arxiv (used by cron) |
| `cite` | `/paper cite [[note-name]]` | Generate APA + BibTeX citation from existing annotation |

**Flags:**
- `--context "ML课第3周"` — override project context
- `--questions "Q1:... Q2:..."` — switch to Question mode (Prompt A)
- `--questions` — interactive question mode (Claude asks you)
- `--output /path/` — override output directory for this run
- `--top N` — return N results (default: 5 for find, daily_max for digest)
- `--lang en` — override annotation language for this run
- `--source arxiv` — search only arxiv (latest preprints, no citation filter)
- `--source venues` — search only OpenAlex, exclude arxiv-only papers (citation-weighted ranking)

---

## Context Resolution (in priority order)

1. `--context` flag in command
2. Current directory's `CLAUDE.md` (auto-loaded by Claude Code)
3. No context → use Prompt B directly (zero-config, logic analysis mode)

---

## Data Sources

### Primary: OpenAlex
Free, no API key required, 10 req/s rate limit.

```
https://api.openalex.org/works?search={query}
  &select=title,authorships,publication_year,cited_by_count,primary_location,doi,abstract_inverted_index
  &per-page=20
  [&mailto={openalex_email}  ← add if configured, joins Polite Pool]
```

Fields returned: title, authors, year, `cited_by_count`, venue (`primary_location.source.display_name`), DOI, abstract.

### Secondary: Semantic Scholar (only if `semantic_scholar_api_key` is set)
```
https://api.semanticscholar.org/graph/v1/paper/search?query={topic}
  &fields=title,authors,year,abstract,citationCount,venue,externalIds&limit=20
```
Header: `x-api-key: {semantic_scholar_api_key}`

### Tertiary: arxiv (fallback + digest source)
```
https://export.arxiv.org/api/query?search_query=(cat:cs.CL+OR+cat:cs.HC+OR+cat:cs.AI+OR+cat:cs.LG)
  +AND+({keywords})&sortBy=submittedDate&sortOrder=descending&max_results=30
```
Note: arxiv papers have no citation count (too new). Label as `preprint`.

### Rate Limit Handling (applies to all sources)
If a source returns 429:
1. Check response for `Retry-After` header → wait that many seconds
2. If no header: wait 2s → retry → wait 5s → retry → wait 10s → retry
3. After 3 retries still failing → skip this source, move to next in priority order
4. Note in output which source was used / skipped

---

## Subcommand: find

1. **Parse flags:** topic query, `--top N` (default: 5), `--lang`, `--source` (default: mixed)

2. **Resolve source strategy** based on `--source`:

   | `--source` | API calls | Ranking formula |
   |------------|-----------|----------------|
   | *(default, mixed)* | Two OpenAlex calls: `sort=cited_by_count:desc` + `sort=publication_year:desc`; merge & dedupe; arxiv fallback | `0.5 × relevance_rank + 0.3 × log(cited_by_count+1) + 0.2 × recency_score` |
   | `arxiv` | arxiv API only (`sortBy=submittedDate`); no `min_citations` filter | `0.5 × relevance_rank + 0.5 × recency_score` (no citation weight) |
   | `venues` | Single OpenAlex call; **filter out** papers where venue is null or contains "arxiv" | `0.5 × relevance_rank + 0.5 × log(cited_by_count+1)` (no recency penalty) |

3. **Fetch papers:**
   - Default/venues: Call OpenAlex with `&filter=cited_by_count:>{min_citations}` (skip filter for arxiv mode)
   - Default: make **two** calls — first `&sort=cited_by_count:desc`, second `&sort=publication_year:desc`; merge results, remove duplicates by DOI/title
   - On any source failure → follow Rate Limit Handling (see Data Sources section)
   - Note which source(s) were used in output

4. **Deduplicate against existing files:**
   - Check `{output_dir}/Venues/` and `{output_dir}/arXiv/` for existing `[FirstAuthor-YYYY-slug].html`
   - If file already exists → mark paper as `[已有]`, skip HTML generation for that paper
   - Still include `[已有]` papers in summary list with their existing path

5. **Rank top N** using the formula for the active source strategy (step 2)

6. **For each new paper:** generate HTML annotation (Mode B default, Mode A if `--questions`)
   - Include **Paper Quality Bar** (see HTML Template section)
   - Annotation language follows `annotation_lang` config (or `--lang` flag)

7. **Save** to subfolder based on source:
   - OpenAlex result with a real venue → `{output_dir}/Venues/[FirstAuthor-YYYY-slug].html`
   - arxiv / preprint (no venue or arxiv-only) → `{output_dir}/arXiv/[FirstAuthor-YYYY-slug].html`
   - Create subdirectory automatically if it doesn't exist

8. **Print summary list:**
   ```
   Found N papers via {source}. Saved:
   - 30_Research/Venues/Jin-2025-llm-teachable-agent.html   [⭐ 47 citations · CHI]
   - 30_Research/arXiv/Cai-2025-intrinsic-load.html         [📄 preprint · arXiv]
   - 30_Research/Venues/Klepsch-2017-two-types-icl.html     [已有]
   ...
   ```

---

## Subcommand: read

1. Read the PDF at provided path
2. Extract: title, authors, year, venue/journal, abstract, full text
3. Resolve mode: `--questions` → Mode A; otherwise → Mode B
4. Resolve annotation language: `--lang` flag > `annotation_lang` config
5. Generate HTML annotation (see HTML Template section)
   - For `read`: Paper Quality Bar uses metadata extracted from PDF (no citation count unless found in PDF)
6. Save to `{output_dir}/[FirstAuthorLastName-YYYY-keyword].html`
7. Output: file path + brief summary of what was found

---

## Subcommand: digest

1. **Check cache:** if `{output_dir}/PaperDigests/YYYY-MM-DD-digest.html` exists today → skip fetch, output cached path
2. **Fetch arxiv:**
   - `https://export.arxiv.org/api/query?search_query=(cat:cs.CL+OR+cat:cs.HC+OR+cat:cs.AI+OR+cat:cs.LG)+AND+({keywords})&sortBy=submittedDate&sortOrder=descending&max_results=30`
3. **Filter:** keep top `daily_max` most relevant to config keywords (semantic match on title+abstract)
4. **Deduplicate:** skip arxiv IDs seen in previous 7 days' digests
5. **Annotate:** for each paper, run Mode B annotation (brief version for digest); annotation language follows config
6. **Combine:** all annotations → single HTML digest file; each paper has a compact Paper Quality Bar (preprint badge)
7. **Save:** `{output_dir}/PaperDigests/YYYY-MM-DD-digest.html`
8. **Update daily note:** append to `10_Daily/YYYY-MM-DD.md`:
   ```
   ## Paper Digest
   → [[30_Research/PaperDigests/YYYY-MM-DD-digest|Today's Paper Digest]] (N papers)
   ```

---

## Subcommand: cite

1. Parse note name from user input (e.g., `[[Klepsch-2017-annotation]]` or file path)
2. Find HTML or MD file in `{output_dir}/` or vault root
3. Extract metadata: title, authors, year, venue, DOI/URL
4. Output directly (no file saved):
   - **APA 7th:** `Author, A., & Author, B. (Year). Title. *Venue*, *vol*(issue), pages–pages. https://doi.org/...`
   - **BibTeX:**
     ```bibtex
     @article{key,
       author = {...},
       title = {...},
       journal = {...},
       year = {...},
       doi = {...}
     }
     ```

---

## HTML Annotation Template

### Paper Quality Bar (find/digest only — insert between navbar and section-nav)

A metadata strip showing provenance and quality at a glance:

```html
<div class="quality-bar">
  <span class="qb-source">{source_icon} {source}</span>
  <span class="qb-venue">{venue or "arXiv"}</span>
  <span class="qb-year">{year}</span>
  <span class="qb-citations badge-{tier}">{citation_badge} {citation_label}</span>
  <span class="qb-venue-type">{venue_type_tag}</span>
</div>
```

**Citation tier logic:**
| Tier | Condition | Badge | CSS class | Color |
|------|-----------|-------|-----------|-------|
| `high` | cited_by_count ≥ 100 | 🔥 高引 N citations | `badge-high` | `#f97316` (orange) |
| `important` | 20–99 | ⭐ N citations | `badge-important` | `#3b82f6` (blue) |
| `valid` | 5–19 | ✓ N citations | `badge-valid` | `#22c55e` (green) |
| `preprint` | < 5 or no data | 📄 Preprint | `badge-preprint` | `#9ca3af` (gray) |

**Venue type tag logic:**
- If venue matches any name in config `venues` list AND is a known top venue (CHI, NeurIPS, ACL, EMNLP, ICML, ICLR, SIGIR, CSCW) → tag: `A* 会议` (dark navy)
- If venue matches config `venues` list (other) → tag: `会议/期刊`
- If venue contains "Workshop" or "workshop" → tag: `Workshop`
- If source is arxiv / preprint → tag: `Preprint`

**Source icon:**
- OpenAlex → `🔍 OpenAlex`
- Semantic Scholar → `📚 Semantic Scholar`
- arXiv → `📄 arXiv`

**CSS for quality bar:**
```css
.quality-bar {
  background: #f0f4f8; border-bottom: 1px solid #d1d5db;
  padding: 0.4rem 2rem; display: flex; gap: 1rem; align-items: center;
  font-family: 'IBM Plex Sans', sans-serif; font-size: 0.8rem; flex-wrap: wrap;
}
.qb-source { color: #6b7280; }
.qb-venue { font-weight: 600; color: #1e3a5f; }
.qb-year { color: #6b7280; }
.badge-high      { background: #fed7aa; color: #9a3412; padding: 2px 8px; border-radius: 10px; font-weight: 600; }
.badge-important { background: #bfdbfe; color: #1e40af; padding: 2px 8px; border-radius: 10px; font-weight: 600; }
.badge-valid     { background: #bbf7d0; color: #166534; padding: 2px 8px; border-radius: 10px; font-weight: 600; }
.badge-preprint  { background: #f3f4f6; color: #6b7280; padding: 2px 8px; border-radius: 10px; }
.qb-venue-type   { background: #1e3a5f; color: white; padding: 2px 8px; border-radius: 10px; font-size: 0.72rem; }
```

---

### Mode B: Logic Analysis (Default — zero config required)

Generate a complete, self-contained HTML file.

**5-Color Highlight System:**
| Color | Class | Represents |
|-------|-------|-----------|
| 🟡 Yellow `#fef08a` | `thesis` | Core thesis / main claim |
| 🔴 Red `#fecaca` | `concept` | Key concepts / terminology |
| 🔵 Blue `#bfdbfe` | `evidence` | Empirical evidence / data |
| 🟢 Green `#bbf7d0` | `concession` | Concessions / counterargument handling |
| 🟣 Purple `#e9d5ff` | `methodology` | Methodology description |

**Document structure:**

```
1. TOP NAVBAR
   - Paper title
   - Context label (course name / project name, or "General Reading" if none)
   - Color legend: 5 colored chips with dimension labels

2. PAPER QUALITY BAR (find/digest only — omit for /paper read)
   - Source | Venue | Year | Citation badge | Venue type tag

3. STICKY SECTION NAV
   - Links: Abstract | Introduction | Related Work | Methods | Results | Discussion | Conclusion
   - Highlight active section on scroll

4. DUAL-COLUMN BODY (grid: 40% left | 60% right)
   Left column — original paper text:
     - Preserve paragraph structure
     - Highlight key phrases: <mark class="thesis">core claim</mark> etc.
     - Font: Lora serif, line-height 1.7, comfortable reading width

   Right column — annotations (one card per paragraph):
     Language follows annotation_lang config (zh or en).
     Card structure:
       [Colored left border matching most relevant highlight in this paragraph]
       ① [zh: 段落功能 / en: Paragraph Function]：...
       ② [zh: 逻辑角色 / en: Logical Role]：...
       ③ [zh: 论证技巧或潜在漏洞 / en: Rhetorical Technique or Logical Gap]：...

5. BOTTOM — Argument Structure Overview
   Language follows annotation_lang.
   zh labels: 问题 / 论点 / 证据 / 反驳处理 / 结论
   en labels: Problem / Argument / Evidence / Concession / Conclusion

   Author's core claim in one sentence
   最强论证 / Strongest argument：...
   最弱论证 / Weakest argument：...
   APA Citation (auto-formatted)
```

### Mode A: Question-Driven (triggered by --questions)

Same structure except:
- Color legend: Q1 color, Q2 color, ... (up to 6, cycling through a distinct palette)
- Left highlights: `<mark class="q1">`, `<mark class="q2">` etc.
- Right annotation cards: labeled 【Q2 核心论点】/ [Q2 Core Argument] etc., three layers:
  ① Paragraph Function ② Argument Logic ③ Which question/layer this paragraph answers
- Bottom: Q&A Worksheet (replaces Argument Structure Overview):
  One card per question — Core argument + Key evidence + Potential counter / limitation

### CSS Design System

```css
/* Fonts: import Lora + IBM Plex Sans from Google Fonts */
:root {
  --navy: #1e3a5f;
  --paper: #f9f6f0;
  --card-bg: #ffffff;
}
.navbar { background: var(--navy); color: white; font-family: 'IBM Plex Sans', sans-serif; padding: 1rem 2rem; }
body { background: var(--paper); font-family: 'Lora', serif; margin: 0; }
.section-nav { position: sticky; top: 0; background: #f0ede6; border-bottom: 1px solid #ddd; }
.content { display: grid; grid-template-columns: 40% 1fr; gap: 2rem; max-width: 1400px; margin: 0 auto; padding: 2rem; }
.annotation-card { border-left: 4px solid [dimension-color]; background: var(--card-bg); padding: 1rem; margin-bottom: 1rem; border-radius: 4px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
/* Highlights */
mark.thesis      { background: #fef08a; }
mark.concept     { background: #fecaca; }
mark.evidence    { background: #bfdbfe; }
mark.concession  { background: #bbf7d0; }
mark.methodology { background: #e9d5ff; }
/* Responsive */
@media (max-width: 768px) { .content { grid-template-columns: 1fr; } }
```

---

## Error Handling

- **PDF unreadable:** output error message with path, suggest checking permissions
- **No results from any source:** create minimal output with note "No papers found matching query — try broader keywords"
- **OpenAlex 429 (rare):** retry with backoff (2s → 5s → 10s); after 3 failures fall back to arxiv
- **Semantic Scholar 429:** retry with backoff; fall back to OpenAlex/arxiv; note in output
- **Missing output_dir:** create directory automatically before saving
- **Malformed --questions:** re-prompt user for correct format
- **Source fallback notice:** always state in output which source(s) were used, e.g. "⚠️ OpenAlex unavailable — results from arXiv only (no citation counts)"

---

## Notes for Open-Source Use

This skill is context-agnostic by design. To customize for your project:

1. Set `output_dir` in your project's `CLAUDE.md`:
   ```
   paper_output_dir: 30_Research
   ```

2. Override keywords/venues/language in your project's `CLAUDE.md`:
   ```
   paper_venues: [CHI, AIED, EDM, LAK]
   paper_keywords: [your, topic, keywords]
   paper_annotation_lang: en
   ```

3. Optionally add your email for OpenAlex Polite Pool (higher rate limits, no account needed):
   ```
   openalex_email: yourname@example.com
   ```

4. Optionally add Semantic Scholar API key as secondary source (free key, higher limits):
   ```
   semantic_scholar_api_key: your_key_here
   ```
   Get a free key at: semanticscholar.org/product/api

5. For Drive storage: configure the appropriate MCP server, then set `output_dir` to the Drive path.
