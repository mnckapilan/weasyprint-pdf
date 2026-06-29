---
name: weasyprint-pdf
description: Generate print-ready HTML/CSS for WeasyPrint and render it to PDF. Use when turning Markdown or content into a PDF, building a paged/print document (invoice, report, brochure, cover, certificate, resume), or whenever the user mentions WeasyPrint, print CSS, or HTML-to-PDF. Covers the binding HTML/CSS ruleset (paged media, @page rules, physical units, no JavaScript / box-shadow / screen-only CSS) and how to preview and render HTML to PDF with the WeasyPrint CLI.
license: MIT
compatibility: "Requires the WeasyPrint CLI plus native libs (Pango, GObject, cairo, fontconfig). macOS - brew install weasyprint. Linux - pipx install weasyprint with libpango/fontconfig. A bare uvx/pip install often fails on macOS due to dylib linking."
metadata:
  author: kapilan
  version: "1.0"
---

# WeasyPrint PDF generation

Produce a complete, static, **print-targeted** HTML document and render it to PDF with
WeasyPrint. WeasyPrint is a from-scratch rendering engine — no JavaScript, strong paged-media
support, and gaps in some modern screen CSS. Generate for paper, not a scrolling viewport.

## Workflow

1. **Read the ruleset first.** Before writing any HTML, read
   [references/ruleset.md](references/ruleset.md). It is **binding, not advisory** — it lists
   exactly which CSS is allowed, which to use with caution, and which is silently dropped.
2. **Generate the HTML** following the rules below and the ruleset detail.
3. **Preview and render to PDF** with the WeasyPrint CLI (see "Iterating" and "Rendering").

## Non-negotiable rules (full detail in references/ruleset.md)

- **Paged print, not screen.** Always emit an `@page` rule; use **physical units** (`mm`, `cm`,
  `pt`, `in`) for layout and `pt` for font sizes.
- **Self-contained, static document.** One `<style>` block; complete valid HTML before it
  reaches WeasyPrint. No client-side templating or lazy loading.
- **Semantic HTML.** Real `<table>`/`<thead>`/`<tbody>`, `<h1>`–`<h6>` (these become PDF
  bookmarks), `<ul>`/`<ol>`.
- **Layout:** prefer tables / block-with-explicit-width for anything load-bearing. Use flexbox
  and grid only for simple, local, one-dimensional arrangements.
- **Page furniture:** running headers/footers and page numbers go in `@page` margin boxes
  (`@bottom-right { content: "Page " counter(page) " of " counter(pages); }`) and `string-set`
  — never `position: fixed`/`sticky`.
- **Page breaks:** wrap each logical unit (card, invoice block, section) in a container with
  `break-inside: avoid`; use `break-after: avoid` on headings.
- **Fonts:** any font installed on this machine may be used directly by family name —
  WeasyPrint resolves system fonts via fontconfig/Pango (e.g. `"Helvetica Neue"`, `"Georgia"`,
  `"Avenir Next"`, `"Menlo"`). List what's available with `fc-list : family`. Only embed a
  `@font-face` TTF/OTF (WOFF/WOFF2 is unreliable) when the document must render on another
  machine that lacks the font, or the user supplies a brand font in `assets/`.
- **Assets:** prefer **SVG** for line art (it stays crisp as true vector). Inline fonts/images
  as data URIs only when a single fully-portable file is required.

### Never emit (silently dropped or unsupported)

`<script>` / any JavaScript / inline event handlers · `box-shadow` · `text-shadow` ·
`text-emphasis-*` · `text-underline-position` · `outline-offset` · `border-image` ·
`:hover`/`:focus`/`:active` · transitions / animations · `position: fixed`/`sticky` ·
`resize`/`cursor`/`caret-*`. When an effect maps to a forbidden property, substitute per the
mapping table in the ruleset (e.g. card depth → 1px solid light-gray border, not `box-shadow`).

## Defaults (unless the source specifies otherwise)

- A4, ~18–20mm margins.
- Footer with `counter(page) " / " counter(pages)`.
- Body ~11pt, `line-height` ~1.45. Pick any installed system font that fits the document's
  tone (always end the stack in a generic fallback, e.g. `"Helvetica Neue", sans-serif`).

Use [assets/template.html](assets/template.html) as a starting skeleton.

## Iterating: HTML preview first, PDF last

Most work should iterate on the **HTML** and only export the PDF when it looks right. Always
include the `@media screen` block from [assets/template.html](assets/template.html) in
generated HTML — it makes a browser preview look like an A4 sheet, and WeasyPrint ignores
`@media screen` so it never changes the PDF.

1. **Preview fast in a browser** for content, typography, and layout. Open the raw HTML:
   ```
   open document.html          # macOS   (Linux: xdg-open document.html)
   ```
   The user asks for changes → you edit the HTML → they refresh the browser. Quick loop.
2. **Live PDF preview** when pagination or page furniture matters — page breaks, page numbers,
   and `@page` margin-box headers/footers don't show in a browser. Render and open the PDF;
   on macOS Preview.app auto-reloads when the file changes, so just re-render after each edit:
   ```
   weasyprint document.html document.pdf && open document.pdf
   ```
3. **Export** the final PDF (see "Rendering").

**Fidelity caveat to tell the user:** a browser is approximate. Anything paged — breaks, page
counters, margin-box headers/footers, `break-inside: avoid` behaviour — must be confirmed in
the PDF (steps 2–3), not the browser.

## Rendering

Render with the WeasyPrint CLI:

```
weasyprint document.html document.pdf
```

Notes:
- WeasyPrint needs native libraries (Pango, GObject, cairo, fontconfig). If rendering fails
  with a `libgobject`/`libpango` load error, install the CLI with its system libs — on
  **macOS** run `brew install weasyprint`. A bare `uvx`/`pip install weasyprint` often fails on
  macOS because it can't link Homebrew's dylibs; prefer the `brew`-installed `weasyprint`
  binary there. On Linux, `uvx weasyprint document.html document.pdf` works when libpango/
  fontconfig are present.
- After generating the HTML, offer to render it and report where the PDF was written.

## Validating this skill

This directory is itself an Agent Skill. Validate structure/frontmatter (run from the parent
dir with an explicit path so the validator can match the folder name):

```
uvx --from skills-ref agentskills validate ./weasyprint-pdf
```
