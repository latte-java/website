# "Why Latte?" Page Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an opinionated `/why/` page that argues why Latte exists (the Java tooling/ecosystem became a second job), using 5 problem→solution pairs with before/after code comparisons, linked from the top nav and the homepage.

**Architecture:** A bespoke Hugo section layout (`layouts/why/list.html`) renders all copy and code inline, mirroring the homepage precedent (`layouts/index.html`). A reusable partial (`layouts/partials/why-compare.html`) renders each side-by-side before/after comparison using Hugo's `highlight` function so the existing Chroma CSS and copy-button script apply. The nav partial and homepage layout get links to the new page.

**Tech Stack:** Hugo 0.159 (extended), Tailwind CSS 4 (sourced from `hugo_stats.json`), Chroma syntax highlighting, vanilla JS for the copy buttons.

---

## Important conventions (read before starting)

- **No unit-test framework exists** — this is a static site. The verification analog for every task is: run a clean Hugo build and confirm (a) it exits 0 with no template errors and (b) the expected strings appear in the generated HTML under `public/`. That is the "test" in each task below.
- **Tailwind class generation:** `assets/css/main.css` uses `@source "../../hugo_stats.json"`. Hugo's `build.buildStats` writes every class name from rendered HTML into `hugo_stats.json`, and CSS generation is deferred (`templates.Defer`) until after rendering — so any new Tailwind utility class used in a template (including `rose-*` colors and `/opacity` modifiers like `bg-accent-500/5`) is generated automatically on a full build. No manual CSS registration is needed.
- **Build command used throughout:** `hugo --gc` (builds to `public/`). Do NOT `git add public/` in commits — only add source files under `content/`, `layouts/`. (`public/` is build output.)
- **Custom color tokens available:** `accent-400`, `accent-500`, `accent-600` (defined in `assets/css/main.css`). `slate-*` and `rose-*` are stock Tailwind.
- **Syntax highlighting in templates:** `{{ highlight (strings.TrimSpace $code) "<lang>" "" }}` returns Chroma-highlighted `template.HTML` wrapped in `<div class="highlight">`, which the existing `.copy-btn` script and `assets/css/syntax.css` already style.
- **Go template raw strings:** multi-line code snippets are defined as backtick raw-string literals in the template (e.g. `{{ $pom := ` + "`" + `...` + "`" + ` }}`). None of the snippets in this plan contain a backtick, so this is safe.

---

## File Structure

- **Create** `content/why/_index.md` — section page; front matter only (title, description, og image, `layout: why`). No body.
- **Create** `layouts/why/list.html` — bespoke section layout: hero, 5 problem→solution sections, closing CTA, and the copy-button script.
- **Create** `layouts/partials/why-compare.html` — renders one before/after two-column comparison.
- **Modify** `layouts/partials/nav.html` — add a "Why Latte?" link (desktop nav + mobile menu).
- **Modify** `layouts/index.html` — add a "Why Latte?" hero button and a callout banner section.

---

## Task 1: Content file + layout skeleton (hero + closing CTA)

Create the section and a minimal bespoke layout that renders the hero and closing CTA. Sections are added in Task 3.

**Files:**
- Create: `content/why/_index.md`
- Create: `layouts/why/list.html`

- [ ] **Step 1: Create the content file**

Create `content/why/_index.md`:

```markdown
---
title: "Why Latte?"
description: "Java is great. The tooling and ecosystem around it became a second job. Latte uncomplicates it."
layout: why
og_image: "/images/og/default.png"
---
```

- [ ] **Step 2: Create the layout skeleton**

Create `layouts/why/list.html`:

```go-html-template
{{ define "main" }}
  <!-- Hero -->
  <section class="relative overflow-hidden border-b border-slate-200 dark:border-slate-800">
    <div class="max-w-3xl mx-auto px-4 sm:px-6 lg:px-8 py-20 sm:py-28 text-center">
      <h1 class="text-4xl sm:text-5xl font-bold tracking-tight text-slate-900 dark:text-white">
        Java got complicated.<br class="hidden sm:block"> We're uncomplicating it.
      </h1>
      <p class="mt-6 text-lg sm:text-xl text-slate-600 dark:text-slate-400">
        The language is great. The tooling and ecosystem that grew around it turned into a
        second job — build files you fight, releases that take a weekend, libraries that drag
        in the world, and ceremony for the simplest things. Latte brings the simple back.
      </p>
    </div>
  </section>

  <div id="why-content">
    <!-- Problem/solution sections inserted in Task 3 -->
  </div>

  <!-- Closing CTA -->
  <section class="border-t border-slate-200 dark:border-slate-800 bg-slate-50 dark:bg-slate-900">
    <div class="max-w-3xl mx-auto px-4 sm:px-6 lg:px-8 py-16 sm:py-24 text-center">
      <h2 class="text-2xl sm:text-3xl font-bold text-slate-900 dark:text-white">Get started in five minutes</h2>
      <p class="mt-4 text-lg text-slate-600 dark:text-slate-400">
        Install Java, install Latte, create a project, and ship it. No XML. No staging dance.
      </p>
      <div class="mt-8 flex items-center justify-center gap-4">
        <a href="/#getting-started-content" class="px-6 py-3 bg-accent-500 hover:bg-accent-600 text-white font-medium rounded-lg transition-colors">
          Get Started
        </a>
        <a href="{{ .Site.Params.github }}" class="px-6 py-3 border border-slate-300 dark:border-slate-700 text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 font-medium rounded-lg transition-colors" target="_blank" rel="noopener">
          GitHub
        </a>
      </div>
    </div>
  </section>
{{ end }}
```

- [ ] **Step 3: Build and verify the page renders**

Run: `hugo --gc`
Expected: exits 0, no `ERROR`/`WARN` template lines. Output includes a `why/index.html` page.

- [ ] **Step 4: Confirm expected content rendered**

Run: `grep -c "uncomplicating it" public/why/index.html`
Expected: `1` (or greater).

Run: `grep -c "Get started in five minutes" public/why/index.html`
Expected: `1`.

- [ ] **Step 5: Commit**

```bash
git add content/why/_index.md layouts/why/list.html
git commit -m "feat: add Why Latte page skeleton (hero + CTA)"
```

---

## Task 2: Before/after comparison partial

A reusable partial that renders a two-column before/after comparison. The **Before** column is labeled in muted rose; the **After** column in the accent color. Stacks on mobile.

**Files:**
- Create: `layouts/partials/why-compare.html`

- [ ] **Step 1: Create the partial**

Create `layouts/partials/why-compare.html`. It expects a dict with: `beforeLabel`, `beforeLang`, `beforeCode`, `beforeNote` (optional), `afterLabel`, `afterLang`, `afterCode`, `afterNote` (optional).

```go-html-template
<div class="grid grid-cols-1 md:grid-cols-2 gap-6 mt-8">
  <!-- Before -->
  <div class="rounded-xl border border-rose-200 dark:border-rose-900/60 overflow-hidden flex flex-col">
    <div class="px-4 py-2 bg-rose-50 dark:bg-rose-950/30 border-b border-rose-200 dark:border-rose-900/60">
      <span class="text-xs font-semibold uppercase tracking-wider text-rose-600 dark:text-rose-400">{{ .beforeLabel }}</span>
    </div>
    <div class="why-code grow text-sm">{{ highlight (strings.TrimSpace .beforeCode) .beforeLang "" }}</div>
    {{ with .beforeNote }}
      <p class="px-4 py-3 text-xs text-slate-500 dark:text-slate-400 border-t border-rose-200 dark:border-rose-900/60">{{ . }}</p>
    {{ end }}
  </div>
  <!-- After -->
  <div class="rounded-xl border border-accent-500/40 dark:border-accent-400/40 overflow-hidden flex flex-col">
    <div class="px-4 py-2 bg-accent-500/10 dark:bg-accent-400/10 border-b border-accent-500/40 dark:border-accent-400/40">
      <span class="text-xs font-semibold uppercase tracking-wider text-accent-600 dark:text-accent-400">{{ .afterLabel }}</span>
    </div>
    <div class="why-code grow text-sm">{{ highlight (strings.TrimSpace .afterCode) .afterLang "" }}</div>
    {{ with .afterNote }}
      <p class="px-4 py-3 text-xs text-slate-500 dark:text-slate-400 border-t border-accent-500/40 dark:border-accent-400/40">{{ . }}</p>
    {{ end }}
  </div>
</div>
```

- [ ] **Step 2: Verify the partial parses (build still clean)**

The partial is not yet referenced, so this just confirms it has no syntax error once used. Proceed to Task 3 which uses it; the build in Task 3 Step 3 validates parsing. No separate commit — commit happens with Task 3.

---

## Task 3: The five problem→solution sections

Insert five sections into `layouts/why/list.html` inside the `<div id="why-content">` placeholder, alternating white / `slate-50` backgrounds, each calling the `why-compare` partial. Then add the copy-button script.

All code snippets below are accurate and defensible (sources noted in the spec). Java 25 class-less `void main` + `IO.println` is JEP 512 (final in Java 25). The Spring snippet is a standard Spring Boot REST controller. The Maven Central plugin block is the standard sources + javadoc + GPG + central-publishing setup. The `mvn dependency:tree` output reflects `com.auth0:java-jwt` pulling Jackson Databind.

**Files:**
- Modify: `layouts/why/list.html`

- [ ] **Step 1: Replace the placeholder div with the five sections**

In `layouts/why/list.html`, replace this:

```go-html-template
  <div id="why-content">
    <!-- Problem/solution sections inserted in Task 3 -->
  </div>
```

with:

```go-html-template
  <div id="why-content">

    <!-- 1. Dependency management -->
    <section class="border-b border-slate-200 dark:border-slate-800">
      <div class="max-w-5xl mx-auto px-4 sm:px-6 lg:px-8 py-16 sm:py-20">
        <h2 class="text-2xl sm:text-3xl font-bold text-slate-900 dark:text-white">Dependency management is a second job</h2>
        <p class="mt-4 text-lg text-slate-600 dark:text-slate-400 max-w-3xl">
          Maven wants walls of XML. Gradle makes you pick between two DSLs and then guess which
          plugin and which configuration block does what. Either way you spend more time feeding
          the build tool than writing code. Latte uses one readable file and two commands.
        </p>
        {{ partial "why-compare.html" (dict
          "beforeLabel" "Maven — pom.xml"
          "beforeLang" "xml"
          "beforeCode" `<project>
  <!-- groupId, artifactId, version, properties... -->
  <dependencies>
    <dependency>
      <groupId>org.lattejava</groupId>
      <artifactId>web</artifactId>
      <version>0.1.0</version>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.13.0</version>
        <configuration><release>25</release></configuration>
      </plugin>
    </plugins>
  </build>
</project>`
          "beforeNote" "...and a separate settings.xml, a wrapper script, and a plugin for anything beyond compiling."
          "afterLabel" "Latte — project.latte"
          "afterLang" "groovy"
          "afterCode" `dependency(id: "org.lattejava:web:0.1.0")`
          "afterNote" "latte install to fetch it, latte build to build. That's the whole story."
        ) }}
      </div>
    </section>

    <!-- 2. Publishing -->
    <section class="border-b border-slate-200 dark:border-slate-800 bg-slate-50 dark:bg-slate-900">
      <div class="max-w-5xl mx-auto px-4 sm:px-6 lg:px-8 py-16 sm:py-20">
        <h2 class="text-2xl sm:text-3xl font-bold text-slate-900 dark:text-white">Publishing an artifact shouldn't take a weekend</h2>
        <p class="mt-4 text-lg text-slate-600 dark:text-slate-400 max-w-3xl">
          Shipping a library to Maven Central means a Sonatype account, a published GPG key,
          signed jars, generated sources and javadoc jars, and the staging "close &amp; release"
          ritual. Most people give up the first time. With Latte you log in, create a Group, and
          release.
        </p>
        {{ partial "why-compare.html" (dict
          "beforeLabel" "Maven Central — pom.xml plugins"
          "beforeLang" "xml"
          "beforeCode" `<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-source-plugin</artifactId>
      <version>3.3.1</version>
      <executions><execution><id>attach-sources</id>
        <goals><goal>jar-no-fork</goal></goals></execution></executions>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-javadoc-plugin</artifactId>
      <version>3.11.2</version>
      <executions><execution><id>attach-javadocs</id>
        <goals><goal>jar</goal></goals></execution></executions>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-gpg-plugin</artifactId>
      <version>3.2.7</version>
      <executions><execution><id>sign-artifacts</id>
        <phase>verify</phase><goals><goal>sign</goal></goals></execution></executions>
    </plugin>
    <plugin>
      <groupId>org.sonatype.central</groupId>
      <artifactId>central-publishing-maven-plugin</artifactId>
      <version>0.7.0</version>
      <extensions>true</extensions>
    </plugin>
  </plugins>
</build>`
          "beforeNote" "Plus a Sonatype account, a GPG key published to a keyserver, credentials in settings.xml, and the staging close & release dance."
          "afterLabel" "Latte"
          "afterLang" "bash"
          "afterCode" `latte login

# Create a Group at app.lattejava.org
# (verify your domain for a reverse-DNS Group)

latte release`
          "afterNote" "release tags the version from project.latte and publishes to the Latte repository."
        ) }}
      </div>
    </section>

    <!-- 3. Bloated libraries -->
    <section class="border-b border-slate-200 dark:border-slate-800">
      <div class="max-w-5xl mx-auto px-4 sm:px-6 lg:px-8 py-16 sm:py-20">
        <h2 class="text-2xl sm:text-3xl font-bold text-slate-900 dark:text-white">Libraries shouldn't drag in the world</h2>
        <p class="mt-4 text-lg text-slate-600 dark:text-slate-400 max-w-3xl">
          Add one library and inherit a dozen more — Jackson, Bouncy Castle, Commons, Guava — each
          a version conflict and a chunk of supply-chain surface you now own. Latte's libraries ship
          with zero runtime dependencies. Pure Java, just add a VM.
        </p>
        {{ partial "why-compare.html" (dict
          "beforeLabel" "A typical JWT library — mvn dependency:tree"
          "beforeLang" "text"
          "beforeCode" `com.auth0:java-jwt:4.4.0
\- com.fasterxml.jackson.core:jackson-databind:2.15.2
   +- com.fasterxml.jackson.core:jackson-annotations:2.15.2
   \- com.fasterxml.jackson.core:jackson-core:2.15.2`
          "beforeNote" "Every transitive jar is a version you must reconcile and a dependency you must trust."
          "afterLabel" "Latte JWT — mvn dependency:tree"
          "afterLang" "text"
          "afterCode" `org.lattejava:jwt:0.1.1
(no dependencies)`
          "afterNote" "No Jackson, Bouncy Castle, Apache Commons, or Guava. The http server is zero-dependency too."
        ) }}
      </div>
    </section>

    <!-- 4. Heavyweight frameworks -->
    <section class="border-b border-slate-200 dark:border-slate-800 bg-slate-50 dark:bg-slate-900">
      <div class="max-w-5xl mx-auto px-4 sm:px-6 lg:px-8 py-16 sm:py-20">
        <h2 class="text-2xl sm:text-3xl font-bold text-slate-900 dark:text-white">Frameworks shouldn't be heavyweight</h2>
        <p class="mt-4 text-lg text-slate-600 dark:text-slate-400 max-w-3xl">
          A "hello world" web service shouldn't need annotation magic, classpath scanning, a
          reflection-driven container, and a cold start to match. Latte <code>web</code> is a real
          server in a handful of lines, built on virtual threads, with no reflection.
        </p>
        {{ partial "why-compare.html" (dict
          "beforeLabel" "Spring Boot"
          "beforeLang" "java"
          "beforeCode" `@SpringBootApplication
public class DemoApplication {
  public static void main(String[] args) {
    SpringApplication.run(DemoApplication.class, args);
  }
}

@RestController
class HelloController {
  @GetMapping("/")
  public String hello() {
    return "Hello, world!";
  }
}`
          "beforeNote" "Plus spring-boot-starter-web and its transitive graph, component scanning, and a slower cold start."
          "afterLabel" "Latte web"
          "afterLang" "java"
          "afterCode" `import module org.lattejava.http;
import module org.lattejava.web;

void main() {
  new Web()
      .get("/", (req, res) -> res.getWriter().write("Hello, world!"))
      .start(8080);
}`
          "afterNote" "The underlying http server matches or beats Netty, Jetty, and Tomcat in our benchmarks."
        ) }}
      </div>
    </section>

    <!-- 5. Ceremony -->
    <section class="border-b border-slate-200 dark:border-slate-800">
      <div class="max-w-5xl mx-auto px-4 sm:px-6 lg:px-8 py-16 sm:py-20">
        <h2 class="text-2xl sm:text-3xl font-bold text-slate-900 dark:text-white">Ceremony for the simplest things</h2>
        <p class="mt-4 text-lg text-slate-600 dark:text-slate-400 max-w-3xl">
          For years, printing a line of text meant a public class, a static main method, and a
          String array you never used. Java 25 fixed the language; Latte leans all the way in —
          and <code>javaenv</code> means you're never hand-juggling JDK installs to get there.
        </p>
        {{ partial "why-compare.html" (dict
          "beforeLabel" "Classic Java"
          "beforeLang" "java"
          "beforeCode" `public class Main {
  public static void main(String[] args) {
    System.out.println("Hello, world!");
  }
}`
          "beforeNote" "Four lines of scaffolding around one line of work."
          "afterLabel" "Java 25 + Latte"
          "afterLang" "java"
          "afterCode" `void main() {
  IO.println("Hello, world!");
}`
          "afterNote" "javaenv install 25 && javaenv global 25 — no SDK juggling to get on the latest JDK."
        ) }}
      </div>
    </section>

  </div>
```

- [ ] **Step 2: Add the copy-button script before `{{ end }}`**

In `layouts/why/list.html`, add this script immediately before the final `{{ end }}` (after the closing CTA `</section>`):

```go-html-template
  <script>
  (function() {
    var blocks = document.querySelectorAll('#why-content .highlight');
    blocks.forEach(function(block) {
      var pre = block.querySelector('pre');
      if (!pre) return;
      var code = pre.querySelector('code');
      var text = (code || pre).textContent.trim();
      block.style.position = 'relative';
      var btn = document.createElement('button');
      btn.className = 'copy-btn';
      btn.setAttribute('aria-label', 'Copy to clipboard');
      btn.innerHTML = '<svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><rect x="9" y="9" width="13" height="13" rx="2"/><path d="M5 15H4a2 2 0 01-2-2V4a2 2 0 012-2h9a2 2 0 012 2v1"/></svg>';
      btn.addEventListener('click', function() {
        navigator.clipboard.writeText(text).then(function() {
          btn.innerHTML = '<svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7"/></svg>';
          setTimeout(function() {
            btn.innerHTML = '<svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><rect x="9" y="9" width="13" height="13" rx="2"/><path d="M5 15H4a2 2 0 01-2-2V4a2 2 0 012-2h9a2 2 0 012 2v1"/></svg>';
          }, 2000);
        });
      });
      block.appendChild(btn);
    });
  })();
  </script>
```

- [ ] **Step 3: Build and verify all sections render**

Run: `hugo --gc`
Expected: exits 0, no `ERROR`/`WARN`.

- [ ] **Step 4: Confirm each section and a sample of before/after content rendered**

Run:
```bash
for s in "second job" "take a weekend" "drag in the world" "heavyweight" "simplest things"; do
  printf '%s: ' "$s"; grep -c "$s" public/why/index.html
done
```
Expected: each line prints `1` or greater.

Run: `grep -c "central-publishing-maven-plugin" public/why/index.html && grep -c "IO.println" public/why/index.html`
Expected: both `1` or greater (confirms a "before" and an "after" snippet rendered through the partial + highlighter).

- [ ] **Step 5: Commit**

```bash
git add layouts/why/list.html layouts/partials/why-compare.html
git commit -m "feat: add Why Latte problem/solution sections and compare partial"
```

---

## Task 4: Navigation link

Add a "Why Latte?" link to the desktop nav (before the Projects dropdown) and to the mobile menu.

**Files:**
- Modify: `layouts/partials/nav.html`

- [ ] **Step 1: Add the desktop nav link**

In `layouts/partials/nav.html`, find the desktop nav container opening:

```go-html-template
      <!-- Desktop nav -->
      <div class="hidden md:flex items-center gap-6">
        <!-- Projects dropdown -->
        <div class="relative" id="projects-dropdown">
```

Insert the link between `gap-6">` and `<!-- Projects dropdown -->`:

```go-html-template
      <!-- Desktop nav -->
      <div class="hidden md:flex items-center gap-6">
        <a href="/why" class="text-sm font-medium text-slate-600 dark:text-slate-300 hover:text-slate-900 dark:hover:text-white transition-colors">Why Latte?</a>
        <!-- Projects dropdown -->
        <div class="relative" id="projects-dropdown">
```

- [ ] **Step 2: Add the mobile menu link**

In the same file, find the mobile menu's projects heading:

```go-html-template
    <div class="px-4 py-3 space-y-1">
      <div class="text-xs font-semibold uppercase tracking-wider text-slate-400 dark:text-slate-500 px-2 py-1">Projects</div>
```

Insert the link between `space-y-1">` and the `Projects` heading div:

```go-html-template
    <div class="px-4 py-3 space-y-1">
      <a href="/why" class="block px-2 py-2 text-sm font-medium text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 rounded">Why Latte?</a>
      <div class="text-xs font-semibold uppercase tracking-wider text-slate-400 dark:text-slate-500 px-2 py-1">Projects</div>
```

- [ ] **Step 3: Build and verify the link appears on another page**

Run: `hugo --gc`
Expected: exits 0, no errors.

Run: `grep -c 'href="/why"' public/index.html`
Expected: `2` (one desktop, one mobile — the nav renders on every page, including the homepage).

- [ ] **Step 4: Commit**

```bash
git add layouts/partials/nav.html
git commit -m "feat: link Why Latte page from the nav"
```

---

## Task 5: Homepage hero button + callout banner

Add a "Why Latte?" secondary button to the homepage hero button row, and a callout banner section linking to the page.

**Files:**
- Modify: `layouts/index.html`

- [ ] **Step 1: Add the hero button**

In `layouts/index.html`, find the hero button row:

```go-html-template
        <div class="mt-10 flex items-center justify-center gap-4">
          <a href="#projects" class="px-6 py-3 bg-accent-500 hover:bg-accent-600 text-white font-medium rounded-lg transition-colors">
            Explore Projects
          </a>
          <a href="{{ .Site.Params.github }}" class="px-6 py-3 border border-slate-300 dark:border-slate-700 text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 font-medium rounded-lg transition-colors" target="_blank" rel="noopener">
            GitHub
          </a>
        </div>
```

Add a "Why Latte?" button as the second item (between Explore Projects and GitHub):

```go-html-template
        <div class="mt-10 flex items-center justify-center gap-4">
          <a href="#projects" class="px-6 py-3 bg-accent-500 hover:bg-accent-600 text-white font-medium rounded-lg transition-colors">
            Explore Projects
          </a>
          <a href="/why" class="px-6 py-3 border border-slate-300 dark:border-slate-700 text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 font-medium rounded-lg transition-colors">
            Why Latte?
          </a>
          <a href="{{ .Site.Params.github }}" class="px-6 py-3 border border-slate-300 dark:border-slate-700 text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 font-medium rounded-lg transition-colors" target="_blank" rel="noopener">
            GitHub
          </a>
        </div>
```

- [ ] **Step 2: Add the callout banner section**

In `layouts/index.html`, find the start of the Getting Started section:

```go-html-template
  <!-- Getting Started -->
  <section class="border-t border-slate-200 dark:border-slate-800 bg-slate-50 dark:bg-slate-900">
```

Insert this banner section immediately before that `<!-- Getting Started -->` comment:

```go-html-template
  <!-- Why Latte banner -->
  <section class="border-t border-slate-200 dark:border-slate-800">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12">
      <a href="/why" class="group flex flex-col sm:flex-row sm:items-center justify-between gap-4 p-6 rounded-2xl bg-accent-500/5 dark:bg-accent-400/10 border border-accent-500/20 dark:border-accent-400/20 hover:border-accent-500/50 dark:hover:border-accent-400/50 transition-colors">
        <div>
          <h2 class="text-xl font-semibold text-slate-900 dark:text-white">Why Latte?</h2>
          <p class="mt-1 text-slate-600 dark:text-slate-400">Java is great. Its tooling became a second job. See the case for a simpler way.</p>
        </div>
        <span class="inline-flex items-center gap-2 text-accent-600 dark:text-accent-400 font-medium shrink-0">
          Read more
          <svg class="w-4 h-4 group-hover:translate-x-1 transition-transform" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M9 5l7 7-7 7" /></svg>
        </span>
      </a>
    </div>
  </section>

```

- [ ] **Step 3: Build and verify the homepage links and banner**

Run: `hugo --gc`
Expected: exits 0, no errors.

Run: `grep -c 'href="/why"' public/index.html`
Expected: `4` (2 from nav added in Task 4, plus the hero button and the banner).

Run: `grep -c "the case for a simpler way" public/index.html`
Expected: `1`.

- [ ] **Step 4: Commit**

```bash
git add layouts/index.html
git commit -m "feat: link Why Latte page from the homepage hero and banner"
```

---

## Task 6: Final full-site verification

Confirm the whole site builds clean and the page is complete, then do a visual pass.

- [ ] **Step 1: Clean production build**

Run: `hugo --gc --minify`
Expected: exits 0 with no `ERROR` or `WARN` lines; the summary lists `why/index.html` among rendered pages (Pages count increased by 1 vs. before the feature).

- [ ] **Step 2: Verify all five comparisons rendered through the highlighter**

Run:
```bash
for s in "maven-compiler-plugin" "central-publishing-maven-plugin" "jackson-databind" "@RestController" "IO.println"; do
  printf '%s: ' "$s"; grep -c "$s" public/why/index.html
done
```
Expected: every line prints `1` or greater (one anchor string from each of the five before/after pairs).

- [ ] **Step 3: Visual check in the browser**

Run: `hugo server` and open `http://localhost:1313/why/`.

Confirm by eye:
- Hero, all five problem→solution sections (alternating white / slate-50 backgrounds), and the closing CTA render in order.
- Each comparison is side-by-side on a wide window and stacks to one column when narrowed to mobile width.
- "Before" headers are rose-tinted; "After" headers are accent-colored.
- Code blocks are syntax-highlighted and show a copy button on hover that copies the snippet.
- Toggle dark/light (top-right) — all sections, borders, and code blocks adapt.
- "Why Latte?" appears in the top nav (and in the mobile hamburger menu at a narrow width) and links here; the homepage hero button and banner also link here.

Stop the server (Ctrl-C) when done.

- [ ] **Step 4: Final commit (only if Step 3 surfaced fixes)**

If the visual check required changes, commit them:

```bash
git add -A
git commit -m "fix: Why Latte page visual adjustments"
```

If no changes were needed, this task adds no commit.

---

## Self-Review notes

- **Spec coverage:** Hero + 5 problem→solution pairs (Tasks 1, 3) ✓; before/after side-by-side comparisons (Task 2 partial, Task 3) ✓; nav link desktop+mobile (Task 4) ✓; homepage hero button + banner (Task 5) ✓; alternating section backgrounds / accent + rose treatment / dark mode / copy buttons (Tasks 2–3) ✓; claims traced to existing content, no invented numbers (snippets use real plugin/library names and the benchmark claim is stated qualitatively per `content/http/_index.md`) ✓; clean build success criterion (Task 6) ✓.
- **No big comparison table, no `compare` shortcode, no new design tokens** beyond stock `rose-*` — consistent with the spec's "Out of Scope".
- **Type/name consistency:** the partial is `layouts/partials/why-compare.html`, invoked as `partial "why-compare.html"` in every section; the content container id `why-content` is defined in Task 1 and targeted by the copy script in Task 3; the section layout is `layouts/why/list.html` selected by `layout: why` front matter in `content/why/_index.md`.
```
