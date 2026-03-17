# pdf2md-skill

A Claude Code / Cursor skill for converting any PDF into clean, structured Markdown — with equations, tables, and figures intact.

## What This Does

**pdf2md-skill** makes PDFs readable for AI agents. Instead of losing tables, mangling equations, or dropping figures, it uses LLM vision to "see" each page and reconstruct the full document as agent-friendly Markdown.

No OCR. No layout heuristics. The LLM is both the reader and the formatter.

### Key Features

- **Vision-Based Extraction** — Renders pages as images, uses LLM vision to read them. No OCR dependency. Works on scanned documents too.
- **LaTeX Equation Precision** — Distinguishes bold vectors (`\mathbf{x}`) from scalars (`x`), validates bracket nesting, preserves equation numbering.
- **Smart Table Extraction** — Handles multi-level headers, merged cells, preserves bold fidelity from the original PDF.
- **Float-Aware Placement** — Figures and tables placed after their first textual reference, not at their visual position on the page.
- **Semantic Heading Hierarchy** — Two-pass approach determines heading levels by document structure, not just font size.
- **Cross-Page Continuity** — Catches split paragraphs and dropped sentences at page boundaries.
- **Metadata Extraction** — Authors, affiliations, arXiv IDs structured in a clean blockquote header.

## Installation

### For Claude Code Users

```bash
git clone https://github.com/yuhaoliu7456/pdf2md-skill.git ~/.claude/skills/pdf2md-skill
```

Or copy manually:

```bash
mkdir -p ~/.claude/skills/pdf2md-skill
cp SKILL.md ~/.claude/skills/pdf2md-skill/
```

### For Cursor Users

```bash
git clone https://github.com/yuhaoliu7456/pdf2md-skill.git .cursor/skills/pdf2md-skill
```

Then invoke by asking your agent to convert a PDF to markdown.

## Usage

```
> "Convert this paper to markdown: ~/papers/attention-is-all-you-need.pdf"
```

```
> "Parse this PDF and extract all the equations: /path/to/document.pdf"
```

The skill will:
1. Render each page to PNG at 200 DPI
2. Send page images to the LLM using vision
3. Extract content following strict quality rules
4. Assemble into a single Markdown file with validation

Output: `<pdf_name>/content.md` next to the original PDF.

## Example Output

```markdown
# Attention Is All You Need

> **Authors:** Ashish Vaswani, Noam Shazeer, Niki Parmar, ...
> **Affiliations:** Google Brain, Google Research, University of Toronto
> **Published:** NeurIPS 2017
> **arXiv:** 1706.03762

## Abstract

The dominant sequence transduction models are based on complex recurrent or
convolutional neural networks...

## Introduction

Recurrent neural networks, long short-term memory and gated recurrent neural
networks in particular, have been firmly established as state of the art
approaches in sequence modeling...

<!-- Figure 1 placeholder -->
![Figure 1](extracted_figures/Figure_1.png)

> **Figure 1.** The Transformer — model architecture.

### Scaled Dot-Product Attention

$$
\text{Attention}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{softmax}\left(\frac{\mathbf{Q}\mathbf{K}^\top}{\sqrt{d_k}}\right)\mathbf{V}
\tag{1}
$$

> **Table 1.** Maximum path lengths, per-layer complexity and minimum number of sequential operations.

| Layer Type       | Complexity per Layer     | Sequential Ops | Max Path Length |
|-----------------|--------------------------|----------------|-----------------|
| Self-Attention  | $O(n^2 \cdot d)$         | $O(1)$         | $O(1)$          |
| Recurrent       | $O(n \cdot d^2)$         | $O(n)$         | $O(n)$          |
| Convolutional   | $O(k \cdot n \cdot d^2)$ | $O(1)$         | $O(\log_k(n))$  |
```

## How It Works

| Step | What Happens |
|------|-------------|
| 0. Confirm | Ask for PDF path, determine output location |
| 1. Render | `pdftoppm` (or PyMuPDF fallback) renders pages to PNG at 200 DPI |
| 2. Extract | LLM reads each page image and extracts content with strict rules |
| 3. Assemble | Concatenate pages, cross-check continuity at page boundaries |
| 4. Link | Match extracted figure images if available |
| 5. Output | Write final Markdown, report stats and warnings |

## Why Vision Instead of Text Extraction?

Traditional PDF parsers (PyPDF, pdfplumber, pdfminer) extract raw text streams — they break on multi-column layouts, lose table structure, and can't read equations. OCR tools add complexity and still struggle with math symbols.

Vision-based extraction lets the LLM see the page exactly as a human would. No layout heuristics, no glyph mapping tables, no external OCR service. The LLM is both the reader and the formatter.

## Architecture

| File | Purpose |
|------|---------|
| `SKILL.md` | Complete skill definition — workflow, extraction rules, quality checklist |

Single-file design. The entire skill is one well-structured `SKILL.md` (~320 lines) that guides the LLM through systematic extraction, validation, and assembly.

## Requirements

- **`pdftoppm`** (from Poppler) — Primary PDF renderer
  - macOS: `brew install poppler`
  - Ubuntu: `apt install poppler-utils`
- **Or PyMuPDF** as fallback: `pip install PyMuPDF`
- An LLM with vision capabilities (e.g., Claude)

## Credits

Created by [@yuhaoliu7456](https://github.com/yuhaoliu7456) with Claude Code.

## License

MIT — Use it, modify it, share it.
