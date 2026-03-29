# Latte Java Website Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the lattejava.org website — an open source organization hub with homepage, project pages, and documentation support using Hugo + Tailwind CSS 4.

**Architecture:** Hugo static site with Tailwind CSS 4 via Hugo's built-in `css.TailwindCSS` pipe. Dark/light/system theme toggle. Projects as Hugo sections with optional docs sub-sections using a 3-column responsive layout. Deployed to GitHub Pages via GitHub Actions.

**Tech Stack:** Hugo (latest), Tailwind CSS 4, @tailwindcss/typography, Chroma syntax highlighting, GitHub Actions

---

### Task 1: Project Scaffolding

**Files:**
- Create: `hugo.toml`
- Create: `package.json`
- Modify: `.gitignore`
- Move: `CNAME` → `static/CNAME`
- Move: `images/logo.svg` → `static/images/logo.svg`

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "latte-java-website",
  "private": true,
  "devDependencies": {
    "tailwindcss": "^4",
    "@tailwindcss/cli": "^4",
    "@tailwindcss/typography": "^0.5"
  }
}
```

- [ ] **Step 2: Create `hugo.toml`**

```toml
baseURL = "https://lattejava.org/"
languageCode = "en-us"
title = "Latte Java"

[build]
  [build.buildStats]
    enable = true

[[build.cachebusters]]
  source = 'assets/notwatching/hugo_stats\.json'
  target = 'css'

[[build.cachebusters]]
  source = '(postcss|tailwind)\.config\.js'
  target = 'css'

[[module.mounts]]
  source = 'assets'
  target = 'assets'

[[module.mounts]]
  disableWatch = true
  source = 'hugo_stats.json'
  target = 'assets/notwatching/hugo_stats.json'

[markup.highlight]
  noClasses = false

[markup.tableOfContents]
  startLevel = 2
  endLevel = 4

[params]
  description = "Latte Java aims to make Java simple and easy to use"
  github = "https://github.com/latte-java"

[[menus.projects]]
  name = "javaenv"
  pageRef = "/javaenv"
  weight = 10

[[menus.projects]]
  name = "cli"
  pageRef = "/cli"
  weight = 20

[[menus.projects]]
  name = "jwt"
  pageRef = "/jwt"
  weight = 30

[[menus.projects]]
  name = "http"
  pageRef = "/http"
  weight = 40
```

- [ ] **Step 3: Update `.gitignore`**

Add these lines to the existing `.gitignore`:

```
public/
resources/
hugo_stats.json
node_modules/
.hugo_build.lock
.superpowers/
```

- [ ] **Step 4: Move static files**

```bash
mkdir -p static/images
mv CNAME static/CNAME
mv images/logo.svg static/images/logo.svg
rm -rf images
```

- [ ] **Step 5: Install npm dependencies**

```bash
npm install
```

- [ ] **Step 6: Commit**

```bash
git add hugo.toml package.json package-lock.json .gitignore static/
git rm CNAME
git commit -m "feat: scaffold Hugo project with Tailwind CSS 4"
```

---

### Task 2: Base Layout + Tailwind CSS

**Files:**
- Create: `assets/css/main.css`
- Create: `layouts/_default/baseof.html`
- Create: `layouts/partials/css.html`

- [ ] **Step 1: Create `assets/css/main.css`**

```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";
@source "../../hugo_stats.json";

@theme {
  --color-accent-400: oklch(0.7 0.15 230);
  --color-accent-500: oklch(0.6 0.18 230);
  --color-accent-600: oklch(0.5 0.18 230);
}
```

- [ ] **Step 2: Create `layouts/partials/css.html`**

```go-html-template
{{ with resources.Get "css/main.css" }}
  {{ $opts := dict "minify" (not hugo.IsDevelopment) }}
  {{ with . | css.TailwindCSS $opts }}
    {{ if hugo.IsDevelopment }}
      <link rel="stylesheet" href="{{ .RelPermalink }}">
    {{ else }}
      {{ with . | fingerprint }}
        <link rel="stylesheet" href="{{ .RelPermalink }}" integrity="{{ .Data.Integrity }}" crossorigin="anonymous">
      {{ end }}
    {{ end }}
  {{ end }}
{{ end }}
```

- [ ] **Step 3: Create `layouts/_default/baseof.html`**

```go-html-template
<!DOCTYPE html>
<html lang="en" class="scroll-smooth">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{ if .IsHome }}{{ .Site.Title }}{{ else }}{{ .Title }} | {{ .Site.Title }}{{ end }}</title>
  <meta name="description" content="{{ with .Description }}{{ . }}{{ else }}{{ .Site.Params.description }}{{ end }}">
  <link rel="icon" href="/images/logo.svg" type="image/svg+xml">
  <script>
    (function() {
      var pref = localStorage.getItem('theme');
      if (pref === 'dark' || (!pref && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
        document.documentElement.classList.add('dark');
      }
    })();
  </script>
  {{ with (templates.Defer (dict "key" "global")) }}
    {{ partial "css.html" . }}
  {{ end }}
</head>
<body class="bg-white text-slate-900 dark:bg-slate-950 dark:text-slate-100 transition-colors duration-200">
  {{ partial "nav.html" . }}
  <main>
    {{ block "main" . }}{{ end }}
  </main>
  {{ partial "footer.html" . }}
</body>
</html>
```

- [ ] **Step 4: Verify Hugo builds (will fail until partials exist — that's expected)**

```bash
hugo --printPathWarnings 2>&1 | head -20
```

Expected: errors about missing partials (nav.html, footer.html). This confirms Hugo recognizes the project structure.

- [ ] **Step 5: Commit**

```bash
git add assets/ layouts/_default/baseof.html layouts/partials/css.html
git commit -m "feat: add base layout with Tailwind CSS 4 integration"
```

---

### Task 3: Chroma Syntax Highlighting Themes

**Files:**
- Create: `assets/css/syntax-light.css`
- Create: `assets/css/syntax-dark.css`
- Modify: `assets/css/main.css`

- [ ] **Step 1: Generate Chroma CSS files**

```bash
hugo gen chromastyles --style=github > assets/css/syntax-light.css
hugo gen chromastyles --style=github-dark > assets/css/syntax-dark.css
```

- [ ] **Step 2: Scope dark theme under `.dark` selector**

Wrap the contents of `assets/css/syntax-dark.css` with `.dark` so it only applies in dark mode. The file should look like:

```css
.dark .chroma { ... }
.dark .chroma .err { ... }
/* etc — every selector prefixed with .dark */
```

Use this command to do the prefixing:

```bash
sed -i '' 's/^\.chroma/.dark .chroma/g; s/^\/\* Background \*\/ \.chroma/\/* Background *\/ .dark .chroma/g' assets/css/syntax-dark.css
```

Verify by checking the first few lines:

```bash
head -5 assets/css/syntax-dark.css
```

Expected: selectors prefixed with `.dark `.

- [ ] **Step 3: Import syntax themes in `main.css`**

Update `assets/css/main.css` to:

```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";
@source "../../hugo_stats.json";

@import "syntax-light.css";
@import "syntax-dark.css";

@theme {
  --color-accent-400: oklch(0.7 0.15 230);
  --color-accent-500: oklch(0.6 0.18 230);
  --color-accent-600: oklch(0.5 0.18 230);
}
```

- [ ] **Step 4: Commit**

```bash
git add assets/css/
git commit -m "feat: add Chroma syntax highlighting for light and dark modes"
```

---

### Task 4: Dark Mode Toggle

**Files:**
- Create: `layouts/partials/theme-toggle.html`

- [ ] **Step 1: Create `layouts/partials/theme-toggle.html`**

This partial renders the 3-state toggle button (light/dark/system) and manages localStorage + `<html>` class.

```go-html-template
<button id="theme-toggle" type="button" class="p-2 rounded-lg text-slate-500 hover:text-slate-700 dark:text-slate-400 dark:hover:text-slate-200 hover:bg-slate-100 dark:hover:bg-slate-800 transition-colors" aria-label="Toggle theme">
  <!-- Sun icon (light mode) -->
  <svg id="theme-light-icon" class="hidden w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
    <path stroke-linecap="round" stroke-linejoin="round" d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z" />
  </svg>
  <!-- Moon icon (dark mode) -->
  <svg id="theme-dark-icon" class="hidden w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
    <path stroke-linecap="round" stroke-linejoin="round" d="M20.354 15.354A9 9 0 018.646 3.646 9.003 9.003 0 0012 21a9.003 9.003 0 008.354-5.646z" />
  </svg>
  <!-- Monitor icon (system mode) -->
  <svg id="theme-system-icon" class="hidden w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
    <path stroke-linecap="round" stroke-linejoin="round" d="M9.75 17L9 20l-1 1h8l-1-1-.75-3M3 13h18M5 17h14a2 2 0 002-2V5a2 2 0 00-2-2H5a2 2 0 00-2 2v10a2 2 0 002 2z" />
  </svg>
</button>

<script>
(function() {
  var btn = document.getElementById('theme-toggle');
  var iconLight = document.getElementById('theme-light-icon');
  var iconDark = document.getElementById('theme-dark-icon');
  var iconSystem = document.getElementById('theme-system-icon');

  function getStoredTheme() {
    return localStorage.getItem('theme');
  }

  function applyTheme(pref) {
    var isDark = pref === 'dark' || (!pref && window.matchMedia('(prefers-color-scheme: dark)').matches);
    document.documentElement.classList.toggle('dark', isDark);
  }

  function updateIcon(pref) {
    iconLight.classList.add('hidden');
    iconDark.classList.add('hidden');
    iconSystem.classList.add('hidden');
    if (pref === 'light') iconLight.classList.remove('hidden');
    else if (pref === 'dark') iconDark.classList.remove('hidden');
    else iconSystem.classList.remove('hidden');
  }

  function init() {
    var pref = getStoredTheme();
    applyTheme(pref);
    updateIcon(pref);
  }

  btn.addEventListener('click', function() {
    var current = getStoredTheme();
    var next;
    if (current === 'light') next = 'dark';
    else if (current === 'dark') next = null;
    else next = 'light';

    if (next) {
      localStorage.setItem('theme', next);
    } else {
      localStorage.removeItem('theme');
    }
    applyTheme(next);
    updateIcon(next);
  });

  window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', function() {
    if (!getStoredTheme()) applyTheme(null);
  });

  init();
})();
</script>
```

- [ ] **Step 2: Commit**

```bash
git add layouts/partials/theme-toggle.html
git commit -m "feat: add dark/light/system theme toggle"
```

---

### Task 5: Navigation

**Files:**
- Create: `layouts/partials/nav.html`

- [ ] **Step 1: Create `layouts/partials/nav.html`**

Top navigation with logo, project dropdown, dark mode toggle, and mobile hamburger menu.

```go-html-template
<nav class="sticky top-0 z-50 border-b border-slate-200 dark:border-slate-800 bg-white/80 dark:bg-slate-950/80 backdrop-blur-sm">
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
    <div class="flex items-center justify-between h-16">
      <!-- Logo -->
      <a href="/" class="flex items-center gap-3 shrink-0">
        <img src="/images/logo.svg" alt="Latte Java" class="h-8 w-auto">
        <span class="font-bold text-lg text-slate-900 dark:text-white">Latte Java</span>
      </a>

      <!-- Desktop nav -->
      <div class="hidden md:flex items-center gap-6">
        <!-- Projects dropdown -->
        <div class="relative" id="projects-dropdown">
          <button type="button" class="flex items-center gap-1 text-sm font-medium text-slate-600 dark:text-slate-300 hover:text-slate-900 dark:hover:text-white transition-colors" onclick="document.getElementById('projects-menu').classList.toggle('hidden')">
            Projects
            <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 9l-7 7-7-7" /></svg>
          </button>
          <div id="projects-menu" class="hidden absolute top-full left-0 mt-2 w-48 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 rounded-lg shadow-lg py-1">
            {{ range .Site.Menus.projects }}
              <a href="{{ .URL }}" class="block px-4 py-2 text-sm text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800">{{ .Name }}</a>
            {{ end }}
          </div>
        </div>

        <a href="{{ .Site.Params.github }}" class="text-sm font-medium text-slate-600 dark:text-slate-300 hover:text-slate-900 dark:hover:text-white transition-colors" target="_blank" rel="noopener">GitHub</a>

        {{ partial "theme-toggle.html" . }}
      </div>

      <!-- Mobile menu button -->
      <div class="flex items-center gap-2 md:hidden">
        {{ partial "theme-toggle.html" . }}
        <button type="button" id="mobile-menu-btn" class="p-2 rounded-lg text-slate-500 hover:text-slate-700 dark:text-slate-400 dark:hover:text-slate-200 hover:bg-slate-100 dark:hover:bg-slate-800 transition-colors" aria-label="Open menu">
          <svg class="w-6 h-6" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M4 6h16M4 12h16M4 18h16" /></svg>
        </button>
      </div>
    </div>
  </div>

  <!-- Mobile menu -->
  <div id="mobile-menu" class="hidden md:hidden border-t border-slate-200 dark:border-slate-800 bg-white dark:bg-slate-950">
    <div class="px-4 py-3 space-y-1">
      <div class="text-xs font-semibold uppercase tracking-wider text-slate-400 dark:text-slate-500 px-2 py-1">Projects</div>
      {{ range .Site.Menus.projects }}
        <a href="{{ .URL }}" class="block px-2 py-2 text-sm text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 rounded">{{ .Name }}</a>
      {{ end }}
      <div class="border-t border-slate-200 dark:border-slate-800 pt-2 mt-2">
        <a href="{{ .Site.Params.github }}" class="block px-2 py-2 text-sm text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 rounded" target="_blank" rel="noopener">GitHub</a>
      </div>
    </div>
  </div>
</nav>

<script>
(function() {
  var mobileBtn = document.getElementById('mobile-menu-btn');
  var mobileMenu = document.getElementById('mobile-menu');
  if (mobileBtn && mobileMenu) {
    mobileBtn.addEventListener('click', function() {
      mobileMenu.classList.toggle('hidden');
    });
  }

  // Close projects dropdown when clicking outside
  document.addEventListener('click', function(e) {
    var dropdown = document.getElementById('projects-dropdown');
    var menu = document.getElementById('projects-menu');
    if (dropdown && menu && !dropdown.contains(e.target)) {
      menu.classList.add('hidden');
    }
  });
})();
</script>
```

- [ ] **Step 2: Commit**

```bash
git add layouts/partials/nav.html
git commit -m "feat: add navigation with project dropdown and mobile menu"
```

---

### Task 6: Footer

**Files:**
- Create: `layouts/partials/footer.html`

- [ ] **Step 1: Create `layouts/partials/footer.html`**

```go-html-template
<footer class="border-t border-slate-200 dark:border-slate-800 bg-slate-50 dark:bg-slate-900">
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12">
    <div class="flex flex-col md:flex-row justify-between gap-8">
      <div>
        <a href="/" class="flex items-center gap-3">
          <img src="/images/logo.svg" alt="Latte Java" class="h-8 w-auto">
          <span class="font-bold text-lg text-slate-900 dark:text-white">Latte Java</span>
        </a>
        <p class="mt-3 text-sm text-slate-500 dark:text-slate-400 max-w-xs">{{ .Site.Params.description }}</p>
      </div>
      <div class="flex gap-16">
        <div>
          <h3 class="text-sm font-semibold text-slate-900 dark:text-white">Projects</h3>
          <ul class="mt-3 space-y-2">
            {{ range .Site.Menus.projects }}
              <li><a href="{{ .URL }}" class="text-sm text-slate-500 dark:text-slate-400 hover:text-slate-900 dark:hover:text-white transition-colors">{{ .Name }}</a></li>
            {{ end }}
          </ul>
        </div>
        <div>
          <h3 class="text-sm font-semibold text-slate-900 dark:text-white">Community</h3>
          <ul class="mt-3 space-y-2">
            <li><a href="{{ .Site.Params.github }}" class="text-sm text-slate-500 dark:text-slate-400 hover:text-slate-900 dark:hover:text-white transition-colors" target="_blank" rel="noopener">GitHub</a></li>
          </ul>
        </div>
      </div>
    </div>
    <div class="mt-8 pt-8 border-t border-slate-200 dark:border-slate-800">
      <p class="text-sm text-slate-400 dark:text-slate-500">&copy; {{ now.Year }} Latte Java. Open source under the Apache 2.0 License.</p>
    </div>
  </div>
</footer>
```

- [ ] **Step 2: Commit**

```bash
git add layouts/partials/footer.html
git commit -m "feat: add footer"
```

---

### Task 7: Homepage

**Files:**
- Create: `layouts/index.html`
- Create: `content/_index.md`

- [ ] **Step 1: Create `content/_index.md`**

```yaml
---
title: "Latte Java"
description: "Latte Java aims to make Java simple and easy to use"
---
```

- [ ] **Step 2: Create `layouts/index.html`**

```go-html-template
{{ define "main" }}
  <!-- Hero -->
  <section class="relative overflow-hidden">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-24 sm:py-32">
      <div class="text-center">
        <img src="/images/logo.svg" alt="Latte Java" class="h-24 w-auto mx-auto mb-8">
        <h1 class="text-4xl sm:text-5xl lg:text-6xl font-bold tracking-tight text-slate-900 dark:text-white">
          Latte Java
        </h1>
        <p class="mt-6 text-lg sm:text-xl text-slate-600 dark:text-slate-400 max-w-2xl mx-auto">
          {{ .Site.Params.description }}
        </p>
        <div class="mt-10 flex items-center justify-center gap-4">
          <a href="#projects" class="px-6 py-3 bg-accent-500 hover:bg-accent-600 text-white font-medium rounded-lg transition-colors">
            Explore Projects
          </a>
          <a href="{{ .Site.Params.github }}" class="px-6 py-3 border border-slate-300 dark:border-slate-700 text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 font-medium rounded-lg transition-colors" target="_blank" rel="noopener">
            GitHub
          </a>
        </div>
      </div>
    </div>
  </section>

  <!-- Project Cards -->
  <section id="projects" class="border-t border-slate-200 dark:border-slate-800 bg-slate-50 dark:bg-slate-900">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-16 sm:py-24">
      <h2 class="text-2xl sm:text-3xl font-bold text-slate-900 dark:text-white text-center mb-12">Projects</h2>
      <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-6">
        {{ range .Site.Menus.projects }}
          {{ with site.GetPage .PageRef }}
            <a href="{{ .RelPermalink }}" class="group block p-6 bg-white dark:bg-slate-800 border border-slate-200 dark:border-slate-700 rounded-xl hover:border-accent-500 dark:hover:border-accent-400 hover:shadow-lg transition-all">
              <h3 class="text-lg font-semibold text-slate-900 dark:text-white group-hover:text-accent-500 dark:group-hover:text-accent-400 transition-colors">{{ .Title }}</h3>
              <p class="mt-2 text-sm text-slate-500 dark:text-slate-400">{{ .Description }}</p>
            </a>
          {{ end }}
        {{ end }}
      </div>
    </div>
  </section>
{{ end }}
```

- [ ] **Step 3: Verify Hugo builds the homepage**

```bash
hugo --printPathWarnings 2>&1 | tail -5
```

Expected: Build succeeds (may show warnings about missing project pages — that's fine, we create them in the next task).

- [ ] **Step 4: Commit**

```bash
git add layouts/index.html content/_index.md
git commit -m "feat: add homepage with hero and project cards"
```

---

### Task 8: Project Landing Pages

**Files:**
- Create: `layouts/_default/landing.html`
- Create: `content/javaenv/_index.md`
- Create: `content/cli/_index.md`
- Create: `content/jwt/_index.md`
- Create: `content/http/_index.md`

- [ ] **Step 1: Create `layouts/_default/landing.html`**

```go-html-template
{{ define "main" }}
  <section class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-16 sm:py-24">
    <div class="max-w-3xl">
      <h1 class="text-3xl sm:text-4xl font-bold text-slate-900 dark:text-white">{{ .Title }}</h1>
      <p class="mt-4 text-lg text-slate-600 dark:text-slate-400">{{ .Description }}</p>
      {{ with .Params.github }}
        <div class="mt-6">
          <a href="{{ . }}" class="inline-flex items-center gap-2 px-4 py-2 text-sm font-medium border border-slate-300 dark:border-slate-700 text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 rounded-lg transition-colors" target="_blank" rel="noopener">
            <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 24 24"><path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"/></svg>
            View on GitHub
          </a>
        </div>
      {{ end }}
    </div>
    {{ if .Content }}
      <div class="mt-12 max-w-3xl prose dark:prose-invert prose-slate">
        {{ .Content }}
      </div>
    {{ end }}
    {{ if .Sections }}
      <div class="mt-12">
        {{ range .Sections }}
          <a href="{{ .RelPermalink }}" class="inline-flex items-center gap-2 px-6 py-3 bg-accent-500 hover:bg-accent-600 text-white font-medium rounded-lg transition-colors">
            Read the Docs
            <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M9 5l7 7-7 7" /></svg>
          </a>
        {{ end }}
      </div>
    {{ end }}
  </section>
{{ end }}
```

- [ ] **Step 2: Create project content files**

`content/javaenv/_index.md`:
```yaml
---
title: "javaenv"
description: "Manage multiple Java versions on your machine"
layout: landing
github: "https://github.com/latte-java/javaenv"
---
```

`content/cli/_index.md`:
```yaml
---
title: "cli"
description: "Build command-line applications in Java"
layout: landing
github: "https://github.com/latte-java/cli"
---
```

`content/jwt/_index.md`:
```yaml
---
title: "jwt"
description: "JSON Web Token library for Java"
layout: landing
github: "https://github.com/latte-java/jwt"
---
```

`content/http/_index.md`:
```yaml
---
title: "http"
description: "Lightweight HTTP client and server for Java"
layout: landing
github: "https://github.com/latte-java/http"
---
```

- [ ] **Step 3: Verify Hugo builds all project pages**

```bash
hugo --printPathWarnings 2>&1 | tail -10
```

Expected: Build succeeds. Check output includes pages for `/javaenv/`, `/cli/`, `/jwt/`, `/http/`.

- [ ] **Step 4: Commit**

```bash
git add layouts/_default/landing.html content/
git commit -m "feat: add project landing page layout and project content"
```

---

### Task 9: Documentation Layout (3-Column)

**Files:**
- Create: `layouts/_default/docs.html`
- Create: `layouts/partials/docs-nav.html`
- Create: `layouts/partials/toc.html`

- [ ] **Step 1: Create `layouts/partials/docs-nav.html`**

Left sidebar navigation for docs. Generates a list of pages within the current project's `docs/` section, ordered by `weight`.

```go-html-template
{{ $section := .CurrentSection }}
{{ $pages := $section.Parent.RegularPagesRecursive }}
<nav class="space-y-1">
  <div class="mb-4">
    <a href="{{ $section.Parent.RelPermalink }}" class="text-sm font-semibold text-slate-900 dark:text-white hover:text-accent-500 dark:hover:text-accent-400 transition-colors">
      &larr; {{ $section.Parent.Title }}
    </a>
  </div>
  <h3 class="text-xs font-semibold uppercase tracking-wider text-slate-400 dark:text-slate-500">Documentation</h3>
  {{ range $section.Pages.ByWeight }}
    <a href="{{ .RelPermalink }}" class="block py-1.5 px-3 text-sm rounded-md transition-colors {{ if eq $.RelPermalink .RelPermalink }}bg-accent-500/10 text-accent-600 dark:text-accent-400 font-medium{{ else }}text-slate-600 dark:text-slate-400 hover:text-slate-900 dark:hover:text-white hover:bg-slate-100 dark:hover:bg-slate-800{{ end }}">
      {{ .Title }}
    </a>
  {{ end }}
</nav>
```

- [ ] **Step 2: Create `layouts/partials/toc.html`**

Right sidebar "On this page" table of contents using Hugo's `.TableOfContents`.

```go-html-template
{{ if .TableOfContents }}
  <nav class="space-y-2">
    <h3 class="text-xs font-semibold uppercase tracking-wider text-slate-400 dark:text-slate-500">On this page</h3>
    <div class="text-sm toc">
      {{ .TableOfContents }}
    </div>
  </nav>
{{ end }}
```

- [ ] **Step 3: Create `layouts/_default/docs.html`**

3-column layout with responsive collapse.

```go-html-template
{{ define "main" }}
  <div class="max-w-8xl mx-auto px-4 sm:px-6 lg:px-8">
    <!-- Mobile docs nav toggle -->
    <div class="md:hidden flex items-center gap-2 py-3 border-b border-slate-200 dark:border-slate-800">
      <button type="button" id="docs-nav-toggle" class="p-2 -ml-2 rounded-lg text-slate-500 hover:text-slate-700 dark:text-slate-400 dark:hover:text-slate-200 hover:bg-slate-100 dark:hover:bg-slate-800 transition-colors" aria-label="Toggle navigation">
        <svg class="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M4 6h16M4 12h16M4 18h16" /></svg>
      </button>
      <span class="text-sm font-medium text-slate-900 dark:text-white">{{ .Title }}</span>
    </div>

    <div class="md:flex md:gap-8 py-8">
      <!-- Left sidebar: docs nav -->
      <aside id="docs-nav" class="hidden md:block md:w-64 shrink-0">
        <div class="sticky top-24 overflow-y-auto max-h-[calc(100vh-6rem)]">
          {{ partial "docs-nav.html" . }}
        </div>
      </aside>

      <!-- Mobile nav overlay -->
      <div id="docs-nav-overlay" class="hidden fixed inset-0 z-40 md:hidden">
        <div class="absolute inset-0 bg-slate-900/50" id="docs-nav-backdrop"></div>
        <div class="relative w-72 h-full bg-white dark:bg-slate-950 border-r border-slate-200 dark:border-slate-800 p-6 overflow-y-auto">
          {{ partial "docs-nav.html" . }}
        </div>
      </div>

      <!-- Main content -->
      <article class="min-w-0 flex-1 max-w-3xl">
        <h1 class="text-3xl font-bold text-slate-900 dark:text-white mb-8">{{ .Title }}</h1>
        <div class="prose dark:prose-invert prose-slate max-w-none
          prose-headings:scroll-mt-20
          prose-a:text-accent-500 dark:prose-a:text-accent-400
          prose-code:bg-slate-100 dark:prose-code:bg-slate-800 prose-code:px-1.5 prose-code:py-0.5 prose-code:rounded prose-code:text-sm
          prose-pre:bg-slate-100 dark:prose-pre:bg-slate-800/50 prose-pre:border prose-pre:border-slate-200 dark:prose-pre:border-slate-700">
          {{ .Content }}
        </div>
      </article>

      <!-- Right sidebar: TOC -->
      <aside class="hidden xl:block xl:w-56 shrink-0">
        <div class="sticky top-24 overflow-y-auto max-h-[calc(100vh-6rem)]">
          {{ partial "toc.html" . }}
        </div>
      </aside>
    </div>
  </div>

  <script>
  (function() {
    var toggle = document.getElementById('docs-nav-toggle');
    var overlay = document.getElementById('docs-nav-overlay');
    var backdrop = document.getElementById('docs-nav-backdrop');
    if (!toggle || !overlay) return;

    toggle.addEventListener('click', function() {
      overlay.classList.toggle('hidden');
    });
    backdrop.addEventListener('click', function() {
      overlay.classList.add('hidden');
    });
  })();
  </script>
{{ end }}
```

- [ ] **Step 4: Add TOC link styling to `assets/css/main.css`**

Append to `assets/css/main.css`:

```css
/* TOC styling */
.toc ul {
  list-style: none;
  padding-left: 0;
}

.toc ul ul {
  padding-left: 1rem;
}

.toc a {
  display: block;
  padding: 0.25rem 0;
  color: var(--color-slate-500);
  transition: color 0.15s;
}

.toc a:hover {
  color: var(--color-slate-900);
}

:is(.dark) .toc a {
  color: var(--color-slate-400);
}

:is(.dark) .toc a:hover {
  color: var(--color-white);
}
```

- [ ] **Step 5: Commit**

```bash
git add layouts/_default/docs.html layouts/partials/docs-nav.html layouts/partials/toc.html assets/css/main.css
git commit -m "feat: add 3-column docs layout with responsive behavior"
```

---

### Task 10: Sample Documentation Content

**Files:**
- Create: `content/javaenv/docs/_index.md`
- Create: `content/javaenv/docs/getting-started.md`
- Create: `content/javaenv/docs/installation.md`

- [ ] **Step 1: Create docs section index**

`content/javaenv/docs/_index.md`:
```yaml
---
title: "javaenv Documentation"
layout: docs
---
```

- [ ] **Step 2: Create sample doc pages**

`content/javaenv/docs/installation.md`:
```yaml
---
title: "Installation"
layout: docs
weight: 10
---

## Prerequisites

You need a Unix-like operating system (macOS or Linux).

## Install via Homebrew

```bash
brew install latte-java/tap/javaenv
```

## Install from Source

```bash
git clone https://github.com/latte-java/javaenv.git
cd javaenv
make install
```

## Verify Installation

```bash
javaenv --version
```
```

`content/javaenv/docs/getting-started.md`:
```yaml
---
title: "Getting Started"
layout: docs
weight: 20
---

## List Available Versions

```bash
javaenv list-remote
```

## Install a Java Version

```bash
javaenv install 21
```

## Set the Global Version

```bash
javaenv global 21
```

## Verify

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello from Java " + System.getProperty("java.version"));
    }
}
```

```bash
javac Hello.java && java Hello
```
```

- [ ] **Step 3: Verify the full site builds**

```bash
hugo --printPathWarnings
```

Expected: Build succeeds with pages for `/`, `/javaenv/`, `/javaenv/docs/`, `/javaenv/docs/installation/`, `/javaenv/docs/getting-started/`, `/cli/`, `/jwt/`, `/http/`.

- [ ] **Step 4: Test local dev server**

```bash
hugo server -D
```

Visit `http://localhost:1313` and verify:
- Homepage renders with hero and project cards
- Project dropdown works in navigation
- Dark/light/system toggle works
- `/javaenv/` landing page renders
- `/javaenv/docs/installation/` renders with 3-column layout
- Left nav highlights current page
- TOC appears on right for doc pages
- Responsive behavior: TOC hides at <1280px, left nav becomes hamburger at <768px

- [ ] **Step 5: Commit**

```bash
git add content/javaenv/docs/
git commit -m "feat: add sample javaenv documentation"
```

---

### Task 11: Default List and Single Templates

**Files:**
- Create: `layouts/_default/list.html`
- Create: `layouts/_default/single.html`

- [ ] **Step 1: Create `layouts/_default/list.html`**

Fallback list template for any section that doesn't specify a layout.

```go-html-template
{{ define "main" }}
  <section class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-16">
    <h1 class="text-3xl font-bold text-slate-900 dark:text-white mb-8">{{ .Title }}</h1>
    {{ if .Content }}
      <div class="prose dark:prose-invert prose-slate max-w-3xl mb-12">
        {{ .Content }}
      </div>
    {{ end }}
    {{ if .Pages }}
      <ul class="space-y-4 max-w-3xl">
        {{ range .Pages }}
          <li>
            <a href="{{ .RelPermalink }}" class="group block p-4 border border-slate-200 dark:border-slate-700 rounded-lg hover:border-accent-500 dark:hover:border-accent-400 transition-colors">
              <h2 class="text-lg font-semibold text-slate-900 dark:text-white group-hover:text-accent-500 dark:group-hover:text-accent-400 transition-colors">{{ .Title }}</h2>
              {{ with .Description }}<p class="mt-1 text-sm text-slate-500 dark:text-slate-400">{{ . }}</p>{{ end }}
            </a>
          </li>
        {{ end }}
      </ul>
    {{ end }}
  </section>
{{ end }}
```

- [ ] **Step 2: Create `layouts/_default/single.html`**

Fallback single page template.

```go-html-template
{{ define "main" }}
  <article class="max-w-3xl mx-auto px-4 sm:px-6 lg:px-8 py-16">
    <h1 class="text-3xl font-bold text-slate-900 dark:text-white mb-8">{{ .Title }}</h1>
    <div class="prose dark:prose-invert prose-slate max-w-none
      prose-a:text-accent-500 dark:prose-a:text-accent-400
      prose-code:bg-slate-100 dark:prose-code:bg-slate-800 prose-code:px-1.5 prose-code:py-0.5 prose-code:rounded prose-code:text-sm
      prose-pre:bg-slate-100 dark:prose-pre:bg-slate-800/50 prose-pre:border prose-pre:border-slate-200 dark:prose-pre:border-slate-700">
      {{ .Content }}
    </div>
  </article>
{{ end }}
```

- [ ] **Step 3: Commit**

```bash
git add layouts/_default/list.html layouts/_default/single.html
git commit -m "feat: add default list and single page templates"
```

---

### Task 12: GitHub Actions Deployment

**Files:**
- Create: `.github/workflows/deploy.yml`

- [ ] **Step 1: Create `.github/workflows/deploy.yml`**

```yaml
name: Build and deploy

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.147.5
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install Node.js dependencies
        run: npm ci

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Install Hugo
        run: |
          curl -sLJO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir -p "${HOME}/.local/bin"
          tar -C "${HOME}/.local/bin" -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          rm "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/.local/bin" >> "${GITHUB_PATH}"

      - name: Build
        run: |
          hugo build \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/deploy.yml
git commit -m "feat: add GitHub Actions workflow for Hugo deployment"
```

---

### Task 13: Final Verification

- [ ] **Step 1: Full build check**

```bash
hugo --minify --printPathWarnings
```

Expected: Clean build, no errors, all pages generated.

- [ ] **Step 2: Run local server and verify all pages**

```bash
hugo server
```

Verify at `http://localhost:1313`:
1. Homepage: hero, mission, project cards all render
2. Navigation: projects dropdown works, mobile hamburger works
3. Dark mode: toggle cycles light → dark → system, persists across page loads
4. Project pages: all 4 render at `/javaenv/`, `/cli/`, `/jwt/`, `/http/`
5. Docs: `/javaenv/docs/installation/` shows 3-column layout
6. Docs responsive: shrink browser — TOC hides at <1280px, left nav becomes overlay at <768px
7. Syntax highlighting: code blocks have proper colors in both light and dark mode
8. Footer: renders with project links and copyright

- [ ] **Step 3: Final commit if any fixes were needed**

```bash
git add -A
git commit -m "fix: final adjustments from verification"
```
