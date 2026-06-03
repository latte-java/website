# "Why Latte?" Page Design Spec

## Overview

A new marketing/positioning page at `/why/` that argues why Latte exists: Java the
language is great, but the tooling and ecosystem that grew around it became a second
job. Latte brings the simplicity back. The page is confident and opinionated — it names
Maven, Gradle, Sonatype/Maven Central, and Spring directly and leans on real,
defensible pain rather than strawmen.

It links from both the top navigation and the homepage.

## Voice & Thesis

- **Thesis:** Java the language is great; the tooling and ecosystem around it became a
  second job. Latte uncomplicates it.
- **Register:** Punchy and opinionated, but credible. Name names. Every claim must be
  defensible (no invented benchmarks, no strawman config). Where a competitor's
  complexity exists for a reason, the copy can acknowledge it briefly, but the page is
  unapologetically a case *for* Latte.

## Page Structure

Hero + 5 problem→solution pairs + closing CTA.

1. **Hero** — headline (working draft: *"Java got complicated. We're uncomplicating
   it."*), one-paragraph thesis. Optional logo per homepage idiom.

2. **Dependency management is a second job**
   - Problem: XML walls; the Groovy-vs-Kotlin DSL maze; plugin soup; transitive
     dependency conflicts.
   - Solution: one readable `project.latte`, `latte install`, `latte build`.
   - Before/after: a representative `pom.xml` (or `build.gradle`) dependency + plugin
     block vs. the equivalent `project.latte`.

3. **Publishing an artifact shouldn't take a weekend**
   - Problem: Sonatype/Maven Central account, GPG signing, sources + javadoc jars, POM
     metadata requirements, the nexus-staging "close & release" dance.
   - Solution: `latte login` → create a Group (verify domain for reverse-DNS groups) →
     `latte release`.
   - Before/after: the Maven Central signing/staging plugin config vs. `latte release`.

4. **Libraries shouldn't drag in the world**
   - Problem: add one JWT library, inherit Jackson, Bouncy Castle, Apache Commons,
     Guava — version conflicts plus supply-chain surface.
   - Solution: Latte `jwt` and `http` ship **zero runtime dependencies** (pure Java).
   - Before/after: a typical transitive dependency tree vs. "no dependencies."

5. **Frameworks shouldn't be heavyweight**
   - Problem: Spring Boot's large dependency graph, annotation/reflection magic,
     classpath scanning, slower startup.
   - Solution: Latte `web` — a working server in a handful of lines, built on virtual
     threads, no reflection. Cite the real `http` benchmark numbers
     (`content/http/_index.md`).
   - Before/after: a Spring `@RestController` + boilerplate vs. the Latte `web`
     class-less `void main()` example.

6. **Ceremony for trivial things**
   - Problem: `public class Main { public static void main(String[] args) }`;
     juggling JDK versions by hand.
   - Solution: Java 25 class-less `void main`, module imports, and `javaenv` for version
     switching.
   - Before/after: classic class-based main vs. class-less `void main()`.

7. **Closing CTA** — "Get started in 5 minutes" linking to the homepage quick-start and
   the GitHub org.

### Claim sourcing

All concrete claims come from existing content so nothing is invented:
- Zero-dependency claims: `content/jwt/_index.md`, `content/http/_index.md`.
- Benchmark numbers: the performance table in `content/http/_index.md` (cite date/HW as
  shown there).
- Publishing flow (`latte login`, Group creation, domain verification, `latte release`):
  `content/_index.md`.
- `project.latte` / `dependency(id: ...)` / `latte install` / `latte build`:
  `content/cli/_index.md`, `content/web/_index.md`, `content/http/_index.md`.
- Class-less `void main` and module imports: `content/web/_index.md`.

Competitor-side before/after snippets (pom.xml, Gradle, Maven Central signing/staging,
Spring controller) are standard, widely-recognized configurations — kept short,
representative, and accurate.

## Visual Treatment

Matches the existing homepage idiom (`layouts/index.html`):

- Alternating section backgrounds (`bg-white` / `bg-slate-50 dark:bg-slate-900`) with
  `border-t border-slate-200 dark:border-slate-800` dividers.
- `max-w-7xl` / `max-w-3xl` containers, `accent-500` / `accent-400` for emphasis and
  links, slate text scale consistent with the rest of the site.
- Each before/after is a responsive two-column grid (`grid grid-cols-1 md:grid-cols-2
  gap-6`) that stacks on mobile. The **Before** column is labeled and tinted muted-red
  (e.g. a `rose`/`red` header chip); the **After** column is labeled in the accent
  color.
- Reuse the existing copy-to-clipboard button script pattern used in
  `layouts/index.html` / `layouts/_default/landing.html` for the code blocks.
- Full dark/light support via the existing `dark:` variants.

## Implementation

### Content
- New section: `content/why/_index.md` → URL `/why/`.
- Front matter: `title: "Why Latte?"`, a `description`, and `layout` selecting the
  bespoke template (or rely on the section's `list.html`). Minimal/no markdown body —
  copy lives in the template.

### Layout
- A **bespoke layout** authored the way `layouts/index.html` already is: the section
  scaffolding and the prose/code blocks live in the template for full control over the
  alternating sections and side-by-side comparison grids.
- File: `layouts/why/list.html` (section index template for the `why` section), defining
  the `main` block per `baseof.html` conventions.
- Rationale: the page is heavily structured (alternating problem/solution blocks +
  side-by-side code). A bespoke template matches the homepage precedent and avoids
  fighting `prose` styling. (Considered and rejected for now: markdown + a custom
  `compare` shortcode — more editable, but works against the heavy custom layout.)

### Navigation
- Add a `Why Latte?` link to `layouts/partials/nav.html`, placed before the Projects
  dropdown, in both the desktop nav and the mobile menu, styled like the existing nav
  links.

### Homepage
- In `layouts/index.html`: add a "Why Latte?" CTA. A secondary button in the hero
  button row (alongside "Explore Projects" / "GitHub") linking to `/why/`, plus a short
  callout/banner section linking to the page. Keep consistent with existing hero/section
  styling.

## Out of Scope

- No big side-by-side comparison/feature table (the page uses problem→solution pairs).
- No new design tokens, colors, or fonts beyond what the site already defines (reuse
  `accent`, `slate`, and add only a muted red for the "Before" label if not already
  present in the Tailwind config).
- No changes to other project pages or docs.
- No `compare` shortcode (template-authored approach chosen).

## Success Criteria

- `/why/` renders with all 5 problem→solution pairs, hero, and CTA, in both light and
  dark mode, responsive (side-by-side on desktop, stacked on mobile).
- "Why Latte?" appears in the top nav (desktop + mobile) and is linked from the
  homepage.
- All factual claims trace to existing content; no invented numbers.
- `hugo` builds cleanly with no template errors.
