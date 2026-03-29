# Latte Java Website Design Spec

## Overview

Organization website for Latte Java (lattejava.org) — an open source organization with multiple Java projects. The site serves as the central hub for the organization's mission, project discovery, and project documentation.

**Mission:** Latte Java aims to make Java simple and easy to use

**Projects:** javaenv, cli, jwt, http

**Domain:** lattejava.org (GitHub Pages with custom CNAME)

## 1. Build & Deployment Architecture

- **Hugo** as the static site generator (latest version)
- **Tailwind CSS 4** with Hugo's built-in `css.TailwindCSS` pipe — no separate CSS build step needed
- **`@tailwindcss/typography`** plugin for markdown content styling via the `prose` class
- **`package.json`** only needs `tailwindcss` and `@tailwindcss/typography` as dependencies
- **Chroma** for syntax highlighting (Hugo's built-in) — eliminates external highlighter dependencies
- **Dark/light/system mode** via Tailwind's `class` strategy — JS manages `dark` class on `<html>`, persists choice in `localStorage`
- **GitHub Actions** for CI/CD: installs Hugo + Node, runs `hugo --minify`, deploys to GitHub Pages

## 2. Site Structure

```
website/
├── hugo.toml                  # Hugo config (site title, menus, params)
├── package.json               # Tailwind + PostCSS deps
├── assets/
│   └── css/
│       └── main.css           # Tailwind directives + Chroma theme overrides
├── content/
│   ├── _index.md              # Homepage content (mission, project cards)
│   ├── javaenv/
│   │   ├── _index.md          # Project landing page
│   │   └── docs/              # Optional docs section
│   │       ├── _index.md
│   │       ├── getting-started.md
│   │       └── ...
│   ├── cli/
│   │   └── _index.md          # Landing page only (no docs yet)
│   ├── jwt/
│   │   └── _index.md
│   └── http/
│       └── _index.md
├── layouts/
│   ├── _default/
│   │   ├── baseof.html        # Base template (head, nav, footer, dark mode JS)
│   │   ├── list.html          # Default list template
│   │   └── single.html        # Default single page template
│   ├── index.html             # Homepage layout (hero + project cards)
│   ├── partials/
│   │   ├── nav.html           # Top navigation with project dropdown + dark mode toggle
│   │   ├── footer.html
│   │   ├── theme-toggle.html  # Dark/light/system switcher
│   │   └── toc.html           # Table of contents partial (right column)
│   └── _default/
│       ├── landing.html       # Project landing page (selected via `layout: landing` front matter)
│       └── docs.html          # 3-column docs layout (selected via `layout: docs` front matter)
├── static/
│   └── images/
│       └── logo.svg
└── .github/
    └── workflows/
        └── deploy.yml         # GitHub Actions: Hugo + Tailwind build → Pages
```

### Key structural decisions

- Each project is a **Hugo section** under `content/` — gives each project its own URL prefix (`/javaenv/`, `/cli/`, etc.)
- Docs are an optional sub-section (`/javaenv/docs/`) — projects without docs just have a landing page
- Front matter in each project's `_index.md` controls which layout to use and provides metadata (description, GitHub URL, etc.)
- Navigation menu defined in `hugo.toml` — adding a new project is a config + content directory addition

## 3. Navigation & Layout

### Top navigation bar

- Logo (links to home) on the left
- "Projects" dropdown menu listing all 4 projects (driven by `hugo.toml` menu config)
- Dark/light/system mode toggle button on the right
- On mobile: collapses to hamburger menu with projects listed inline (no nested dropdown)

### Homepage

- Full-width hero section with the Latte Java logo, mission statement, and a CTA button (e.g., "Explore Projects" or link to GitHub org)
- Grid of project cards below — each card has project name, one-line description, and a link to its section

### Project landing page

- Hero/header area with project name and description
- Content from the project's `_index.md` (features, install instructions, links to docs/GitHub)
- Single-column layout

### Docs layout (3-column)

- **Left sidebar** — docs navigation tree for the current project, auto-generated from the docs section's pages, ordered by `weight` front matter
- **Center** — main content rendered from markdown, styled with `@tailwindcss/typography` `prose` classes
- **Right sidebar** — "On this page" TOC, auto-generated from headings via Hugo's `.TableOfContents`

### Responsive behavior

- **Large screens (>=1280px):** all 3 columns visible
- **Medium screens (>=768px, <1280px):** TOC (right sidebar) hidden, left nav + content remain
- **Small screens (<768px):** left nav hidden, replaced by hamburger button that reveals it as a slide-out overlay. Only content column visible by default.

## 4. Dark Mode Implementation

### Three states: Light, Dark, System

**Page load (flash prevention):**
- Inline `<script>` in `<head>` (before any render) reads `localStorage` for saved preference
- If "system" or no preference, checks `prefers-color-scheme: dark` media query
- Sets or removes `dark` class on `<html>` before first paint

**Toggle button:**
- Cycles: Light (sun icon) → Dark (moon icon) → System (monitor icon)
- Saves choice to `localStorage`
- Updates `<html>` class immediately

**Chroma syntax highlighting:**
- Two Chroma themes: one for light (e.g., `github`), one for dark (e.g., `monokai`)
- Light theme styles are default; dark theme styles scoped under `.dark` selector
- Both embedded in `main.css` so they swap instantly with mode toggle

**All elements respond:**
- Backgrounds, text, borders, cards, code blocks, nav, footer — all have Tailwind `dark:` variants
- Smooth transition via `transition-colors` on `<html>`

## 5. Visual Style

- **Dark & technical** aesthetic — similar to Tailwind CSS or Vercel sites
- **Dark mode:** deep navy/slate backgrounds (`slate-900`, `slate-950`), vibrant accent colors for links and interactive elements
- **Light mode:** white/light gray backgrounds, same accent colors adapted for light context
- Accent color chosen during implementation — vibrant blue or teal that works in both modes
- The Latte Java logo provides brand warmth; the site chrome stays clean and technical

## 6. GitHub Actions Deployment

**Workflow:** `.github/workflows/deploy.yml`

- **Trigger:** push to `main` branch
- **Steps:**
  1. Checkout repo
  2. Setup Node.js, run `npm install` (for Tailwind dependency)
  3. Setup Hugo (latest version via `peaceiris/actions-hugo`)
  4. Run `hugo --minify`
  5. Deploy `public/` to GitHub Pages via `actions/deploy-pages`
- **CNAME:** existing `CNAME` file moved to `static/` so Hugo copies it to output
