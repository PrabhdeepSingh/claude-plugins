---
name: seo-standards
description: Best-in-class technical SEO for any web page. INVOKE THIS PROACTIVELY — even when the user never says "SEO" — whenever creating or editing any of these — HTML, JSX/TSX, Vue, Svelte, Astro, or other page/component templates; route or URL definitions; redirects (301/302/410); `<head>` metadata (title, meta description, canonical, robots, hreflang, Open Graph); structured data / schema.org JSON-LD; XML sitemaps or robots.txt; or anything served as a web page or that affects how one is crawled or indexed. Covers heading structure, canonical tags, title/meta length, URL strategy, redirect rules, schema markup, render-blocking JS/CSS, and indexation controls. It is the baseline SEO quality bar for all web output produced here, so reach for it on any web-page or frontend task — not only when SEO is named. For the WRITING itself (prose, articles, landing copy), use the [[content-seo]] sibling instead — this skill is the plumbing, not the editorial.
---

# SEO Standards — build it like Google would reward it

These rules exist because SEO is cheaper to bake in than to bolt on. A correct heading structure costs nothing at build time; fixing it after launch requires a crawl, a redeploy, and weeks of re-indexing. Every rule below optimizes for the crawler and the human reader simultaneously — they almost always want the same thing.

## When to apply this

Any time you are writing or reviewing: HTML templates, page components, routing configuration, redirect rules, `<head>` metadata, sitemaps, robots.txt, structured data, CMS templates, or URL patterns. If the output will be served as a web page or affect how one is indexed, this skill applies.

---

## 1. HTML structure

- **One `<h1>` per URL.** It should be the main on-page heading inside `<body>`, associated with the page's focus keyword. Never use it for a site name, nav label, or logo.
- **Multiple `<h2>`/`<h3>` are fine.** Heading tags are for content hierarchy, not navigation elements.
- **Heading tags (`<h1>`–`<h6>`) must not be used for navigation.** Nav links that happen to be styled large are not headings.
- **Avoid useless anchor text.** "Read more," "click here," "view product," "visit page" tell neither the user nor the crawler what the destination is. Use descriptive text.
- **Don't pile redundant links to the same URL.** A few repeats are normal and fine (a nav link plus a body link, an image plus its caption). But when one listing links to the same destination four times (image, title, price, "view details"), trim it — Google has never defined which anchor counts when links repeat (first, last, or some blend), so redundant anchors add clutter and markup weight without adding any signal you can rely on.
- **Every page needs enough indexable body text in the initial HTML to establish what it's about** — ideally with the key content above the fold, not behind JavaScript execution. There is no minimum word count (Google explicitly says so); "enough" is a quality judgment: a page whose HTML carries only a heading and boilerplate gives the crawler nothing to rank.
- **Text shown in images must also exist as real HTML text.** Crawlers cannot read image-embedded text.
- **Build navigation from real `<a href>` links, not controls.** Crawlers follow anchors but may not treat `<button>`, form submissions, or JS click handlers as links — so don't use those for navigation. (An `<a href>` is perfectly crawlable even inside a `<form>`.)

## 2. Page titles and meta descriptions

- **Page title: unique per URL, 50–60 characters (including spaces).**
- **Meta description: unique per URL, ~150–160 characters** (Google truncates around there; ~120 shows on mobile).
- **Only one `<title>` element per page.**
- Both should be manageable at the page level in any CMS without a deploy.

## 3. Canonical tags

- **Every URL must carry a `rel="canonical"` tag.**
- The canonical must use an **absolute `https://` URL** — never `http://`, never relative.
- It must point to the page's own canonical URL (not a duplicate, not a parameterized variant).

## 4. URL strategy

- **Keep the canonical URL clean — no parameters baked into it.** Campaign tracking uses standard query params (`?utm_source=…`); just point `rel=canonical` at the clean, parameter-free URL so the tracked variants aren't indexed as duplicates. Don't stuff tracking into the `#fragment` — analytics tools don't read it.
- **All URLs forced lowercase** — serve a `301` from mixed-case to lowercase.
- **Pick ONE trailing-slash convention and enforce it site-wide** — `301` the other variant to the chosen one. Google treats `/page` and `/page/` as different URLs; what matters is that exactly one form is canonical and reachable, not which form you pick. (Whichever you choose, be consistent across the site, the sitemap, and every internal link.)
- **No extraneous folders** like `/cms/`, `/content/`, `/app/` in public-facing URLs.
- **Hyphens as delimiters**, never underscores or spaces.
- **Flat category structure preferred** — avoid deep nesting of sub-folders.
- All non-alphanumeric characters stripped; forward slashes and periods in product names replaced with hyphens; double spaces collapsed to a single hyphen.
- **URLs should not expose dates** (no `/2020/08/` style WordPress paths unless meaningful to the content).

## 5. Domain and HTTPS

- All pages live on the single canonical hostname (e.g. `https://www.example.com/`) — no sub-domains, ccTLDs, or microsites for content that should benefit SEO.
- `http://` must redirect to `https://` with a `301`, enforced via HSTS.
- No content should be accessible on more than one hostname.

## 6. Redirects

- **Use `301` (permanent) for every permanent move** — URL changes, canonicalization, `http`→`https`, lowercase/trailing-slash enforcement. A `301` is the clear, unambiguous signal to move ranking to the new URL; a `302`/`307` says "temporary — keep the old URL," which is the wrong signal for a permanent move (Google may eventually treat a long-lived `302` as permanent, but don't leave it to that guesswork). Reserve `302`/`307` for genuinely temporary redirects (A/B tests, short-lived promos, maintenance). Never use JavaScript or meta-refresh redirects for either.
- **No redirect chains.** Every redirect must point directly to the final destination URL.
- **Legacy URLs with no relevant match** should `301` to the most relevant page.
- **Deprecated URLs with no replacement**: prefer `410 Gone` — it says "deliberately removed" and drops out of the index marginally faster. A correct `404` is also acceptable (Google treats the two nearly identically); what's wrong is a `301` to an irrelevant page (a soft redirect Google ignores) or a `200` rendering an error page.
- When a URL is changed in the CMS, all internal links generated by the CMS (navigation, breadcrumbs) must update automatically to the new URL.

## 7. Schema.org markup

- Mark up every applicable on-page element with **JSON-LD** — Google's recommended format and the easiest to maintain. (Google still parses Microdata and RDFa, but prefer JSON-LD for new work.)
- **Prioritize types that earn rich results** — markup exists to unlock search features, not as decoration. Bare `WebPage` markup on every page drives no rich result; add it only if your tooling wants a consistent graph, and spend the effort on the types below instead.
- Common types to include where content exists:
  - `Product` — express pricing as a nested `Offer` (or `AggregateOffer`) using the `price` and `priceCurrency` properties (there is no `Price` or `Currency` type)
  - `ImageObject`
  - `AggregateRating`
  - `VideoObject`
  - `BreadcrumbList`
- Schema goes in a `<script type="application/ld+json">` block — `<head>` or `<body>` both work (Google reads either, including JS-injected); `<head>` is the tidier convention.

## 8. Internal linking

- Every link must carry meaningful, machine-readable text — visible anchor text, or a descriptive `alt`/`aria-label` when the link wraps an image or icon. Avoid links whose only content is an undescribed image.
- Links must point to the canonical URL of the destination — not to a URL that redirects, is parameterized, or is a known duplicate.
- Don't pepper a page with many redundant links to the same destination — a natural repeat (nav + contextual) is fine; trim boilerplate duplication.
- Prefer **absolute URLs** in navigation — not because relative URLs can't be crawled (they can), but because they're unambiguous about the canonical destination and sidestep relative-path duplication bugs.
- Navigation must be HTML/CSS-renderable without JavaScript — menus must degrade gracefully for crawlers.

## 9. JavaScript and content rendering

- **Content that needs to be indexed must not require JavaScript to render.** Googlebot does eventually execute JS, but it's slower, less reliable, and risks content being missed during crawl budget. Indexable body text, headings, and links must exist in the initial HTML response.
- All non-critical JavaScript externalized to files, not inline `<script>` blocks.
- All JS files minified.
- Remove render-blocking JS and CSS from above-the-fold content.

## 10. CSS and performance

- All non-critical CSS externalized — no inline `<style>` blocks for anything non-dynamic.
- All CSS files minified.
- No whitespace in outputted HTML (minify HTML on production).
- No developer comments in rendered HTML.
- No commented-out code in rendered HTML.
- Images served efficiently: modern formats (WebP/AVIF) where supported, correctly sized (`srcset`/`sizes`), lazy-loaded below the fold (`loading="lazy"`), with long-lived caching. (Do NOT resurrect the old "cookie-free domain" advice — under HTTP/2+ a separate image hostname costs an extra connection and hurts more than cookie bytes ever did.)
- Browser caching enabled.
- HTTP/2 (or HTTP/3) and HSTS enabled.

## 11. Indexation and crawl controls

- `robots.txt` editable by the SEO team outside of release cycles.
- Internal search result URLs, filter/facet URLs beyond the canonical sub-category, and sort/parameter URLs: `META NOINDEX, FOLLOW`. Know what this actually buys: `FOLLOW` lets the crawler discover links on the page *initially*, but Google has said a page that stays `noindex` long-term is eventually treated as `noindex, nofollow` — its links stop being followed. So `noindex, follow` is the right tool for keeping junk URLs out of the index, **not** a durable way to route link equity through them; make sure everything important is also reachable through indexable navigation.
- Development, staging, and test environments: login-gated AND blocked by `robots.txt`. The staging `robots.txt` **must never be promoted to production**.
- During maintenance/downtime: serve `503` (not `404`).
- Beta/preview releases: hosted at a known path on the main domain (e.g. `/beta/`) and blocked from indexing, not on a separate hostname.

## 12. 404 and gone pages

- `404` pages must serve an actual `404` status code (not `200`).
- `404` page should contain helpful internal links: popular pages, product pages, "back to home," "back to previous page."
- Retired URLs with no replacement: `410 Gone`.

## 13. XML sitemaps

- Auto-generated XML sitemaps covering all indexable product and category pages.
- Ability to manually create and upload sitemap documents.
- Sitemap referenced correctly in `robots.txt`.
- Video sitemap generated if video content exists.

## 14. CMS requirements

Any CMS used must provide per-page manual control of:
- `<title>` (with template-driven defaults)
- `rel="canonical"`
- `meta robots` (NOINDEX/NOFOLLOW overrides)
- `rel=alternate hreflang` (if multi-language)

---

## Self-check before shipping any web change

Run this against your diff or template. Fix any "no" before finishing:

- Is there exactly one `<h1>` per page, connected to the focus keyword and not used in nav?
- Is `rel="canonical"` present on every URL, using an absolute `https://` URL pointing to itself?
- Is the `<title>` unique, 50–60 chars, and present exactly once?
- Is the meta description unique and ~150–160 chars?
- Are all URLs lowercase, consistent with the site's single trailing-slash convention, parameter-free, and without extraneous folders?
- Does every *permanent* move use a `301` (not a `302`/`307`) and point directly to the final URL with no chains?
- Is every anchor using descriptive text (no "read more"/"click here") and pointing to a canonical URL, without piling on redundant duplicate links to the same destination?
- Is all indexable body content in the initial HTML response (not JavaScript-rendered)?
- Is schema.org markup present as JSON-LD for every applicable on-page element?
- Is non-critical JS/CSS externalized and minified? No inline styles, no `<script>` blocks except JSON-LD?
- Is rendered HTML free of whitespace, developer comments, and commented-out code?
- Are filter/facet/sort URLs carrying `META NOINDEX, FOLLOW`?
- Do `404` pages serve status code `404` with helpful internal links?
- Are deprecated, unreplaceable URLs returning `410 Gone` (or a correct `404`) — never a `200` error page or a `301` to an irrelevant destination?
- Is the staging environment login-gated and blocked in `robots.txt`, with its own `robots.txt` never promoted to production?

---

## Provenance and maintenance

Search-engine behavior drifts. The claims here were last verified against Google Search Central documentation in **2026-07**. Before leaning hard on any of these in a dispute, re-verify with a web search of Google's current docs:

- Trailing-slash / URL-consistency guidance (§4) — "Google URL structure best practices".
- No-minimum-word-count and thin-content treatment (§1) — "Google word count ranking".
- Long-term `noindex, follow` degrading to `nofollow` (§11) — "Google long-term noindex nofollow".
- `404` vs `410` handling (§6, §12) — "Google 404 vs 410 difference".
- JSON-LD placement and rich-result-eligible types (§7) — "Google structured data guidelines".
- Title/meta display lengths (§2) — these are SERP-rendering behaviors that shift; the 50–60 / 150–160 character targets are the durable rule of thumb. **These numbers' canonical home is this file** — [[content-seo]] references them; if they change, change them here first.
