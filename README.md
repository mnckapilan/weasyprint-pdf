# weasyprint-pdf

An [Agent Skill](https://agent-skills.com) for generating print-ready HTML/CSS for
[WeasyPrint](https://weasyprint.org/) and rendering it to PDF.

WeasyPrint is a from-scratch rendering engine — no JavaScript, strong paged-media support, and
gaps in some modern screen CSS. This skill teaches an agent to generate HTML targeted at
**paper, not a scrolling viewport**: `@page` rules, physical units, margin-box headers/footers,
semantic tables for layout, and embedded fonts/SVG — while avoiding the CSS that WeasyPrint
silently drops (`box-shadow`, `text-shadow`, `position: fixed`, hover/animation, etc.).

Good for invoices, reports, brochures, covers, certificates, resumes — anything paged.

## Contents

| Path | What it is |
|------|------------|
| `SKILL.md` | The skill definition (workflow, rules, rendering instructions). |
| `references/ruleset.md` | The binding HTML/CSS ruleset — allowed / use-with-caution / forbidden. |
| `assets/template.html` | A starting skeleton with an `@page` rule and a browser-preview block. |
| `examples/` | A sample invoice in Markdown, the generated HTML, and the rendered PDF. |

## Using the skill

Drop this directory into your agent's skills location (e.g. `~/.claude/skills/weasyprint-pdf/`
for Claude Code) and the agent will load it automatically when a task involves WeasyPrint or
HTML-to-PDF.

## Rendering locally

WeasyPrint needs native libraries (Pango, GObject, cairo, fontconfig):

```sh
# macOS — bare pip/uvx installs often fail to link Homebrew dylibs, so prefer brew:
brew install weasyprint

# Linux — with libpango/fontconfig present:
pipx install weasyprint   # or: uvx weasyprint document.html document.pdf
```

Then:

```sh
weasyprint document.html document.pdf
```

## Validating

This directory is itself an Agent Skill. Validate its structure and frontmatter:

```sh
uvx --from skills-ref agentskills validate ./weasyprint-pdf
```

## License

[MIT](LICENSE)
