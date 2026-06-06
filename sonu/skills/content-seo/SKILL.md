---
name: content-seo
description: On-page / editorial SEO for written content — so what you publish ranks AND gets cited by AI answer engines. INVOKE THIS PROACTIVELY — even when the user never says "SEO" — whenever writing or editing prose meant to be published on the web — blog posts, articles, guides, tutorials, landing-page copy, product/marketing pages, press releases, "what's new"/changelog/release-note entries, FAQ pages, documentation, or Markdown content (`content/**/*.md`, `*.mdx`) and its frontmatter (title, description, slug, canonical). Covers search intent, title tags, meta descriptions, heading structure, keyword targeting, content depth, internal linking, image alt text, URL slugs, featured-snippet and AI-citation formatting, E-E-A-T, and the do's/don'ts that get content penalized. It is the editorial-SEO quality bar for all published writing produced here — reach for it on any content-writing task, not only when SEO is named. (Pairs with the technical [[seo-standards]] skill, which covers the HTML/template/redirect side.)
---

# Content SEO — write it so humans rank it and AI cites it

This is the editorial counterpart to the technical [[seo-standards]] skill. That one governs the *plumbing* (HTML structure, canonical tags, redirects, sitemaps). This one governs the *writing* — what makes a blog post, article, or landing page actually earn visibility.

## The shift that frames everything

On-page SEO is no longer about keyword counts and word counts. In 2025-2026 it rests on three things: **genuine expertise**, **structure a machine can extract**, and **earning citations in AI answers**. Two things are true at once:

- **The fundamentals are the price of entry.** AI Overviews now appear on a large and growing share of Google searches, and the pages they cite overwhelmingly *also* rank in the traditional top 10. So ranking organically is now a *prerequisite* for AI visibility, not an alternative to it.
- **A new layer sits on top.** Getting *cited* by AI answer engines (Google AI Overviews, ChatGPT, Claude, Perplexity, Gemini) rewards content that reads as **evidentiary** — quotes, statistics, citations, clear structure. Optimize for being *the cited source*, not just for a blue-link click, because many searches now end without one.

Everything below serves both readers and machines — they almost always want the same thing: a clear, trustworthy, well-structured answer.

## When to apply this

Any time you're writing or editing prose destined for the web: blog posts, articles, guides, tutorials, landing/marketing copy, press releases, "what's new"/changelog entries, FAQs, docs, or Markdown content and its frontmatter. If it will be published and read, this applies — even when nobody said "SEO."

---

## 1. Start from search intent — one page, one job

Before writing a word, know the **single search intent** the page serves and the **one primary query** it targets. Match the format to the intent:

- **Informational** ("how does X work") → guide, explainer, tutorial.
- **Commercial** ("best X for Y") → comparison, roundup, review.
- **Transactional** ("buy X", "X pricing") → product/pricing page.
- **Navigational** ("Acme login") → the destination itself.

Intent match is the dominant ranking factor — a single thorough page that nails the intent beats several thin pages chasing keyword variants.

**Don't create keyword cannibalization.** Two pages targeting the *same keyword and same intent* compete with each other, split authority, and confuse Google about which to rank — both suffer. One primary intent per URL. (Same keyword with *different* intent is fine: a "features" page and a "buy" page for the same product legitimately coexist.) If two existing pages overlap, merge them into one stronger page and `301` the weaker URL (a redirect job for [[seo-standards]]).

## 2. Title tags

The title is the single highest-leverage on-page element — it's the headline in search results and the strongest on-page relevance signal.

- **~50–60 characters.** The real limit is pixel width (~600px), so Google may truncate longer titles — **front-load** what matters so it survives.
- **Primary keyword at or near the front.** Leading with a function word ("How", "The") before it is fine.
- **Brand at the end:** `Primary Keyword — Brand`.
- **Be accurate and match the `<h1>`.** Google rewrites a large share of titles when they're too long, stuffed, misleading, or inconsistent with the page heading. Keep it truthful and aligned with the H1 and you keep control of it.
- **Earn the click** with specificity (numbers, a clear value prop) — never clickbait the page can't deliver.

```text
Avoid:  Welcome to Our Blog Where We Discuss Many Topics About Software and More
Avoid:  Project Management Tips, Project Management Tools, Project Management Software   (stuffed)
Prefer: 7 Project Management Tips for Remote Teams — Acme
```

## 3. Meta descriptions

Not a ranking factor — but it's your ad copy in the results: a stronger description wins more clicks (more traffic) from the same position. Don't treat CTR itself as a ranking lever, though — Google says click metrics aren't a direct ranking signal.

- **150–160 characters** (≈120 shows on mobile). Beyond ~160 truncates.
- **Address the intent directly**, include the primary keyword naturally (it gets **bolded** when it matches the query), and add a soft CTA ("See the data", "Learn how").
- **Unique per page** — never duplicate or template descriptions across pages.
- Google rewrites these often, but a good one that *does* show lifts CTR — so still write it. Spark curiosity; don't give the whole answer away.

```text
Avoid:  Read our blog post about project management for remote teams. Click here to learn more.
Prefer: Managing a remote team? These 7 project-management tactics cut status meetings and keep
        delivery on track — with templates you can copy today.
```

## 4. Heading structure

Headings are how both a skimming human and an extracting AI navigate the page.

- **Exactly one `<h1>`**, containing the primary keyword naturally, closely matching (not necessarily identical to) the title.
- **`<h2>`** for major sections (use keyword variations and related concepts), **`<h3>`** for subtopics nested under them. Logical hierarchy — never skip levels.
- **Phrase headings as the questions users actually ask** — "How much does X cost?", "What is Y?" — and answer immediately underneath (see §10). This is what wins featured snippets and AI citations.

## 5. Keyword targeting — and the myths to drop

Place the **primary keyword** in: the title (front), the `<h1>`, the **first 100 words**, the URL slug, at least one `<h2>`, and image alt text where natural — then write naturally for a human.

Kill these outdated tactics — they don't help and several actively hurt:

- **Keyword density is a myth.** There's no target percentage. Stuffing *reduces* AI visibility and can trigger spam classification.
- **"LSI keywords" aren't a real thing** Google uses. The valid idea underneath is **semantic coverage**: include synonyms, related entities, and sub-topics so engines understand the topic by its *relationships*, not its word frequency. Google reasons over entities, not counts.
- **Word count is not a ranking factor.** Top-ranking pages skew long because thorough coverage *tends* to run long — length is a byproduct, not a cause. Don't pad to a number; in AI Overviews, word count and citation are essentially uncorrelated. Write the length the query needs and stop.

## 6. Depth over length

What actually ranks is **topical completeness** — covering the question, its sub-questions, and the relevant entities better than the competing pages. Before publishing, check: does this answer the follow-up questions a curious reader would ask next? Match length to intent — a definition needs a paragraph; a buying guide needs the whole journey. A 600-word page that fully answers the query beats a padded 2,500-word one.

## 7. Internal linking and topic clusters

Internal links spread authority and tell search engines how your content relates.

- **~2–5 contextual internal links per 1,000 words** as a natural baseline — never forced.
- **Descriptive, varied anchor text** that previews the destination. Never "click here" or "read more"; don't hammer the same exact-match anchor sitewide.
- **Build topic clusters:** a broad **pillar page** on a core topic, linked to and from **cluster pages** on specific sub-topics. This is how you build the **topical authority** modern ranking rewards — and the link relationships also help AI engines infer what you're authoritative on.
- Link to **canonical URLs**, not redirecting or parameterized ones (a [[seo-standards]] rule).

## 8. Image SEO

- **Alt text: descriptive, ≤125 characters, keyword used naturally** — describe what the image shows and means. Never stuff keywords (Google flags it). Purely decorative images get empty `alt=""`; functional images (icon buttons) describe the *action* (`alt="Search"`, not `alt="magnifying-glass icon"`).
- **File names: lowercase, hyphen-separated, descriptive** — `remote-team-standup.jpg`, never `IMG_0023.JPG` or `image1.png`.
- **Captions** get read more than body copy — use them for genuine context where natural.

```text
Avoid:  alt="seo tips seo guide seo best practices marketing"
Avoid:  alt="image"
Prefer: alt="Kanban board showing a remote team's sprint split into To-Do, Doing, and Done"
```

## 9. URL slugs

- **Short — 3–5 words, under ~60 characters.** Shorter URLs tend to rank better.
- **Lowercase, hyphens (not underscores), primary keyword front-loaded**, stop words dropped where natural.
- **Don't** cram multiple keywords (reads as spam) or bake in dates unless the content is genuinely date-specific (news, events).

```text
Avoid:  /blog/2026/01/the-top-10-best-project-management-tips-for-remote-teams-this-year
Prefer: /blog/remote-team-project-management
```

## 10. Write for extraction — snippets and AI citations

This is where modern content SEO and AI-answer optimization converge. The format that wins both:

- **Lead each section with a direct 40–60 word answer** placed immediately under a question-style heading, *then* expand. This "answer-first" shape is what gets pulled into featured snippets and AI summaries.
- **Use structure machines can lift:** short paragraphs (1–3 sentences), bulleted and numbered lists, comparison tables, bolded key phrases.
- **Make it evidentiary** — this is the strongest lever for getting cited by LLMs. In a 2024 Princeton study of generative engines (the paper that coined "GEO"), the biggest gains came from **attributed expert quotes (~+41%)**, **statistics and data (~+30%)**, and **inline citations to sources (~+30%)**, while **keyword stuffing reduced visibility (~-9%)**. Treat the exact percentages as directional, but the direction is robust: cite sources, quote named experts, include real numbers, and write clearly.
- **Publish original information** — first-hand testing, original data, a genuine point of view. "Information gain" (something not already on every other page) is what both Google and AI engines reward.
- **Keep facts consistent** with what's stated elsewhere about you and the topic, and **keep content fresh** (a real "Last updated" date helps, especially with Perplexity).

## 11. E-E-A-T and people-first content (Google's bar)

Google rewards content made **primarily to help people**, not primarily to rank — and it judges **E-E-A-T**: Experience (first-hand), Expertise, Authoritativeness, and Trustworthiness (the most important of the four).

- **Show who wrote it** — a real byline, bio, and credentials.
- **Demonstrate first-hand experience** — original screenshots, tests, data, specifics only someone who's actually done it would know.
- **Cite sources and link to evidence**; keep facts accurate and current.
- Apply Google's **Who / How / Why** test: is authorship clear (*who*), is any automation disclosed where relevant (*how*), and — the one that matters most — does this exist to **help the reader** rather than to game search (*why*)?
- Ask: *would someone bookmark, share, or recommend this?* If not, it's probably search-engine-first filler.

## 12. AI-generated content — the line that matters

Google is **authorship-neutral**: it rewards helpful, reliable content "however it is produced." AI assistance is fine. What gets penalized is **scaled content abuse** — mass-producing pages (by AI or anyone) **primarily to manipulate rankings, without adding value**. The trigger is *low value at scale*, not the use of AI. So: one genuinely useful, human-reviewed, AI-assisted article is fine; 500 templated thin pages are not. Whatever the source, ensure accuracy, add something original, and keep a human in the loop.

## 13. Anti-patterns — what gets content penalized or ignored

Never do these:

- **Keyword stuffing** anywhere — body, title, meta, alt text. No density target exists; it hurts rankings *and* AI visibility.
- **Thin or scaled content** — pages with little original info, effort, or added value; mass-produced pages to chase rankings. This is the top current penalty trigger.
- **Doorway pages** — near-duplicate pages for slight keyword/location variants funneling to one place.
- **Keyword cannibalization** — multiple same-intent pages fighting over one term (§1).
- **Misleading titles/meta or clickbait** the page doesn't deliver — drives Google to rewrite your title and readers to bounce.
- **Duplicate/templated titles and metas** across pages.
- **Over-optimization** — exact-match anchors everywhere, a forced keyword in every heading, robotic phrasing.
- **Dead tactics still floating around:** chasing a keyword-density %, "LSI keywords," writing to an arbitrary word count, the `meta keywords` tag (ignored since 2009), spun/auto-generated filler, and publishing AI output with no human value added.

## A note on `llms.txt`

You may be asked about `/llms.txt` (a proposed Markdown file mapping a site's key content for LLMs). Be honest: **adoption is low and no major answer engine has confirmed using it for ranking or citation** — Google has publicly said it does *not* use it. Its clearest real value today is giving **coding assistants** clean context for API/library docs. For editorial content, **schema.org structured data is the established, higher-priority way** to be machine-readable (a [[seo-standards]] concern). Treat `llms.txt` as optional/nice-to-have, not a core tactic — and don't oversell it.

---

## Self-check before publishing

Run this against the piece. Fix any "no" before it ships:

- Is there **one clear search intent and primary query**, with no other page already targeting the same intent (no cannibalization)?
- **Title:** ~50–60 chars, primary keyword near the front, accurate, matches the `<h1>`, earns the click without clickbait?
- **Meta description:** unique, 150–160 chars, addresses the intent, keyword used naturally, soft CTA?
- **One `<h1>`** with the keyword; logical `<h2>`/`<h3>` hierarchy; headings phrased as real user questions?
- Primary keyword in the **title, H1, first 100 words, slug** — and the body written naturally, with **no stuffing** and no density-chasing?
- Does it cover the topic **completely** (the reader's follow-up questions answered), with length driven by the intent rather than a word-count target?
- **2–5 descriptive internal links per 1,000 words** to relevant canonical pages — no "click here," no forced links?
- Every image: **descriptive ≤125-char alt text** (or empty alt if decorative), **hyphenated lowercase file name**?
- **Slug:** short, lowercase, hyphenated, keyword front-loaded, no dates-unless-needed?
- Does each section **lead with a 40–60 word direct answer** under a question heading, and is the content **scannable** (short paragraphs, lists, tables)?
- Is it **evidentiary** — real statistics, attributed quotes, inline citations, original information someone would cite?
- Does it demonstrate **E-E-A-T** — clear authorship, first-hand experience, accurate sourced facts — and read as **made to help the reader**, not to game search?
- If AI-assisted: has a **human added original value and verified the facts** (no fabricated stats or quotes)?
- Free of every anti-pattern in §13?
