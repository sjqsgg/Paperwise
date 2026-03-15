---
name: paper
description: Academic paper workflow — find, read/annotate (HTML), daily digest, cite. Use when user invokes /paper, needs paper annotation, literature search, citation generation, or daily paper discovery.
---
# /paper — Academic Paper Workflow

## Default Config
Edit this section to customize for your project.

```yaml
output_dir: "./papers"
# Obsidian vault users: set paper_output_dir in your project's CLAUDE.md
# Drive/OneDrive: use the appropriate MCP, then set path (e.g. "gdrive://My Drive/Papers")

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

annotation_lang: zh   # zh = Chinese annotations | en = English; override with --lang

openalex_email: ""    # Optional. Add email to join OpenAlex Polite Pool (higher rate limits).
semantic_scholar_api_key: ""  # Optional. Free key: semanticscholar.org/product/api
```

Override any config value in your project's `CLAUDE.md` using keys: `paper_output_dir`, `paper_venues`, `paper_keywords`, `paper_annotation_lang`.

---

## Subcommands

| Command | Usage | Description |
|---------|-------|-------------|
| `find` | `/paper find "cognitive load LLM"` | Search papers → Digest HTML with top N cards |
| `read` | `/paper read /path/to/file.pdf` | Annotate a single PDF → full dual-column HTML |
| `digest` | `/paper digest` | Daily new papers from arxiv (used by cron) |
| `cite` | `/paper cite [[note-name]]` | Generate APA + BibTeX citation from existing annotation |

**Flags:**
- `--context "ML课第3周"` — override project context
- `--questions "Q1:... Q2:..."` — switch to Question mode (Mode A)
- `--questions` — interactive question mode (Claude asks you)
- `--output /path/` — override output directory for this run
- `--top N` — return N results (default: 5 for find, daily_max for digest); also accepts bare number after query
- `--lang en` — override annotation language for this run
- `--source arxiv` — search only arxiv (latest preprints, no citation filter)
- `--source venues` — search only papers from config `venues` list (citation-weighted ranking)
- `--local /path/` — read local PDFs from folder (no API calls); or `--local a.pdf b.pdf` for specific files

> *Flags can be written with or without `--`. Claude accepts natural language equivalents (e.g. `top 5`, `source venues`, `local /path/`, `questions "..."`).*

---

## Context Resolution (in priority order)

1. `--context` flag in command
2. Current directory's `CLAUDE.md` (auto-loaded by Claude Code)
3. No context → use Mode B directly (zero-config, logic analysis mode)

---

## Data Sources

### Primary: OpenAlex
Free, no API key required, 10 req/s rate limit.

```
https://api.openalex.org/works?search={query}
  &select=title,authorships,publication_year,cited_by_count,primary_location,doi
  &per-page=20
  [&mailto={openalex_email}  ← add if configured]
```

**After fetching:** immediately extract and keep only — title, first author, year, cited_by_count, venue name (`primary_location.source.display_name`), DOI. Discard all other fields from the response before further processing.

### Secondary: Semantic Scholar (only if `semantic_scholar_api_key` is set)
```
https://api.semanticscholar.org/graph/v1/paper/search?query={topic}
  &fields=title,authors,year,citationCount,venue,externalIds&limit=20
```
Header: `x-api-key: {semantic_scholar_api_key}`

### Tertiary: arxiv (fallback + digest source)
```
https://export.arxiv.org/api/query?search_query=(cat:cs.CL+OR+cat:cs.HC+OR+cat:cs.AI+OR+cat:cs.LG)
  +AND+({keywords})&sortBy=submittedDate&sortOrder=descending&max_results=30
```
Note: arxiv papers have no citation count. Label as `preprint`.

### Rate Limit Handling
If a source returns 429: check `Retry-After` header → wait that many seconds. If no header: wait 2s → 5s → 10s (3 retries). After 3 failures → skip source, move to next in priority. Note which source was skipped in output.

---

## Subcommand: find

1. **Parse flags:** topic query, `--top N` (default: 5), `--lang`, `--source` (default: mixed), `--local` (path or file list)
   - `--top N` also accepts a bare number immediately after the query string (e.g. `/paper find "query" 10` → top 10)

2. **Resolve source strategy:**

   | `--source` / flag | API calls | Ranking formula |
   |-------------------|-----------|----------------|
   | *(default, mixed)* | Single OpenAlex call `sort=cited_by_count:desc`; arxiv fallback if <3 results | `0.5 × relevance_rank + 0.3 × log(cited_by_count+1) + 0.2 × recency_score` |
   | `arxiv` | arxiv API only (`sortBy=submittedDate`); no `min_citations` filter | `0.5 × relevance_rank + 0.5 × recency_score` |
   | `venues` | Single OpenAlex call; keep only papers where venue name matches any entry in config `venues` list (case-insensitive partial match on `primary_location.source.display_name`) | `0.5 × relevance_rank + 0.5 × log(cited_by_count+1)` |
   | `--local /path/` or `--local a.pdf b.pdf` | No API calls — read local PDFs only | Take first `--top N` files (alphabetical order) |

   **`--local` and `--source` are mutually exclusive.** If both are given, return an error.

3. **If `--local` flag present — execute local PDF steps instead of steps 3–6:**
   1. Scan the given folder for all `.pdf` files, or use the explicitly listed files directly
   2. If file count > `--top N`, take first N (alphabetical order)
   3. For each PDF, extract: title, authors, year, abstract (first 300 words), conclusion paragraph (last 300 words)
   4. Generate a Digest Card per PDF (same 4-sentence format: 研究问题 / 核心方法 / 关键发现 / 相关性)
   5. Citation badge: fixed as `📄 Local PDF` (no OpenAlex lookup)
   6. Card bottom link: `/paper read /absolute/path/to/file.pdf 精读` (use absolute path)
   7. Save to: `{output_dir}/FindResults/find-local-YYYY-MM-DD-HHmm-{folder-slug}.html`
   8. Print summary and exit — skip steps 4–8 below

3b. **Fetch papers (online mode):**
   - Add `&filter=cited_by_count:>{min_citations}` for default/venues mode (skip for arxiv)
   - After fetch: immediately discard all fields except title, first author, year, cited_by_count, venue name, DOI
   - On any source failure → follow Rate Limit Handling

4. **Deduplicate against existing files:**
   - Check `{output_dir}/FindResults/` and `{output_dir}/` root for `[FirstAuthor-YYYY-slug].html`
   - Mark existing papers as `[已有]`, include in summary with their path, skip card generation

5. **Rank top N** using the formula for the active source strategy

6. **Generate Digest HTML** — a single self-contained HTML file containing N paper cards.

   **Digest Card format** (one card per paper):
   ```
   [Title] — [First Author] et al., [Year] — [Venue or arXiv]
   [Citation badge]  [Source tag]
   研究问题: [1 sentence — what problem does this paper solve?]
   核心方法: [1 sentence — what approach do they use?]
   关键发现: [1 sentence — what is the main result?]
   相关性:   [1 sentence — why this matters for your research, based on CLAUDE.md context]
   [DOI link if available] · → /paper read <doi_or_path> 精读
   ```
   - Citation badge uses tier logic from Paper Quality Bar section
   - Annotation language for cards follows `annotation_lang` config (or `--lang` flag)
   - HTML structure: simple cards layout, embed `references/template.css` in `<style>`

7. **Save** Digest HTML to: `{output_dir}/FindResults/find-YYYY-MM-DD-HHmm-{query-slug}.html`
   - Create subdirectory automatically if it doesn't exist

8. **Print summary:**
   ```
   Found N papers via {source}. Digest saved:
   → 30_Research/FindResults/find-2026-03-14-1430-cognitive-load-llm.html

   Papers included:
   - Jin-2025-llm-teachable-agent        [⭐ 47 citations · CHI]
   - Cai-2025-intrinsic-load             [📄 preprint · arXiv]
   - Klepsch-2017-two-types-icl          [已有 → 30_Research/FindResults/...]
   ```

---

## Subcommand: read

1. Read the PDF at provided path
2. Extract: title, authors, year, venue/journal, abstract, full text
3. Resolve mode:
   - No `--questions` → **Mode B** (logic analysis)
   - `--questions "..."` → **Mode A (inline)**. Parse questions using these rules:
     - Strip optional `Q1:` / `Q2:` etc. prefixes (they are decorative, not required)
     - Split on: comma `,` / semicolon `;` / newline / `Q\d+` pattern boundaries
     - Extract up to 6 questions; if more are given, take first 6 and warn the user
     - Accept any of these formats:
       ```
       "Q1: 方法? Q2: 与 baseline 比较?"   ← Q-label + colon
       "方法?, 与 baseline 比较?"            ← comma-separated
       "方法?; 与 baseline 比较?"            ← semicolon-separated
       ```
   - `--questions` (no argument) → **Mode A (interactive)**:
     Output exactly: `请输入你的阅读问题（每行一个，或逗号分隔，最多6个）：`
     Wait for user input, then parse using the same rules above, then proceed with Mode A
4. Resolve annotation language: `--lang` flag > `annotation_lang` config
5. Generate full dual-column HTML annotation (see HTML Template section)
   - Paper Quality Bar: omit (no citation data from local PDF unless found in text)
6. Save to `{output_dir}/[FirstAuthorLastName-YYYY-keyword].html`
   - Write directly to path; do not ls the directory beforehand
7. Output: file path + brief summary of key arguments found

---

## Subcommand: digest

1. **Check cache:** if `{output_dir}/PaperDigests/YYYY-MM-DD-digest.html` exists today → skip fetch, output cached path
2. **Fetch arxiv:** same URL as Data Sources section above
3. **Filter:** keep top `daily_max` most relevant to config keywords (semantic match on title+abstract)
4. **Deduplicate:** skip arxiv IDs seen in previous 7 days' digests
5. **Annotate:** for each paper, run Mode B annotation (brief version); annotation language follows config
6. **Combine:** all annotations → single HTML digest file; each paper has a compact Paper Quality Bar
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
   - **APA 7th:** `Author, A., & Author, B. (Year). Title. *Venue*, *vol*(issue), pages. https://doi.org/...`
   - **BibTeX:**
     ```bibtex
     @article{key,
       author = {...}, title = {...}, journal = {...}, year = {...}, doi = {...}
     }
     ```

---

## HTML Annotation Template

### Paper Quality Bar (find/digest only — between navbar and section-nav)

Shows: `{source_icon} {source}` | `{venue}` | `{year}` | `{citation_badge}` | `{venue_type_tag}`

**Citation tiers:**
| Condition | Badge | CSS class |
|-----------|-------|-----------|
| cited_by_count ≥ 100 | 🔥 高引 N citations | `badge-high` |
| 20–99 | ⭐ N citations | `badge-important` |
| 5–19 | ✓ N citations | `badge-valid` |
| < 5 or no data | 📄 Preprint | `badge-preprint` |

**Venue type tag:** `A* 会议` (top venues: CHI/NeurIPS/ACL/EMNLP/ICML/ICLR/SIGIR/CSCW) | `会议/期刊` (other config venues) | `Workshop` | `Preprint`

**Source icon:** 🔍 OpenAlex | 📚 Semantic Scholar | 📄 arXiv

CSS: see `references/template.css` (`.quality-bar`, `.badge-*`, `.qb-*` classes).

---

### Mode B: Logic Analysis (Default)

Generate a complete, self-contained HTML file. CSS: embed `references/template.css` as `<style>` in `<head>`. Google Fonts: import Lora + IBM Plex Sans.

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
   - Context label (or "General Reading" if none)
   - Color legend: 5 colored chips with dimension labels

2. PAPER QUALITY BAR (find/digest only — omit for /paper read)

3. STICKY SECTION NAV
   - Links: Abstract | Introduction | Related Work | Methods | Results | Discussion | Conclusion
   - Highlight active section on scroll

4. DUAL-COLUMN BODY — one <div class="paragraph-group"> per paragraph
   Left column — original text (ALWAYS IN ORIGINAL LANGUAGE — never translate):
     - Copy verbatim from PDF; annotation_lang ONLY controls the right column
     - Highlight key phrases: <mark class="thesis">...</mark> etc.
   Right column — annotation cards (language follows annotation_lang):
     [Colored left border matching paragraph's dominant highlight]
     ① 段落功能 / Paragraph Function：...
     ② 逻辑角色 / Logical Role：...
     ③ 论证技巧或潜在漏洞 / Rhetorical Technique or Logical Gap：...

5. BACK-TO-TOP BUTTON
   <button id="back-to-top" title="返回顶部">↑</button>
   <script>
     const btn = document.getElementById('back-to-top');
     window.addEventListener('scroll', () => { btn.style.display = window.scrollY > 300 ? 'flex' : 'none'; });
     btn.addEventListener('click', () => window.scrollTo({ top: 0, behavior: 'smooth' }));
   </script>

6. BOTTOM — Argument Structure Overview (language follows annotation_lang)
   zh labels: 问题 / 论点 / 证据 / 反驳处理 / 结论
   en labels: Problem / Argument / Evidence / Concession / Conclusion
   - Author's core claim (1 sentence)
   - 最强论证 / Strongest argument
   - 最弱论证 / Weakest argument
   - APA Citation (auto-formatted) + copy button:
     <button class="copy-btn" data-label="复制 APA" onclick="copyText(this, '{apa_string}')">复制 APA</button>
   - BibTeX block + copy button:
     <button class="copy-btn" data-label="复制 BibTeX" onclick="copyText(this, '{bibtex_string}')">复制 BibTeX</button>

   Add this script alongside the back-to-top script:
   <script>
   function copyText(btn, text) {
     navigator.clipboard.writeText(text);
     btn.textContent = '✓ 已复制'; btn.classList.add('copied');
     setTimeout(() => { btn.textContent = btn.dataset.label; btn.classList.remove('copied'); }, 1500);
   }
   </script>
```

---

### Mode A: Question-Driven (triggered by --questions)

Same structure as Mode B, with these differences:
- Color legend: per-question colors Q1–Q6 (cycling through a distinct palette)
- Left highlights: `<mark class="q1">`, `<mark class="q2">`, etc.
- Right annotation cards: labeled `【Q2 核心论点】` / `[Q2 Core Argument]`:
  ① Paragraph Function ② Argument Logic ③ Which question/layer this paragraph answers
- Bottom: **Q&A Worksheet** (replaces Argument Structure Overview):
  One card per question — Core argument + Key evidence + Potential counter/limitation

---

### CSS

All CSS is in `references/template.css` (adjacent to this SKILL.md). When generating any HTML output, embed the file contents as a `<style>` block in `<head>`. Only load when generating HTML — `/paper cite` does not need it.

---

## Error Handling

- **PDF unreadable:** output error with path, suggest checking permissions
- **No results from any source:** output "No papers found — try broader keywords"
- **OpenAlex 429:** retry with backoff (2s→5s→10s); after 3 failures fall back to arxiv
- **Semantic Scholar 429:** retry with backoff; fall back to OpenAlex/arxiv
- **Missing output_dir:** create directory automatically before saving
- **Malformed --questions:** re-prompt user for correct format
- **Source fallback notice:** always state which source(s) were used, e.g. "⚠️ OpenAlex unavailable — results from arXiv only (no citation counts)"

---

## Notes for Open-Source Use

1. Set `output_dir` in your project's `CLAUDE.md`:
   ```
   paper_output_dir: 30_Research
   ```

2. Override keywords/venues/language:
   ```
   paper_venues: [CHI, AIED, EDM, LAK]
   paper_keywords: [your, topic, keywords]
   paper_annotation_lang: en
   ```

3. Optional — OpenAlex Polite Pool (higher rate limits, no account needed):
   ```
   openalex_email: yourname@example.com
   ```

4. Optional — Semantic Scholar secondary source (free key):
   ```
   semantic_scholar_api_key: your_key_here
   ```
