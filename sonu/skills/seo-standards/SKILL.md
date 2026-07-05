---
name: seo-standards
description: Best-in-class technical SEO for any web page. INVOKE whenever creating or editing HTML/JSX/TSX/Vue/Svelte/Astro templates; route or URL definitions; redirects; `<head>` metadata (title, meta description, canonical, robots, hreflang, Open Graph); structured data/JSON-LD; sitemaps or robots.txt; or anything served as a web page or that affects how one is crawled or indexed — even when nobody says "SEO." For the WRITING itself (prose, articles, landing copy), use the [[content-seo]] sibling instead — this skill is the plumbing, not the editorial.
---

# SEO Standards — build it like Google would reward it

These rules exist because SEO is cheaper to bake in than to bolt on. A correct heading structure costs nothing at build time; fixing it after launch requires a crawl, a redeploy, and weeks of re-indexing. Every rule below optimizes for the crawler and the human reader simultaneously — they almost always want the same thing.

## When to apply this

Any time you are writing or reviewing: HTML templates, page components, routing configuration, redirect rules, `<head>` metadata, sitemaps, robots.txt, structured data, CMS templates, or URL patterns. If the output will be served as a web page or affect how one is indexed, this skill applies.

---

## 1. HTML structure

- **One `<h1>` per URL**, the main on-page heading inside `<body>`, tied to the page's focus keyword — never a site name, nav label, or logo.
- **Multiple `<h2>`/`<h3>` are fine**; heading tags are for content hierarchy, not navigation elements.
- **Avoid useless anchor text** ("read more," "click here") — it tells neither the user nor the crawler what the destination is.
- **Don't pile redundant links to the same URL.** A repeat or two is normal (nav + body link); four links to one destination from one listing is clutter with no reliable added signal.
- **Every page needs enough indexable body text in the initial HTML to establish what it's about**, ideally above the fold. No minimum word count exists — "enough" is a quality judgment; a page whose HTML carries only a heading and boilerplate gives the crawler nothing to rank.
- **Text shown in images must also exist as real HTML text** — crawlers cannot read image-embedded text.
- **Build navigation from real `<a href>` links, not controls** — crawlers may not treat buttons or JS click handlers as links.

## 2. Page titles and meta descriptions

- **Title: unique per URL, 50–60 characters.** Only one `<title>` per page.
- **Meta description: unique per URL, ~150–160 characters** (Google truncates there; ~120 shows on mobile).
- Both manageable at the page level in any CMS without a deploy.

## 3. Canonical tags

Every URL must carry `rel="canonical"`, using an **absolute `https://` URL** — never relative — pointing to the page's own canonical URL, not a duplicate or parameterized variant.

## 4. URL strategy

- **Keep the canonical URL clean — no parameters baked in.** Point `rel=canonical` at the parameter-free URL so tracked variants aren't indexed as duplicates.
- **Force lowercase** via `301`; **pick one trailing-slash convention and enforce it site-wide** (`301` the other variant) — Google treats `/page` and `/page/` as different URLs, so what matters is exactly one canonical, consistently-used form.
- **No extraneous folders** (`/cms/`, `/app/`) in public URLs; **hyphens as delimiters**, never underscores or spaces; **flat category structure preferred**; **no exposed dates** unless meaningful to the content.

## 5. Domain and HTTPS

All pages on a single canonical hostname — no sub-domains/ccTLDs/microsites splitting SEO-relevant content. `http://` redirects to `https://` with a `301`, enforced via HSTS. No content reachable on more than one hostname.

## 6. Redirects

- **`301` for every permanent move** (URL changes, canonicalization, `http`→`https`, lowercase/slash enforcement) — a `301` is the unambiguous "move ranking here" signal; `302`/`307` says "temporary," the wrong signal for a permanent move. Reserve `302`/`307` for genuinely temporary redirects.
- **No redirect chains** — every redirect points directly to the final URL.
- **Legacy URLs with no relevant match** `301` to the most relevant page.
- **Deprecated URLs with no replacement**: prefer `410 Gone` (a correct `404` is also fine); never a `301` to an irrelevant page or a `200` rendering an error page.
- CMS-generated internal links (nav, breadcrumbs) must auto-update when a URL changes.

## 7. Schema.org markup

Mark up applicable elements with **JSON-LD** (Google's preferred format for new work). **Prioritize types that earn rich results** — markup exists to unlock search features, not as decoration; bare `WebPage` markup drives no rich result. Common types where content exists: `Product` (price via a nested `Offer`/`AggregateOffer`), `ImageObject`, `AggregateRating`, `VideoObject`, `BreadcrumbList`. Goes in a `<script type="application/ld+json">` block, `<head>` or `<body>` (either works; `<head>` is the tidier convention).

## 8. Internal linking

Every link carries meaningful, machine-readable text (visible anchor, or descriptive `alt`/`aria-label` when wrapping an image). Links point to the destination's canonical URL, not a redirecting/parameterized/duplicate one. Prefer **absolute URLs** in navigation for unambiguous canonical destinations. Navigation must be HTML/CSS-renderable without JavaScript.

## 9. JavaScript and content rendering

**Content that needs to be indexed must not require JavaScript to render** — Googlebot executes JS eventually, but slower and less reliably, risking missed content during crawl budget. Indexable body text, headings, and links must exist in the initial HTML response. Externalize and minify non-critical JS; remove render-blocking JS/CSS from above-the-fold content.

## 10. CSS and performance

Externalize and minify non-critical CSS — no inline `<style>` for anything non-dynamic. No whitespace, developer comments, or commented-out code in rendered HTML. Images: modern formats (WebP/AVIF), correctly sized (`srcset`/`sizes`), lazy-loaded below the fold, long-lived caching. Browser caching, HTTP/2+, and HSTS enabled. (Don't resurrect the old "cookie-free domain" advice — under HTTP/2+ it costs more than it saves.)

## 11. Indexation and crawl controls

- `robots.txt` editable by the SEO team outside release cycles.
- Internal search results, filter/facet URLs beyond the canonical sub-category, and sort/parameter URLs: `META NOINDEX, FOLLOW`. Know the limit: a page that stays `noindex` long-term is eventually treated as `noindex, nofollow` too — its links stop being followed — so this keeps junk out of the index but isn't a durable way to route link equity; keep everything important reachable through indexable navigation too.
- Dev/staging/test environments: login-gated **and** blocked by `robots.txt` — the staging `robots.txt` must never be promoted to production.
- Maintenance/downtime: serve `503` (not `404`). Beta/preview releases: a known path on the main domain (e.g. `/beta/`), blocked from indexing — not a separate hostname.

## 12. 404 and gone pages

`404` pages serve an actual `404` status (never `200`) and contain helpful internal links (popular pages, "back to home"). Retired URLs with no replacement: `410 Gone`.

## 13. XML sitemaps

Auto-generated, covering all indexable pages; manual upload also possible; referenced correctly in `robots.txt`; a video sitemap if video content exists.

## 14. CMS requirements

Any CMS must provide per-page manual control of `<title>` (with template defaults), `rel="canonical"`, `meta robots` overrides, and `rel=alternate hreflang` (if multi-language).

---

## Self-check before shipping any web change

Run this against your diff or template. Fix any "no" before finishing:

- Exactly one `<h1>` per page, tied to the focus keyword, not used in nav?
- `rel="canonical"` on every URL, absolute `https://`, pointing to itself?
- `<title>` unique, 50–60 chars, present exactly once? Meta description unique, ~150–160 chars?
- URLs lowercase, one consistent trailing-slash convention, parameter-free, no extraneous folders?
- Every permanent move a `301` (not `302`/`307`), pointing directly to the final URL, no chains?
- Every anchor descriptive (no "read more") and pointing to a canonical URL, without redundant duplicate links?
- All indexable body content in the initial HTML response, not JS-rendered?
- Schema.org JSON-LD present for every applicable element?
- Non-critical JS/CSS externalized and minified; no inline styles; rendered HTML free of whitespace/comments?
- Filter/facet/sort URLs carrying `META NOINDEX, FOLLOW`?
- `404` pages serve status `404` with helpful links; deprecated unreplaceable URLs return `410` (or a correct `404`) — never `200` or a `301` to an irrelevant page?
- Staging login-gated and blocked in `robots.txt`, with its own `robots.txt` never promoted to production?

---

## Provenance and maintenance

Search-engine behavior drifts. Claims here were last verified against Google Search Central documentation in **2026-07**. Before leaning hard on any of these in a dispute, re-verify with a web search of Google's current docs: trailing-slash/URL-consistency guidance (§4); no-minimum-word-count and thin-content treatment (§1); long-term `noindex, follow` degrading to `nofollow` (§11); `404` vs `410` handling (§6, §12); JSON-LD placement and rich-result-eligible types (§7); title/meta display lengths (§2 — **these numbers' canonical home is this file**; [[content-seo]] references them, so change them here first).
