---
name: content-seo
description: >-
  Editorial SEO for published prose — so it ranks and gets cited by AI answer engines. INVOKE when writing or editing content meant for the web: posts, articles, guides, landing copy, press releases, changelogs, FAQs, docs, and Markdown content with its frontmatter — even when nobody says "SEO". The HTML/template/redirect side is [[seo-standards]].
---

# Content SEO — write it so humans rank it and AI cites it

This is the editorial counterpart to the technical [[seo-standards]] skill. That one governs the *plumbing* (HTML structure, canonical tags, redirects, sitemaps). This one governs the *writing* — what makes a blog post, article, or landing page actually earn visibility.

## The shift that frames everything

On-page SEO is no longer about keyword counts and word counts. It now rests on three things: **genuine expertise**, **structure a machine can extract**, and **earning citations in AI answers**. Two things are true at once (the ecosystem claims below were last verified 2026-07 — see Provenance):

- **The fundamentals are the price of entry.** AI Overviews now appear on a large and growing share of Google searches, and the pages they cite overwhelmingly *also* rank in the traditional top 10 — ranking organically is a *prerequisite* for AI visibility, not an alternative to it.
- **A new layer sits on top.** Getting *cited* by AI answer engines (Google AI Overviews, ChatGPT, Claude, Perplexity, Gemini) rewards content that reads as **evidentiary** — quotes, statistics, citations, clear structure. Optimize for being *the cited source*, not just for a blue-link click, because many searches now end without one.

Everything below serves both readers and machines — they almost always want the same thing: a clear, trustworthy, well-structured answer.

## When to apply this

Any time you're writing or editing prose destined for the web. If it will be published and read, this applies — even when nobody said "SEO."

---

## 1. Start from search intent — one page, one job

Before writing a word, know the **single search intent** the page serves and the **one primary query** it targets. Match the format: **informational** ("how does X work") → guide/explainer; **commercial** ("best X for Y") → comparison/roundup; **transactional** ("buy X") → product/pricing page; **navigational** ("Acme login") → the destination itself. Intent match is the dominant ranking factor — one thorough page that nails the intent beats several thin pages chasing keyword variants.

**Don't create keyword cannibalization** — two pages targeting the same keyword *and* intent compete with each other and split authority. One primary intent per URL. If two existing pages overlap, merge them and `301` the weaker URL (a [[seo-standards]] job).

## 2. Title tags

The title is the single highest-leverage on-page element. **~50–60 characters** (pixel-width limited, so front-load what matters), **primary keyword at or near the front**, **brand at the end** (`Primary Keyword — Brand`), **accurate and matching the `<h1>`** (Google rewrites titles that are too long, stuffed, or inconsistent), and **earn the click** with specificity — never clickbait the page can't deliver.

→ `references/examples.md` §2 — before/after title examples — read when writing or fixing a title.

## 3. Meta descriptions

Not a ranking factor, but it's your ad copy in the results — a stronger one wins more clicks from the same position (don't mistake CTR itself for a ranking lever, though). **150–160 characters** (~120 shows on mobile), **address the intent directly** with the primary keyword used naturally and a soft CTA, **unique per page** — never duplicate or template across pages.

→ `references/examples.md` §3 — before/after meta description examples — read when writing a meta description.

## 4. Heading structure

**Exactly one `<h1>`**, containing the primary keyword naturally, closely matching the title. **`<h2>`** for major sections, **`<h3>`** for nested subtopics — logical hierarchy, never skip levels. **Phrase headings as the questions users actually ask** ("How much does X cost?") and answer immediately underneath (§10) — this is what wins featured snippets and AI citations.

## 5. Keyword targeting — and the myths to drop

Place the **primary keyword** in the title (front), the `<h1>`, the **first 100 words**, the URL slug, at least one `<h2>`, and image alt text where natural — then write naturally for a human. Kill these outdated tactics: **keyword density is a myth** (no target percentage; stuffing *reduces* AI visibility and can trigger spam classification); **"LSI keywords" aren't real** — the valid idea underneath is **semantic coverage** (synonyms, related entities, sub-topics, since engines reason over entities, not word frequency); **word count is not a ranking factor** — thorough coverage tends to run long, but length is a byproduct, not a cause.

## 6. Depth over length

What actually ranks is **topical completeness** — covering the question, its sub-questions, and the relevant entities better than competing pages. Before publishing, check: does this answer the follow-up questions a curious reader would ask next? Match length to intent — a definition needs a paragraph; a buying guide needs the whole journey.

## 7. Internal linking and topic clusters

**~2–5 contextual internal links per 1,000 words** as a natural baseline, with **descriptive, varied anchor text** (never "click here"). **Build topic clusters:** a broad **pillar page** linked to and from specific **cluster pages** — this is how you build the **topical authority** modern ranking rewards, and it helps AI engines infer what you're authoritative on. Link to **canonical URLs**, not redirecting or parameterized ones (a [[seo-standards]] rule).

## 8. Image SEO

**Alt text: descriptive, ≤125 characters, keyword used naturally** — describe what the image shows; empty `alt=""` for purely decorative images. This applies to genuinely descriptive **content images** only: functional and decorative images follow [[accessibility]]'s purpose rules (name the function, or stay empty), and a keyword never overrides that — "project management dashboard icon" on a Settings button is a screen-reader regression, not SEO. **File names: lowercase, hyphen-separated, descriptive.** **Captions** get read more than body copy — use them for genuine context.

→ `references/examples.md` §8 — before/after alt-text examples — read when writing alt text.

## 9. URL slugs

**Short — 3–5 words, under ~60 characters. Lowercase, hyphens, primary keyword front-loaded.** Don't cram multiple keywords or bake in dates unless the content is genuinely date-specific.

→ `references/examples.md` §9 — before/after slug examples — read when choosing a slug.

## 10. Write for extraction — snippets and AI citations

Where modern content SEO and AI-answer optimization converge:

- **Lead each section with a direct 40–60 word answer** immediately under a question-style heading, *then* expand — this is what gets pulled into featured snippets and AI summaries.
- **Use structure machines can lift:** short paragraphs (1–3 sentences), bulleted/numbered lists, comparison tables, bolded key phrases.
- **Make it evidentiary** — the strongest lever for LLM citation. What earns citations, ranked: **attributed expert quotes** first, then **concrete statistics and data**, then **inline citations to sources** — while **keyword stuffing actively reduces AI visibility**.
- **Publish original information** — first-hand testing, original data, a genuine point of view ("information gain" is what both Google and AI engines reward).
- **Keep facts consistent** with what's stated elsewhere, and **keep content fresh** (a real "Last updated" date helps, especially with Perplexity).

## 11. E-E-A-T and people-first content

Google rewards content made **primarily to help people**, judged on **E-E-A-T**: Experience, Expertise, Authoritativeness, Trustworthiness (the most important of the four). **Show who wrote it** (byline, bio, credentials), **demonstrate first-hand experience** (original screenshots, tests, specifics only a practitioner would know), **cite sources**. Apply Google's **Who / How / Why** test — is authorship clear, is automation disclosed, and does this exist to help the reader rather than game search?

## 12. AI-generated content — the line that matters

Google is **authorship-neutral**: it rewards helpful, reliable content "however it is produced." What gets penalized is **scaled content abuse** — mass-producing pages, by AI or anyone, primarily to manipulate rankings without adding value. The trigger is *low value at scale*, not the use of AI. One genuinely useful, human-reviewed, AI-assisted article is fine; 500 templated thin pages are not.

## 13. Anti-patterns — what gets content penalized or ignored

Never: **keyword stuffing** anywhere; **thin or scaled content** (the top current penalty trigger); **doorway pages** (near-duplicate pages for keyword/location variants); **keyword cannibalization** (§1); **misleading titles/meta or clickbait**; **duplicate/templated titles and metas**; **over-optimization** (exact-match anchors everywhere, forced keywords, robotic phrasing); and the dead tactics still floating around — keyword-density %, "LSI keywords," writing to an arbitrary word count, the `meta keywords` tag, spun/auto-generated filler.

→ `references/examples.md` — "A note on `llms.txt`", read only if specifically asked about it.

---

## Self-check before publishing

Run this against the piece. Fix any "no" before it ships. Items unverifiable from the piece alone — cannibalization against pages you haven't seen, uniqueness across the site — get an explicit "not verified: <what>" line, never an assumed "yes":

- One clear search intent and primary query, no cannibalization with an existing page?
- Title ~50–60 chars, keyword near the front, accurate, matches the `<h1>`, earns the click?
- Meta description unique, 150–160 chars, addresses the intent, soft CTA?
- One `<h1>` with the keyword; logical heading hierarchy phrased as real user questions?
- Primary keyword in title/H1/first-100-words/slug, written naturally with no stuffing?
- Covers the topic completely, with length driven by intent rather than a word-count target?
- 2–5 descriptive internal links per 1,000 words to relevant canonical pages, no forced anchors?
- Every image: descriptive ≤125-char alt (or empty if decorative), hyphenated lowercase file name?
- Slug short, lowercase, hyphenated, keyword front-loaded, no unneeded dates?
- Each section leads with a 40–60 word direct answer under a question heading, and is scannable?
- Evidentiary — real statistics, attributed quotes, inline citations, original information?
- Demonstrates E-E-A-T — clear authorship, first-hand experience, accurate sourced facts — and reads as made to help the reader?
- If AI-assisted: has a human added original value and verified the facts?
- Free of every anti-pattern in §13?

---

## Provenance and maintenance

The editorial fundamentals here (intent, structure, E-E-A-T, anti-patterns) are durable. The **ecosystem claims** are not — last verified **2026-07**, re-check with a web search before asserting as current: AI Overviews prevalence and the cited-pages-also-rank-top-10 overlap; the `llms.txt` adoption status; freshness signals mattering "especially with Perplexity"; which named answer engines matter.

**Numeric limits live elsewhere:** title (~50–60 chars) and meta-description (~150–160) targets are stated editorially here, but their canonical home is [[seo-standards]] §2 — if the numbers ever change, change them there first. These two files drifted apart once before; keep one home per number.

## Reference files

| File | What it answers |
|------|-----------------|
| `references/examples.md` | Before/after examples for titles, meta descriptions, alt text, and URL slugs; plus the `llms.txt` digression |
