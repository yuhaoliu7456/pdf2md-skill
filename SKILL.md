---
name: pdf2-md
description: >
  Convert academic PDF papers into clean, well-structured Markdown files using
  LLM vision. Use when the user provides a PDF paper and asks to extract text,
  tables, figures, or equations into Markdown, or mentions "parse paper",
  "PDF to markdown", "extract paper content", or similar.
---

# PDF-to-Markdown: Academic Paper Conversion

Convert a PDF paper into a single Markdown file by sending pages as images to
the LLM and reconstructing the full document.

## Strategy Overview

1. Render each PDF page to a high-quality image (PNG, 200 DPI)
2. Send images to the LLM (yourself) page by page or in batches
3. Reconstruct the complete Markdown with correct reading order, headings,
   figures, tables, and LaTeX equations

## Step 0 — Confirm Input

- Ask the user for the PDF path if not already provided.
- Determine output location: default to `<pdf_stem>/content.md` next to the PDF.

## Step 1 — Render PDF Pages to Images

Use `pdftoppm` (Poppler) to render each page to PNG:

```bash
mkdir -p /tmp/pdf2md_pages
pdftoppm -png -r 200 "<pdf_path>" /tmp/pdf2md_pages/page
```

If `pdftoppm` is unavailable, use PyMuPDF as fallback:

```python
import fitz
doc = fitz.open(pdf_path)
for i, page in enumerate(doc):
    pix = page.get_pixmap(dpi=200)
    pix.save(f"/tmp/pdf2md_pages/page-{i+1:03d}.png")
doc.close()
```

Verify image count matches expected page count.

## Step 2 — Page-by-Page LLM Extraction

Read each rendered page image and extract its content. For each page, apply
these rules:

### 2.1 Reading Order

- For two-column layouts: left column first, then right column, top to bottom.
- Full-width elements (title, abstract, full-width figures) at their natural
  vertical position.

### 2.2 Heading Hierarchy

**CRITICAL: Do NOT assign heading levels by font size alone.** Font size only
provides initial hints. You MUST determine the final hierarchy by semantic
context — the parent-child nesting of sections.

Two-pass approach:

1. **First pass (per-page):** Tag each heading candidate with its visual weight
   (font size, bold, numbering pattern) and approximate level.
2. **Second pass (whole-document):** After all pages are processed, review the
   full heading list and correct levels based on these rules:

**Rules for heading level assignment:**

- `#` — Paper title only (exactly one).
- `##` — Top-level sections. Identify these by: (a) they appear in the paper's
  logical flow as major divisions (Abstract, Introduction, Related Work,
  Method, Experiments, Conclusion, Acknowledgments, References); (b) they
  typically share the same font size/weight; (c) numbered sections at the same
  depth (1, 2, 3…) are all `##`.
- `###` — Subsections that live *inside* a `##` section. Identified by
  sub-numbering (3.1, 3.2) or smaller/different font weight under a parent.
- `####` — Sub-subsections (3.1.1) or paragraph-level bold headings.

**Common pitfalls to avoid:**

- A section that visually appears on a new page after a `###` subsection may
  actually be a sibling `##`, not another `###`. Check whether it has its own
  independent numbering or is at the same font weight as other `##` headings.
- A subsection with a large bold title may look like a top-level section.
  Always check whether it is logically nested under a parent.
- If the PDF suppresses section numbering (common in AAAI, NeurIPS, etc.),
  do NOT invent numbers. Reproduce the heading text exactly as printed.

**Section numbering rule:** Only include numbers in headings if the original
PDF visibly prints them. If the PDF shows "Introduction" without a number,
write `## Introduction`, not `## 1 Introduction`.

### 2.3 Metadata & Noise Filtering

These elements must NOT appear inline within body text:

- **Header/footer noise**: page numbers, journal names, conference names,
  dates, DOIs, "Preprint" tags, arXiv identifiers, running headers.
- **Author affiliations, emails**: belong in a metadata block after the title.

Place metadata in a structured block immediately after the title:

```markdown
# Paper Title

> **Authors:** Author1, Author2, ...
> **Affiliations:** Univ A, Lab B, ...
> **Published:** Conference/Journal, Year
> **arXiv:** 2402.00341
```

### 2.4 Figures — Placement Strategy

**CRITICAL: Figures in academic PDFs are floats — their visual position on the
page often does NOT match their logical position in the text.** A figure may
appear at the top of a page but logically belongs after a paragraph in the
middle of that page.

**Placement rules (in priority order):**

1. **Find the first textual reference** to the figure (e.g., "as shown in
   Fig. 3", "see Figure 5"). Place the figure immediately after the paragraph
   that contains this first reference.
2. **If no explicit reference exists**, place the figure at the end of the
   section it visually appears in.
3. **Never place a figure between two halves of a paragraph** that continues
   across pages or columns.
4. **Never place a figure before the section it belongs to.** E.g., if Figure 1
   is referenced in Introduction, it must not appear between Abstract and
   Introduction.

For every figure:

```markdown
<!-- Figure N placeholder -->
![Figure N](figures/figure_N.png)

> **Figure N.** Caption text exactly as it appears in the paper.
```

### 2.5 Tables — Placement & Formatting

**Tables are also floats.** Apply the same placement rules as figures:
place after the paragraph containing the first textual reference.

Extract table content as Markdown tables when possible:

```markdown
> **Table N.** Caption text.

| Header1 | Header2 | Header3 |
|---------|---------|---------|
| val1    | val2    | val3    |
```

**Bold/highlight fidelity:** Only bold cell values that are explicitly bolded
in the original PDF. Do not infer "best result" bolding — if you cannot
clearly see bold formatting in the image, leave the value unbolded.

**Complex tables (multirow/multicolumn):** When a table has merged cells or
multi-level headers, ensure all rows have the same number of columns. Use
one of these strategies:
- Flatten multi-level headers into a single row with combined labels
  (e.g., "RMSE S" | "RMSE NS" | "RMSE All" instead of a merged "RMSE" header)
- If truly impossible to flatten, use a placeholder:
  `<!-- Table N: complex layout, see original PDF -->` with the caption.

### 2.6 Equations — Precision Rules

Reconstruct all math as LaTeX within Markdown.

- **Inline math**: `$E = mc^2$`
- **Display math** (numbered equations):

```markdown
$$
\mathcal{L} = -\sum_{i} y_i \log(\hat{y}_i)
\tag{1}
$$
```

**CRITICAL — Symbol precision:**

- **Bold vectors vs scalars:** `\mathbf{x}` (vector/matrix) vs `x` (scalar).
  In academic papers, bold symbols almost always denote vectors, matrices, or
  special objects (e.g., identity matrix **I**, all-ones mask **1**). Pay
  careful attention to whether the PDF renders a symbol in bold or not.
  - `$\mathbf{0}$` (zero vector) ≠ `$0$` (scalar zero)
  - `$\mathbf{I}$` (identity matrix) ≠ `$I$` (scalar/image)
  - `$\mathbf{1}$` (all-ones mask) ≠ `$1$` (scalar one)
- **Bracket/parenthesis nesting:** Carefully match `\left(` with `\right)`.
  When transcribing multi-level nested expressions (e.g., Softmax over an
  attention score), read the full expression before writing, and verify that
  each opener has a closer at the correct depth.
- **Subscript characters:** Distinguish visually similar subscripts — `_t`
  (time) vs `_l` (layer/local) vs `_1` (one). Zoom in mentally on each
  subscript and cross-check with the variable's definition in the surrounding
  text.
- **Preserve equation numbering** with `\tag{N}`.
- For multi-line equations, use `aligned` or `split` environments.
- When uncertain about a symbol, add a comment:
  `<!-- uncertain: symbol may be \xi or \zeta -->`

### 2.7 Lists, Algorithms, Code

- Bullet/numbered lists: standard Markdown.
- Algorithms: use a fenced code block with label:

````markdown
```
Algorithm 1: Training Procedure
Input: dataset D, learning rate η
...
```
````

- Code snippets: fenced code blocks with language tag.

### 2.8 References / Bibliography

Format as a numbered list:

```markdown
## References

1. Author et al. "Title." *Venue*, Year.
2. ...
```

### 2.9 Appendices

Keep appendix content with matching heading levels (`## Appendix A`, etc.).

## Step 3 — Assemble & Cross-check

Concatenate page-by-page extractions into a single Markdown file.

**Cross-page continuity check** (CRITICAL):

Many errors occur at page boundaries. For every page transition:
1. Check if a paragraph was split mid-sentence. If so, merge into one
   continuous paragraph.
2. Check if a figure/table at the top of page N+1 caused text from page N to
   be dropped. Re-read both page images and verify no sentences are missing.
3. Check if a sentence references a figure/table ("As shown in Fig. 5, …")
   but the following discussion was lost. The referencing sentence and its
   continuation must both be present.

**Heading hierarchy validation:**

After assembly, print the full heading tree and verify:
- Exactly one `#` (title)
- All `##` are true top-level sections
- No `###` appears outside of a `##` parent
- Subsections are nested under the correct parent

**Float placement validation:**

For each figure and table, confirm:
- It appears after the paragraph containing its first textual reference
- It does not break a paragraph in half
- The ordering of figures (1, 2, 3…) and tables (1, 2, 3…) is sequential

## Step 4 — Link Extracted Figures

Before writing the final file, check if pre-extracted figure images exist in
the output folder. Look for a directory named `extracted_figures/` next to the
PDF (i.e., in the same parent folder where `content.md` will be saved).

```bash
ls "<output_dir>/extracted_figures/"
```

If it exists and contains files matching the pattern `Figure_N.png` (e.g.,
`Figure_1.png`, `Figure_2.png`, …), replace each figure placeholder path:

- `![Figure 1](figures/figure_1.png)` → `![Figure 1](extracted_figures/Figure_1.png)`

Match by figure number: Figure N in the Markdown maps to
`extracted_figures/Figure_N.png`. The match is case-insensitive on the
filename (`Figure_1.png`, `figure_1.png`, `Figure_01.png` all match Figure 1).

If `extracted_figures/` does not exist or a specific figure image is missing,
keep the original placeholder path unchanged for that figure.

## Step 5 — Write Output

Save the assembled Markdown to the output path. Report:
- Total pages processed
- Number of figures (with real images linked / placeholders remaining)
- Number of tables extracted
- Number of equations reconstructed
- Any warnings (uncertain symbols, complex tables skipped, etc.)

## Quality Checklist

Before delivering the final Markdown:

- [ ] Title and metadata block are correct
- [ ] All sections present in correct order
- [ ] Heading hierarchy is semantically correct (not just font-size-based)
- [ ] Section numbers match original (or omitted if original has none)
- [ ] No page numbers, running headers, or journal footers in body text
- [ ] Every figure has a placeholder + caption, placed after first reference
- [ ] Every table has content (or placeholder) + caption, placed after first reference
- [ ] Figure/table ordering is sequential (no Figure 5 before Figure 4)
- [ ] Equations render correctly in LaTeX with correct bold/scalar distinction
- [ ] Bracket nesting in all equations is correct
- [ ] No paragraphs lost at page boundaries
- [ ] References section is complete
- [ ] Reading order follows original paper flow

## Notes

- For very long papers (>20 pages), process in batches of 5 pages to stay
  within context limits. Cross-check continuity at batch boundaries.
- If the PDF is a scanned document (no selectable text), the vision approach
  still works — the LLM reads directly from the image.
- Prefer accuracy over speed. It's better to take multiple passes than to
  produce incorrect output.
