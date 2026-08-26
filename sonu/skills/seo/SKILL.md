---
name: seo
description: >-
  SEO for anything served as a web page — the plumbing (templates, routes, redirects, <head> metadata, JSON-LD, sitemaps, robots.txt) and the prose (posts, guides, landing copy, docs) so pages rank and get cited by AI answer engines. INVOKE when creating or editing page templates (HTML/JSX/TSX/Vue/Svelte/Astro), routes, metadata, or any content meant for the web — even when nobody says "SEO". Interface microcopy is [[ux-writing]], not SEO prose.
---

# SEO — build it and write it so search rewards it

SEO is cheaper to bake in than to bolt on: a correct heading structure costs nothing at build time, while fixing it after launch takes a crawl, a redeploy, and weeks of re-indexing. And the writing carries the same property — a page that nails one search intent with extractable structure earns rankings and AI citations that no retrofit recovers. Every rule below optimizes for the crawler, the answer engine, and the human reader simultaneously; they almost always want the same thing. Sections 1–14 are the plumbing (templates, URLs, redirects, markup); sections 15–22 are the editorial layer (intent, structure, evidence). A template-only change uses the first half; a prose-only change the second; a new page both.

## When to apply this

Any time you write or review something served as a web page or affecting how one is indexed: HTML templates, page components, routing, redirect rules, `<head>` metadata, sitemaps, robots.txt, structured data, CMS templates, URL patterns — and any prose destined for the web (posts, articles, guides, landing copy, press releases, changelogs, FAQs, docs, Markdown with frontmatter), even when nobody said "SEO".

---

## Part A — the plumbing

## 1. HTML structure

- **One `<h1>` per URL**, the main on-page heading inside `<body>`, tied to the page's focus keyword — never a site name, nav label, or logo.
- **Multiple `<h2>`/`<h3>` are fine**; heading tags are for content hierarchy, not navigation elements.
- **Avoid useless anchor text** ("read more," "click here") — it tells neither the user nor the crawler what the destination is.
- **Don't pile redundant links to the same URL.** A repeat or two is normal (nav + body link); four links to one destination from one listing is clutter with no reliable added signal.
- **Every page needs enough indexable body text in the initial HTML to establish what it's about**, ideally above the fold. No minimum word count exists — "enough" is a quality judgment; a page whose HTML carries only a heading and boilerplate gives the crawler nothing to rank.
- **Text shown in images must also exist as real HTML text** — crawlers cannot read image-embedded text.
- **Build navigation from real `<a href>` links, not controls** — crawlers may not treat buttons or JS click handlers as links.

## 2. Titles and meta descriptions

The canonical home of these numbers is this section — every other mention references it.

- **Title: unique per URL, ~50–60 characters** (pixel-width limited, so front-load what matters), only one `<title>` per page. The title is the single highest-leverage on-page element: **primary keyword at or near the front**, **brand at the end** (`Primary Keyword — Brand`), **accurate and matching the `<h1>`** (Google rewrites titles that are too long, stuffed, or inconsistent), and **earn the click** with specificity — never clickbait the page can't deliver.
- **Meta description: unique per URL, ~150–160 characters** (~120 shows on mobile) — never duplicated or templated across pages. Not a ranking factor, but it's your ad copy in the results: address the intent directly, use the primary keyword naturally, end with a soft CTA. (Don't mistake CTR itself for a ranking lever.)
- Both manageable at the page level in any CMS without a deploy.

→ `references/examples.md` §2 — before/after title and meta-description examples — read when writing or fixing either.

## 3. Canonical tags

Every URL must carry `rel="canonical"`, using an **absolute `https://` URL** — never relative — pointing to the page's own canonical URL, not a duplicate or parameterized variant.

## 4. URL strategy and slugs

- **Keep the canonical URL clean — no parameters baked in.** Point `rel=canonical` at the parameter-free URL so tracked variants aren't indexed as duplicates.
- **Force lowercase** via `301`; **pick one trailing-slash convention and enforce it site-wide** (`301` the other variant) — Google treats `/page` and `/page/` as different URLs, so what matters is exactly one canonical, consistently-used form.
- **No extraneous folders** (`/cms/`, `/app/`) in public URLs; **hyphens as delimiters**, never underscores or spaces — strip or hyphenate all other punctuation too; **flat category structure preferred**; **no exposed dates** unless meaningful to the content.
- **Slugs: short — 3–5 words, under ~60 characters, primary keyword front-loaded.** Don't cram multiple keywords.

→ `references/examples.md` §4 — before/after slug examples — read when choosing a slug.

## 5. Domain and HTTPS

All pages on a single canonical hostname — no sub-domains/ccTLDs/microsites splitting SEO-relevant content. `http://` redirects to `https://` with a `301`, enforced via HSTS. No content reachable on more than one hostname.

## 6. Redirects

- **`301` for every permanent move** (URL changes, canonicalization, `http`→`https`, lowercase/slash enforcement) — a `301` is the unambiguous "move ranking here" signal; `302`/`307` says "temporary," the wrong signal for a permanent move. Reserve `302`/`307` for genuinely temporary redirects. Never use JavaScript or meta-refresh redirects for either — always a server-side status code.
- **No redirect chains** — every redirect points directly to the final URL.
- **Legacy URLs with no relevant match** `301` to the most relevant page.
- **Deprecated URLs with no replacement**: prefer `410 Gone` (a correct `404` is also fine); never a `301` to an irrelevant page or a `200` rendering an error page.
- CMS-generated internal links (nav, breadcrumbs) must auto-update when a URL changes.

## 7. Schema.org markup

Mark up applicable elements with **JSON-LD** (Google's preferred format for new work). **Prioritize types that earn rich results** — markup exists to unlock search features, not as decoration; bare `WebPage` markup drives no rich result. Common types where content exists: `Product` (price via a nested `Offer`/`AggregateOffer`), `ImageObject`, `AggregateRating`, `VideoObject`, `BreadcrumbList`. Goes in a `<script type="application/ld+json">` block, `<head>` or `<body>` (either works; `<head>` is the tidier convention).

## 8. Internal linking

Every link carries meaningful, machine-readable text (visible anchor, or descriptive `alt`/`aria-label` when wrapping an image) — descriptive and varied, never "click here". Links point to the destination's canonical URL, not a redirecting/parameterized/duplicate one. Prefer **absolute URLs** in navigation. Navigation must be HTML/CSS-renderable without JavaScript. In prose, **~2–5 contextual internal links per 1,000 words** is a natural baseline, and **topic clusters** — a broad pillar page linked to and from specific cluster pages — are how you build the topical authority modern ranking rewards and help AI engines infer what you're authoritative on.

## 9. JavaScript and content rendering

**Content that needs to be indexed must not require JavaScript to render** — Googlebot executes JS eventually, but slower and less reliably, risking missed content during crawl budget. Indexable body text, headings, and links must exist in the initial HTML response. Externalize and minify non-critical JS; remove render-blocking JS/CSS from above-the-fold content.

## 10. CSS, images, and performance

Externalize and minify non-critical CSS — no inline `<style>` for anything non-dynamic. No whitespace, developer comments, or commented-out code in rendered HTML. Images: modern formats (WebP/AVIF), correctly sized (`srcset`/`sizes`), lazy-loaded below the fold, long-lived caching. Browser caching, HTTP/2+, and HSTS enabled. (Don't resurrect the old "cookie-free domain" advice — under HTTP/2+ it costs more than it saves.)

## 11. Indexation and crawl controls

- `robots.txt` editable by the SEO team outside release cycles.
- Internal search results, filter/facet URLs beyond the canonical sub-category, and sort/parameter URLs: `<meta name="robots" content="noindex, follow">`. Know the limit: a page that stays `noindex` long-term is eventually treated as `noindex, nofollow` too — its links stop being followed — so this keeps junk out of the index but isn't a durable way to route link equity; keep everything important reachable through indexable navigation too.
- Dev/staging/test environments: login-gated **and** blocked by `robots.txt` — the staging `robots.txt` must never be promoted to production.
- Maintenance/downtime: serve `503` (not `404`). Beta/preview releases: a known path on the main domain (e.g. `/beta/`), blocked from indexing — not a separate hostname.

## 12. 404 and gone pages

`404` pages serve an actual `404` status (never `200`) and contain helpful internal links (popular pages, "back to home"). Retired URLs with no replacement: `410 Gone`.

## 13. XML sitemaps

Auto-generated, covering all indexable pages; manual upload also possible; referenced correctly in `robots.txt`; a video sitemap if video content exists.

## 14. CMS requirements

Any CMS must provide per-page manual control of `<title>` (with template defaults), `rel="canonical"`, `meta robots` overrides, and `rel=alternate hreflang` (if multi-language).

---

## Part B — the editorial layer

On-page SEO no longer rests on keyword counts and word counts; it rests on **genuine expertise**, **structure a machine can extract**, and **earning citations in AI answers**. Two things are true at once (ecosystem claims last verified 2026-07 — see Provenance): the fundamentals are the price of entry — pages cited by AI Overviews overwhelmingly *also* rank in the traditional top 10, so ranking organically is a prerequisite for AI visibility, not an alternative; and a new layer sits on top — getting *cited* by answer engines (Google AI Overviews, ChatGPT, Claude, Perplexity, Gemini) rewards content that reads as **evidentiary**. Optimize for being *the cited source*, not just for a blue-link click, because many searches now end without one.

## 15. Start from search intent — one page, one job

Before writing a word, know the **single search intent** the page serves and the **one primary query** it targets. Match the format: **informational** ("how does X work") → guide/explainer; **commercial** ("best X for Y") → comparison/roundup; **transactional** ("buy X") → product/pricing page; **navigational** ("Acme login") → the destination itself. Intent match is the dominant ranking factor — one thorough page that nails the intent beats several thin pages chasing keyword variants.

**Don't create keyword cannibalization** — two pages targeting the same keyword *and* intent compete with each other and split authority. One primary intent per URL. If two existing pages overlap, merge them and `301` the weaker URL (§6).

## 16. Heading structure as questions

Exactly one `<h1>` (§1), containing the primary keyword naturally and closely matching the title. `<h2>` for major sections, `<h3>` for nested subtopics — logical hierarchy, never skip levels. **Phrase headings as the questions users actually ask** ("How much does X cost?") and answer immediately underneath (§19) — this is what wins featured snippets and AI citations.

## 17. Keyword targeting — and the myths to drop

Place the **primary keyword** in the title (front), the `<h1>`, the **first 100 words**, the URL slug, at least one `<h2>`, and image alt text where natural — then write naturally for a human. Kill these outdated tactics: **keyword density is a myth** (no target percentage; stuffing *reduces* AI visibility and can trigger spam classification); **"LSI keywords" aren't real** — the valid idea underneath is **semantic coverage** (synonyms, related entities, sub-topics, since engines reason over entities, not word frequency); **word count is not a ranking factor** — thorough coverage tends to run long, but length is a byproduct, not a cause.

## 18. Depth over length

What actually ranks is **topical completeness** — covering the question, its sub-questions, and the relevant entities better than competing pages. Before publishing, check: does this answer the follow-up questions a curious reader would ask next? Match length to intent — a definition needs a paragraph; a buying guide needs the whole journey.

## 19. Write for extraction — snippets and AI citations

Where modern content SEO and AI-answer optimization converge:

- **Lead each section with a direct 40–60 word answer** immediately under a question-style heading, *then* expand — this is what gets pulled into featured snippets and AI summaries.
- **Use structure machines can lift:** short paragraphs (1–3 sentences), bulleted/numbered lists, comparison tables, bolded key phrases.
- **Make it evidentiary** — the strongest lever for LLM citation. What earns citations, ranked: **attributed expert quotes** first, then **concrete statistics and data**, then **inline citations to sources** — while **keyword stuffing actively reduces AI visibility**.
- **Publish original information** — first-hand testing, original data, a genuine point of view ("information gain" is what both Google and AI engines reward).
- **Keep facts consistent** with what's stated elsewhere, and **keep content fresh** (a real "Last updated" date helps, especially with Perplexity).

## 20. E-E-A-T, people-first, and AI-generated content

Google rewards content made **primarily to help people**, judged on **E-E-A-T**: Experience, Expertise, Authoritativeness, Trustworthiness (the most important of the four). **Show who wrote it** (byline, bio, credentials), **demonstrate first-hand experience** (original screenshots, tests, specifics only a practitioner would know), **cite sources**. Apply the **Who / How / Why** test — is authorship clear, is automation disclosed, and does this exist to help the reader rather than game search?

On AI-generated content, Google is **authorship-neutral**: it rewards helpful, reliable content "however it is produced." What gets penalized is **scaled content abuse** — mass-producing pages, by AI or anyone, primarily to manipulate rankings without adding value. The trigger is *low value at scale*, not the use of AI. One genuinely useful, human-reviewed, AI-assisted article is fine; 500 templated thin pages are not.

## 21. Image alt text and file names

**Alt text: descriptive, ≤125 characters, keyword used naturally** — describe what the image shows; empty `alt=""` for purely decorative images. This applies to genuinely descriptive **content images** only: functional and decorative images follow [[accessibility]]'s purpose rules (name the function, or stay empty), and a keyword never overrides that — "project management dashboard icon" on a Settings button is a screen-reader regression, not SEO. **File names: lowercase, hyphen-separated, descriptive.** **Captions** get read more than body copy — use them for genuine context.

→ `references/examples.md` §21 — before/after alt-text examples — read when writing alt text.

## 22. Anti-patterns — what gets content penalized or ignored

Never: **keyword stuffing** anywhere; **thin or scaled content** (the top current penalty trigger); **doorway pages** (near-duplicate pages for keyword/location variants); **keyword cannibalization** (§15); **misleading titles/meta or clickbait**; **duplicate/templated titles and metas**; **over-optimization** (exact-match anchors everywhere, forced keywords, robotic phrasing); and the dead tactics still floating around — keyword-density %, "LSI keywords," writing to an arbitrary word count, the `meta keywords` tag, spun/auto-generated filler.

→ `references/examples.md` — "A note on `llms.txt`", read only if specifically asked about it.

---

## Self-check before shipping

Run this against the diff, template, or piece. Fix any "no" before finishing. Items unverifiable from the change alone — site-wide uniqueness, redirect chains, staging gating, cannibalization against pages you haven't seen — get an explicit "not verified: <what>" line in the hand-off, never an assumed "yes":

- Exactly one `<h1>` per page, tied to the focus keyword, not used in nav? (An SEO convention — [[accessibility]] deliberately does not treat multiple `<h1>`s as a failure; report violations as SEO findings.)
- Title unique, ~50–60 chars, keyword near the front, matching the `<h1>`, present exactly once? Meta description unique, ~150–160 chars, addressing the intent?
- `rel="canonical"` on every URL, absolute `https://`, pointing to its own parameter-free canonical form — never self-canonicalizing a tracked or faceted variant?
- URLs lowercase, one consistent trailing-slash convention, parameter-free, no extraneous folders — slugs short with the keyword front-loaded?
- Every permanent move a `301` (not `302`/`307`), pointing directly to the final URL, no chains — and deprecated unreplaceable URLs returning `410` (or a correct `404`), never `200` or an irrelevant `301`?
- All indexable body content in the initial HTML response, not JS-rendered — and Schema.org JSON-LD present for every applicable element?
- Filter/facet/sort URLs carrying `noindex, follow`, and staging login-gated with its `robots.txt` never promoted to production?
- One clear search intent and primary query, no cannibalization — with the piece covering the topic to completion, length driven by intent?
- Each section led by a 40–60 word direct answer under a question heading, evidentiary (statistics, attributed quotes, citations, original information), with clear authorship?
- Images: descriptive ≤125-char alt (or empty if decorative), hyphenated lowercase file names — and 2–5 descriptive internal links per 1,000 words of prose?
- Free of every anti-pattern in §22?

---

## Provenance and maintenance

Search-engine behavior drifts. Technical claims were last verified against Google Search Central documentation in **2026-07**; re-verify with a search of Google's current docs before leaning hard on any in a dispute: trailing-slash/URL-consistency (§4); no-minimum-word-count and thin-content treatment (§1); long-term `noindex, follow` degrading to `nofollow` (§11); `404` vs `410` handling (§6, §12); JSON-LD placement and rich-result-eligible types (§7); title/meta display lengths (§2 — **the canonical home of those numbers is §2**; change them there first). The editorial fundamentals (intent, structure, E-E-A-T, anti-patterns) are durable, but the **ecosystem claims** are not — last verified **2026-07**: AI Overviews prevalence and the cited-pages-also-rank-top-10 overlap; the `llms.txt` adoption status; freshness mattering "especially with Perplexity"; which named answer engines matter.

## Reference files

| File | What it answers |
|------|-----------------|
| `references/examples.md` | Before/after examples for titles, meta descriptions, alt text, and URL slugs; plus the `llms.txt` digression |
