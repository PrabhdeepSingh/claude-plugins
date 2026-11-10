---
name: seo
description: >-
  SEO for pages meant to be found on the public web: head metadata, URLs, redirects, structured data, sitemaps, and prose that ranks and gets cited by answer engines. Use for public sites and content. Not for an app's own screens or interface copy ([[ux-writing]]).
---

# SEO — for pages meant to be found

The project's own instructions win: where its AGENTS.md, CLAUDE.md, design system or established conventions say otherwise, follow them and skip that part of this.

This applies only to pages that are public and meant to be found through search: marketing sites, landing pages, blogs, guides, public documentation. It does not apply to an application's own screens, a desktop or signed-in product interface, internal tools, or anything behind a login. A component file being HTML or JSX is not a reason to use this.

Part A is the plumbing and Part B the writing. A template change uses A, a prose change uses B, a new page both. Apply the rules to what the task touches; report other problems you notice instead of fixing them. Site-wide facts you cannot check from the change (uniqueness, redirect chains, cannibalization against pages you have not seen) are stated as not verified.

---

## Part A — the plumbing

**Headings and body.** One `<h1>` per URL, the main on-page heading, matching the page's topic and not a site name or nav label. (This is a search convention; [[accessibility]] does not treat several `<h1>`s as a failure.) Indexable body text, headings and links exist in the initial HTML response and do not depend on JavaScript, which crawlers run later and less reliably. Text shown in images also exists as real text. Navigation is real `<a href>` links.

**Titles and meta descriptions.** These numbers live here.

- Title: unique per URL, about 50–60 characters, one per page, primary keyword near the front, brand at the end (`Primary Keyword — Brand`), matching the `<h1>`. Search engines rewrite titles that are too long, stuffed or inconsistent with the page.
- Meta description: unique per URL, about 150–160 characters (about 120 show on mobile). It is not a ranking factor; it is the page's pitch in the results, so answer the intent directly.

**Canonical.** Every URL carries `rel="canonical"` with an absolute `https://` URL pointing at its own parameter-free canonical form, so tracked and faceted variants are not indexed as duplicates.

**URLs.** Lowercase, hyphens as delimiters, one trailing-slash convention enforced site-wide with a `301` (`/page` and `/page/` are different URLs), no extraneous folders, no dates unless they mean something. Slugs are 3–5 words, under about 60 characters, keyword first.

**Host.** One canonical hostname for all search-relevant content, `http` redirected to `https` with a `301`, HSTS on.

**Redirects.** `301` for every permanent move; `302` and `307` say "temporary" and are for moves that are. Always a server-side status, not JavaScript or meta refresh. No chains: each redirect points at the final URL. A retired URL with no relevant replacement returns `410` (a correct `404` is also fine), not a `301` to an unrelated page and not a `200` rendering an error page. A `404` page returns a real `404` status. Maintenance returns `503`.

**Structured data.** JSON-LD in a `<script type="application/ld+json">` block. Mark up types that earn rich results where the content exists (`Product` with a nested `Offer`, `AggregateRating`, `VideoObject`, `BreadcrumbList`); bare `WebPage` markup earns nothing.

**Crawl controls.** Internal search results and filter, facet and sort URLs carry `noindex, follow`. A page left `noindex` long-term is eventually treated as `nofollow` too, so keep everything important reachable through indexable navigation. Staging is login-gated and blocked in `robots.txt`, and the staging `robots.txt` is never promoted to production. Previews live on a blocked path of the main domain, not a separate hostname.

**Sitemaps.** Generated automatically, covering all indexable pages, referenced from `robots.txt`.

**Links.** Anchor text describes the destination and points at its canonical URL. In prose, about 2–5 contextual internal links per 1,000 words; a broad pillar page linked to and from specific pages builds topical authority.

**Images.** Modern formats, correctly sized (`srcset`, `sizes`), lazy-loaded below the fold. Alt text on content images is descriptive, 125 characters or fewer, with the keyword only where natural; functional and decorative images follow [[accessibility]], and a keyword never overrides that. File names are lowercase and hyphenated.

**A CMS** gives per-page control of `<title>`, canonical, `meta robots` and, for multi-language sites, `hreflang`, and updates internal links when a URL changes.

---

## Part B — the writing

Ranking in the ordinary results is still the entry price: pages cited in AI answers overwhelmingly also rank in the top results. On top of that, being cited by answer engines rewards content that reads as evidence (ecosystem claims last verified 2026-07; see Provenance).

**One page, one intent.** Know the single search intent and primary query the page serves, and match the format: informational gets a guide, commercial a comparison, transactional a product or pricing page. Two pages targeting the same keyword and intent compete with each other; merge them and `301` the weaker URL.

**Keyword placement.** Primary keyword in the title, the `<h1>`, the first 100 words, the slug and at least one `<h2>`, then write for a person. Keyword density is not a target, "LSI keywords" are not real (the valid idea is covering related entities and sub-topics), and word count is not a ranking factor. Length follows from covering the question and the follow-up questions a reader would ask next.

**Write for extraction.** Where it suits the content, phrase a heading as the question people ask and open the section with a direct 40–60 word answer before expanding. Short paragraphs, lists and comparison tables are what snippets and AI summaries lift.

**Be evidence.** What earns citations, in order: attributed expert quotes, concrete statistics and data, inline citations to sources. Publish original information (first-hand testing, original data, a real point of view). Keep facts consistent with what is stated elsewhere and show a real last-updated date.

**Authorship and AI-assisted content.** Show who wrote it and their first-hand experience, and cite sources. Search engines reward helpful content however it is produced; what is penalized is low-value content at scale. One useful, human-reviewed, AI-assisted article is fine, and 500 templated thin pages are not.

**What gets penalized or ignored.** Keyword stuffing, thin or scaled content, doorway pages for keyword or location variants, misleading or duplicated titles and descriptions, exact-match anchors everywhere, the `meta keywords` tag.

**`llms.txt`.** If asked: adoption is low and no major answer engine has confirmed using it for ranking or citation. Its clearest value today is giving coding assistants clean context for API and library documentation. Treat it as optional; structured data is the established way to be machine-readable.

---

## Provenance and maintenance

Search-engine behavior drifts. Technical claims were last verified against the major search engine's own documentation in 2026-07; re-verify before leaning on one in a dispute: trailing-slash and URL consistency; no minimum word count and thin-content treatment; long-term `noindex, follow` degrading to `nofollow`; `404` vs `410` handling; JSON-LD placement and rich-result-eligible types; title and meta display lengths (their home is the titles paragraph above; change them there first). The editorial fundamentals are durable, but the ecosystem claims are not, last verified 2026-07: the overlap between pages cited in AI answers and pages that rank; `llms.txt` adoption; freshness mattering to answer engines; which answer engines matter.
