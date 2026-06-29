# WeasyPrint HTML/CSS Generation Ruleset

Instructions for generating HTML/CSS intended **solely** for rendering to PDF via WeasyPrint.
WeasyPrint is a from-scratch Python rendering engine (not a browser). It has no JavaScript,
strong support for paged-media/print CSS, and gaps in some modern "screen" CSS. Follow these
rules to avoid generating output that silently breaks or renders incorrectly.

---

## 0. Hard constraints (never violate)

1. **No JavaScript.** WeasyPrint never executes scripts. Do not emit `<script>` tags, inline
   event handlers (`onclick`, etc.), or any design that depends on JS to render. All content
   must be fully present in the static HTML.
2. **No external network assumptions.** Prefer local/embedded assets. If referencing remote
   resources, assume they may not resolve. Inline critical CSS in a `<style>` block or link a
   local stylesheet.
3. **Self-contained document.** The HTML must be complete and valid before it reaches WeasyPrint
   — no client-side templating, hydration, or lazy loading.
4. **Target print, not screen.** Design for fixed pages (A4/Letter), not a scrolling viewport.
   Think in pages, not in infinite vertical scroll.

---

## 1. Document skeleton

Always produce a full document with an explicit `@page` rule and physical units.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <style>
    @page {
      size: A4;              /* or: Letter, A5, or "210mm 297mm", or "... landscape" */
      margin: 20mm;
    }
    body { font-family: "DejaVu Sans", sans-serif; font-size: 11pt; color: #1a1a1a; }
  </style>
</head>
<body>
  <!-- content -->
</body>
</html>
```

- Use **physical units** for layout that matters: `mm`, `cm`, `pt`, `in`. Use `pt` for font
  sizes. `px` works but is less natural for print (96px = 1in).
- Use `%` for fluid widths inside a page, but anchor the page itself with physical units.

---

## 2. Page layout & pagination (WeasyPrint's strength — use it)

These are the features WeasyPrint does *better* than browsers. Prefer them.

- **Page size / margins:** set via `@page { size: ...; margin: ...; }`.
- **Named pages** for different sections:
  ```css
  @page cover { margin: 0; }
  .cover-section { page: cover; }
  ```
- **Page breaks:**
  - Force: `break-before: page;` / `break-after: page;` (legacy `page-break-before: always;`
    also works).
  - Prevent splitting an element across pages: `break-inside: avoid;`
  - Keep headings with following content: apply `break-after: avoid;` to headings.
- **Orphans / widows:** `orphans: 3; widows: 3;` to control line breaks at page boundaries.
- **Running headers/footers** via margin boxes and `string-set`:
  ```css
  h1 { string-set: chaptertitle content(); }
  @page {
    @top-center { content: string(chaptertitle); }
    @bottom-right { content: "Page " counter(page) " of " counter(pages); }
  }
  ```
- **Page counters:** `counter(page)` and `counter(pages)` are available inside `@page` margin
  boxes. Do not try to compute page numbers in content.
- **Table of contents / cross-references:** use `target-counter(attr(href url), page)` to print
  the page number of a link target. Headings auto-generate PDF bookmarks.
- **Repeating table headers:** `thead` repeats automatically on each page a table spans. Use
  real `<thead>`/`<tbody>`.

> Avoid CSS page floats (`float: top`/`bottom` from GCPM) — support is incomplete.

---

## 3. CSS features: ALLOWED

Safe to use freely:

- **Box model:** margin, padding, border, `border-radius`, width/height, min/max, `box-sizing`.
- **Backgrounds:** `background-color`, `background-image` (incl. multiple layers, gradients via
  `linear-gradient`/`radial-gradient`), `background-size`, `background-position`, `background-clip`.
- **Typography:** `font-family`, `font-size`, `font-weight`, `font-style`, `line-height`,
  `letter-spacing`, `word-spacing`, `text-align` (incl. `justify`), `text-indent`,
  `text-transform`, `text-decoration`, `color`, `white-space`.
- **`@font-face`** with embedded custom fonts (see §6).
- **Tables:** full CSS 2.1 table model, `border-collapse`, `thead`/`tfoot` repetition.
- **Multi-column:** `column-count`, `column-gap`, `column-width`, `column-rule`.
- **Lists & counters:** `list-style`, `@counter-style`, `counter-reset`, `counter-increment`.
- **Logical properties:** start/end/block/inline variants are supported.
- **SVG:** rendered as **vectors** in the PDF (crisp at any zoom). Prefer SVG for logos,
  icons, charts, diagrams.
- **Links, bookmarks, PDF metadata, PDF/A & PDF/UA variants** (set at render time, not in HTML).

---

## 4. CSS features: USE WITH CAUTION (keep simple)

These work but are not deeply tested — keep layouts straightforward and verify output.

- **Flexbox:** simple one-dimensional rows/columns are fine. Supported: `flex-*`, `align-*`,
  `justify-*`, `order`, and the `flex`/`flex-flow` shorthands. **Avoid** deeply nested flex,
  complex wrapping, or relying on flexbox for whole-page layout. For robust column layouts,
  prefer tables or `display: block` with widths.
- **CSS Grid:** simple, explicitly-defined grids work. Avoid complex auto-placement, dense
  packing, subgrid, and intricate `grid-template-areas`. When in doubt, fall back to a table.
- **Form fields:** `appearance: auto` renders real PDF form fields, but only for text inputs,
  checkboxes, and textareas. Don't rely on styled `<select>`/radio fidelity.

**Rule of thumb:** For predictable multi-column or card layouts in a print context, a plain
HTML `<table>` or block-with-width approach is the *most* reliable. Reach for fl/grid only when
the layout is simple.

---

## 5. CSS features: FORBIDDEN (will not render)

Do **not** emit these — they are silently dropped or unsupported:

- **`box-shadow`** — not supported. For depth, use a border or a subtle background instead.
- **`text-shadow`** — not supported.
- **`text-emphasis-*`**, **`text-underline-position`** — not supported.
- **`outline-offset`** — not supported (`outline` itself is fine).
- **`resize`, `cursor`, `caret-*`, `nav-*`** — screen/interaction-only, ignored.
- **`border-image`** and related longhands — not supported; use `border` or a background image.
- **Any `:hover`/`:focus`/`:active` or transitions/animations** — there is no interaction or
  time in a PDF. Static styles only.
- **`position: fixed`/`sticky` for "floating" UI** — for repeating page furniture, use `@page`
  margin boxes (§2), not fixed positioning.

When a desired effect maps to a forbidden property, substitute:
| Want | Don't use | Use instead |
|------|-----------|-------------|
| Drop shadow / card depth | `box-shadow` | 1px solid light-gray border, or layered background |
| Glow / emphasis on text | `text-shadow` | bold weight, color, or letter-spacing |
| Decorative border | `border-image` | repeating `background-image` or solid border |
| Repeating header/footer | `position: fixed` | `@page` margin boxes + `string()` |

---

## 6. Fonts

- WeasyPrint uses **Pango/fontconfig**; fonts must be available to the system or embedded via
  `@font-face`. They are auto-embedded into the PDF.
- **Any font installed on the rendering machine may be used directly by family name** — no
  `@font-face` needed. On macOS/Linux fontconfig exposes the whole system library; run
  `fc-list : family` to list exact family names. Reach for `@font-face` only when the PDF must
  render on a machine that lacks the font, or for a supplied brand font.
- **Prefer TTF/OTF** for `@font-face` sources. **WOFF/WOFF2 support is unreliable across
  versions** — do not depend on it. Supply a `.ttf`/`.otf` URL where possible.
  ```css
  @font-face {
    font-family: "Brand Sans";
    src: url("fonts/BrandSans.ttf") format("truetype");
    font-weight: normal;
  }
  ```
- Always declare a **fallback stack** ending in a safe generic family (`sans-serif`/`serif`).
  When generating for a known local machine, a system font (e.g. `"Helvetica Neue"`,
  `"Georgia"`, `"Avenir Next"`) is fine; for portable output prefer families likely present
  everywhere (`DejaVu Sans`, `Liberation`) or embed via `@font-face`.
- Do not assume an unembedded *web* font exists on a server you don't control — verify with
  `fc-list` or embed it.

---

## 7. Images & graphics

- **Raster:** PNG, JPEG, GIF supported. Reference local paths or data URIs. For crisp print,
  supply images at ~150–300 DPI of their printed size.
- **Vector:** **prefer SVG** for anything line-based — it stays sharp as true vector in the PDF.
- Use `width`/`height` or CSS to size images explicitly; don't rely on intrinsic sizing alone
  for layout-critical images.
- Embed images as data URIs when you need a fully self-contained single HTML file.

---

## 8. Colors & print considerations

- Full color is supported: hex, `rgb()`, `rgba()`, `hsl()`, named colors, gradients.
- Set background colors explicitly; there is no "print backgrounds" toggle to worry about —
  what you style is what prints.
- For backgrounds/borders to reach the paper edge, design within the `@page` margin or use a
  zero-margin named page (e.g. full-bleed covers) — WeasyPrint has no automatic bleed/crop
  marks unless you set up the page box for it.

---

## 9. Structural best practices for the agent

1. **One `<style>` block (or one linked stylesheet).** Keep all CSS together; avoid scattering
   inline styles except for one-off values.
2. **Use semantic HTML:** real `<table>`, `<thead>`, `<h1>`–`<h6>` (drives bookmarks), `<ul>`/`<ol>`.
3. **Set explicit widths** for columns and containers; print has no flexible viewport to fill.
4. **Test page breaks mentally:** wrap each logical unit (invoice block, card, section) in a
   container with `break-inside: avoid` so it never splits awkwardly.
5. **Prefer tables/block layout over fl/grid** for anything load-bearing; reserve flexbox/grid
   for simple local arrangements.
6. **No interactivity, no animation, no JS-derived content** — ever.
7. **Validate that every referenced asset (font, image, stylesheet) is resolvable** at render
   time, ideally local or inlined.

---

## 10. Minimal reference template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <style>
    @page {
      size: A4;
      margin: 18mm 20mm;
      @bottom-right { content: "Page " counter(page) " / " counter(pages); font-size: 9pt; color: #888; }
    }
    body { font-family: "DejaVu Sans", sans-serif; font-size: 11pt; color: #1a1a1a; line-height: 1.45; }
    h1 { font-size: 20pt; margin: 0 0 4mm; break-after: avoid; }
    h2 { font-size: 14pt; margin: 8mm 0 3mm; break-after: avoid; }
    .card { border: 1px solid #ddd; border-radius: 4px; padding: 6mm; margin-bottom: 5mm; break-inside: avoid; }
    table { width: 100%; border-collapse: collapse; }
    th, td { border: 1px solid #ccc; padding: 2mm 3mm; text-align: left; }
    thead th { background: #f3f3f3; }
  </style>
</head>
<body>
  <h1>Document Title</h1>
  <div class="card">
    <h2>Section</h2>
    <p>Body content goes here.</p>
  </div>
  <table>
    <thead><tr><th>Item</th><th>Amount</th></tr></thead>
    <tbody>
      <tr><td>Example</td><td>£10.00</td></tr>
    </tbody>
  </table>
</body>
</html>
```

---

### One-line summary for the agent
> Generate a complete, static HTML document for print: physical units, an `@page` rule with
> margin-box headers/footers, semantic tables/blocks for layout, simple fl/grid only, embedded
> TTF/OTF fonts and SVG graphics — and **never** use JavaScript, `box-shadow`, `text-shadow`,
> `border-image`, hover/animation, or any screen-only/interactive CSS.