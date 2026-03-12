[English](README.md) | [中文](README.zh.md)

# Paperwise — Academic Paper Skill for Claude Code

> Turn any paper into a **browser-readable annotated webpage** — dual-column layout, color-coded logic highlights, and per-paragraph annotation cards. Powered by OpenAlex. Zero API keys required.

![Demo](Demo-cn.png)

## What it does

| Command | Output |
|---------|--------|
| `/paper find "cognitive load LLM"` | Searches OpenAlex → generates annotated HTML for top N papers |
| `/paper read /path/to/file.pdf` | Annotates a single local PDF |
| `/paper digest` | Daily digest of new arxiv papers matching your keywords |
| `/paper cite [[Author-YYYY-note]]` | Generates APA 7th + BibTeX from an existing annotation |

Each annotation is a **self-contained HTML file** with:
- **Paper Quality Bar** — source, venue, year, citation tier badge (🔥 high · ⭐ important · ✓ valid · 📄 preprint)
- **5-color logical highlights** — thesis · concept · evidence · concession · methodology
- **Dual-column layout** — original text on the left, annotation cards on the right
- **Argument structure overview** at the bottom — full logical skeleton of the paper

## Installation

Copy `SKILL.md` into your project's skills directory:

```
your-project/
└── .agents/
    └── skills/
        └── paper/
            └── SKILL.md
```

Then run `/paper find "your topic"` in Claude Code. That's it.

## Configuration

Edit the `## Default Config` block at the top of `SKILL.md`. All fields are optional.

```yaml
output_dir: "./papers"        # where to save HTML files

venues: [CHI, ACL, NeurIPS, ...]   # venues for badge tagging
keywords: [...]                     # default keywords for /paper digest

min_citations: 5              # filter threshold for /paper find
daily_max: 10                 # max papers per /paper digest run

annotation_lang: zh           # zh = Chinese | en = English
                              # override per-command: --lang en

openalex_email: ""            # optional: join OpenAlex Polite Pool (higher rate limits)
semantic_scholar_api_key: ""  # optional secondary source
```

**Per-project override** — add to your project's `CLAUDE.md`:
```
paper_output_dir: 30_Research
paper_annotation_lang: en
```

## Usage

### Find papers

```
/paper find "retrieval augmented generation"
/paper find "transformer attention" --top 10
/paper find "diffusion models" --source venues    # OpenAlex only, peer-reviewed
/paper find "LLM agents 2025" --source arxiv      # latest preprints only
/paper find "X" --lang en                         # English annotations
```

Results are organized into subfolders automatically:
```
papers/
├── Venues/    ← OpenAlex results with a real venue
└── arXiv/     ← preprints
```

Re-running the same query skips existing files and marks them `[already exists]` in the summary.

### Read a local PDF

```
/paper read ~/Downloads/paper.pdf
/paper read paper.pdf --context "Week 3 of NLP course"
/paper read paper.pdf --questions "Q1: What is the method? Q2: How does it compare?"
```

### Daily digest

```
/paper digest
```

Fetches today's new arxiv papers matching your configured keywords. Saves to `{output_dir}/PaperDigests/YYYY-MM-DD-digest.html`.

### Generate citation

```
/paper cite [[Jin-2025-llm-teachable-agent]]
```

Outputs APA 7th + BibTeX directly. Nothing saved — just copy from the terminal.

## `--source` flag

| Flag | Strategy | Ranking | Best for |
|------|----------|---------|----------|
| *(default)* | 2× OpenAlex (by citations + by year), merged | `0.5 relevance + 0.3 log(citations) + 0.2 recency` | Balanced: classics + recent |
| `--source venues` | OpenAlex only, arxiv filtered out | `0.5 relevance + 0.5 log(citations)` | Peer-reviewed only |
| `--source arxiv` | arxiv only | `0.5 relevance + 0.5 recency` | Latest preprints |

The default mode makes **two** API calls (sorted by citations, then by year) so both highly-cited classics and recent publications appear in results.

## Annotation modes

**Mode B (default):** Logic analysis. Right-column cards break down each paragraph's function, logical role, and rhetorical technique. Works with zero configuration.

**Mode A (`--questions`):** Question-driven. Highlights and cards are organized around your specific research questions — useful for systematic literature reviews.

## Data sources

| Source | Role | API key? |
|--------|------|----------|
| **OpenAlex** | Primary | None required |
| **arxiv** | Digest + fallback | None required |
| **Semantic Scholar** | Optional secondary | Free key |

## Requirements

- Claude Code
- Network access (OpenAlex / arxiv APIs)
- No API keys required for basic use

## License

MIT © [sjqsgg](https://github.com/sjqsgg)
