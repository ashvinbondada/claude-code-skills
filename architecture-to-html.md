---
name: architecture-to-html
description: Convert an architecture markdown document (docs/architecture/*.md) to a styled HTML file. Use when asked to render, convert, or publish an architecture doc as HTML.
---

# Architecture to HTML

## Overview

Converts architecture markdown documents at `docs/architecture/*.md` into styled HTML files saved alongside the source with `.html` extension. All markdown is converted by hand — no external libraries assumed.

**CSS and styling:** Defer entirely to the `html-document` skill. Do NOT duplicate CSS here. Read `html-document` for the full style block, spacing rules, and Mermaid handling before writing any HTML.

---

## Checklist

- [ ] Read the source `.md` file
- [ ] Read `html-document` skill for current CSS defaults
- [ ] Collect all `## Heading` section IDs for the sidebar nav
- [ ] Detect any ` ```mermaid ` blocks — include Mermaid CDN if present
- [ ] Convert all markdown to HTML (see Process)
- [ ] Apply compact layout and color annotations (see Visual Guidelines)
- [ ] Wrap in the full Output Shell with sidebar + scroll-spy
- [ ] Write to same directory, same filename, `.html` extension

---

## Process

1. **Read the markdown file** with its absolute path.

2. **Collect section headings** — scan all `## Heading` lines and derive their anchor IDs (lowercase, spaces → hyphens). These populate the sidebar nav.

3. **Convert markdown to HTML by hand:**
   - `# Heading` → `<h1>`
   - `## Heading` → `<h2 id="{{anchor}}">`  ← always add `id` for sidebar links
   - `### Heading` → `<h3>`
   - `**bold**` → `<strong>`
   - `*italic*` / `_italic_` → `<em>`
   - `` `code` `` → `<code>`
   - `[text](url)` → `<a href="url">text</a>`
   - `---` → `<hr>`
   - Bullet lists → `<ul><li>`
   - Ordered lists → `<ol><li>`
   - `> blockquote` → `<blockquote>`
   - Fenced code → `<pre><code>` (escape HTML entities)
   - Fenced mermaid → `<div class="mermaid">` (see html-document for CDN)
   - Tables → `<table><thead><tbody>`
   - Paragraphs → `<p>`

4. **Apply Visual Guidelines** below — add color spans as you write.

5. **Wrap in Output Shell** (see below) — includes sidebar, main content area, and scroll-spy script.

6. **Write** the `.html` file.

---

## Visual Guidelines

### Compact layout
- `<h2>` top margin: `28px` not `48px+`
- Skip `<hr>` dividers between sections unless truly needed
- Use `<dl><dt><dd>` for key-value content (design choices, trade-offs) instead of `<p>` blocks
- `<li>` padding: `8px 0`

### Color annotations
Define in `<style>` and apply inline with `<span>`:

```css
.highlight { color: #f5c842; }           /* key insight, conclusion */
.insight   { color: #6ec6f5; }           /* architectural observation */
.warning   { color: #f57c6e; }           /* trade-off, risk, constraint */
.emphasis  { text-decoration: underline; color: #c8f0c8; } /* the one critical point */
```

- Commandment titles → `<span class="highlight">`
- Risks / trade-offs → `<span class="warning">`
- Architectural insights → `<span class="insight">`
- Single most important line per section → `<span class="emphasis">`

Max one or two spans per section. Color is signal, not decoration.

---

## Output Shell

Every architecture HTML file uses this exact shell. Replace `{{TITLE}}`, `{{SIDEBAR_LINKS}}`, `{{BODY}}`, `{{FILENAME}}`.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{TITLE}}</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      background: #000;
      color: #fff;
      font-family: Arial, sans-serif;
      font-size: 15px;
      line-height: 1.7;
      display: flex;
      min-height: 100vh;
    }

    /* Sidebar */
    .sidebar {
      width: 200px;
      min-width: 200px;
      position: sticky;
      top: 0;
      height: 100vh;
      overflow-y: auto;
      border-right: 1px solid #111;
      padding: 40px 0;
    }
    .sidebar-title {
      font-family: "Helvetica Neue", Helvetica, sans-serif;
      font-size: 0.65rem;
      letter-spacing: 0.15em;
      text-transform: uppercase;
      color: #333;
      padding: 0 20px 16px;
      border-bottom: 1px solid #111;
      margin-bottom: 8px;
    }
    .sidebar nav a {
      display: block;
      padding: 6px 20px;
      font-size: 0.78rem;
      color: #444;
      text-decoration: none;
      border-left: 2px solid transparent;
    }
    .sidebar nav a:hover { color: #fff; border-left-color: #333; }
    .sidebar nav a.active { color: #f5c842; border-left-color: #f5c842; }

    /* Main */
    .main {
      flex: 1;
      max-width: 700px;
      padding: 60px 40px 100px;
      overflow-x: hidden;
    }

    /* Typography — from html-document skill */
    h1 { font-family: "Helvetica Neue", Helvetica, sans-serif; font-size: 2rem; font-weight: 700; margin-bottom: 4px; }
    h2 { font-family: "Helvetica Neue", Helvetica, sans-serif; font-size: 0.7rem; font-weight: 400; letter-spacing: 0.2em; text-transform: uppercase; color: #444; margin-top: 28px; margin-bottom: 14px; padding-bottom: 8px; border-bottom: 1px solid #1a1a1a; }
    h3 { font-family: "Helvetica Neue", Helvetica, sans-serif; font-size: 0.85rem; font-weight: 600; color: #fff; margin-bottom: 6px; margin-top: 16px; }
    p { color: #aaa; font-size: 0.9rem; margin-bottom: 10px; }
    ul, ol { list-style: none; padding: 0; margin: 0; }
    li { border-bottom: 1px solid #111; padding: 8px 0; color: #777; font-size: 0.875rem; }
    li:last-child { border-bottom: none; }
    a { color: #fff; text-decoration: underline; }
    strong { color: #ccc; font-weight: 600; }
    blockquote { border-left: 2px solid #222; padding-left: 12px; color: #444; font-size: 0.85rem; margin-bottom: 10px; }
    code { background: #111; color: #aaa; padding: 2px 5px; font-size: 0.85em; }
    pre { background: #0a0a0a; border: 1px solid #1a1a1a; padding: 16px; overflow-x: auto; margin-bottom: 14px; }
    table { width: 100%; border-collapse: collapse; margin-bottom: 14px; }
    th { font-family: "Helvetica Neue", Helvetica, sans-serif; font-size: 0.7rem; text-transform: uppercase; letter-spacing: 0.1em; color: #444; padding: 6px 0; border-bottom: 1px solid #1a1a1a; text-align: left; }
    td { padding: 8px 0; border-bottom: 1px solid #111; color: #666; font-size: 0.875rem; }
    dl { margin: 4px 0; }
    dt { color: #555; font-size: 0.75rem; text-transform: uppercase; letter-spacing: 0.08em; display: inline; }
    dd { display: inline; color: #777; font-size: 0.85rem; margin-left: 6px; }
    dd::after { content: ''; display: block; }

    .mermaid { background: #0a0a0a; border: 1px solid #1a1a1a; padding: 1.5rem; margin: 16px 0; text-align: center; }
    .kicker { color: #333; font-size: 0.75rem; letter-spacing: 0.12em; text-transform: uppercase; margin-bottom: 36px; }
    .meta { margin-top: 60px; font-size: 0.75rem; color: #222; letter-spacing: 0.08em; text-transform: uppercase; }

    /* Color annotations */
    .highlight { color: #f5c842; }
    .insight   { color: #6ec6f5; }
    .warning   { color: #f57c6e; }
    .emphasis  { text-decoration: underline; color: #c8f0c8; }
  </style>
</head>
<body>

  <aside class="sidebar">
    <div class="sidebar-title">{{TITLE}}</div>
    <nav>
      {{SIDEBAR_LINKS}}
      <!-- One <a href="#section-id">Section Name</a> per ## heading -->
    </nav>
  </aside>

  <main class="main">
    {{BODY}}
    <div class="meta">docs/architecture/{{FILENAME}}.html</div>
  </main>

  <!-- Mermaid — only if doc has diagrams -->
  <script type="module">
    import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
    mermaid.initialize({ startOnLoad: true, theme: 'dark' });
  </script>

  <!-- Scroll-spy: highlights active sidebar link as user scrolls -->
  <script>
    const sections = document.querySelectorAll('h2[id]');
    const navLinks = document.querySelectorAll('.sidebar nav a');
    const observer = new IntersectionObserver(entries => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          navLinks.forEach(a => a.classList.remove('active'));
          const active = document.querySelector(`.sidebar nav a[href="#${entry.target.id}"]`);
          if (active) active.classList.add('active');
        }
      });
    }, { rootMargin: '-20% 0px -70% 0px' });
    sections.forEach(s => observer.observe(s));
  </script>

</body>
</html>
```

### Sidebar link generation

For each `## Heading` in the document, generate:
- `id` on the `<h2>`: lowercase heading text, spaces → hyphens, strip punctuation
- A nav link: `<a href="#the-id">Heading Text</a>`

Example: `## Design Choices & Trade-offs` → `id="design-choices-trade-offs"` → `<a href="#design-choices-trade-offs">Design Choices & Trade-offs</a>`

### Notes
- `{{TITLE}}` — first `<h1>` text, or filename if none
- `{{SIDEBAR_LINKS}}` — one `<a>` per `## heading`
- `{{BODY}}` — all converted HTML content
- `{{FILENAME}}` — base filename without extension
- Omit the Mermaid `<script>` block if no mermaid diagrams are present
