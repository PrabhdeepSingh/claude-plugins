# Content SEO — worked examples and the llms.txt aside

Read the section matching what you're currently checking. The rules and their *why* live inline in `content-seo/SKILL.md`; this file is illustration plus one rarely-needed digression.

## §2 — title tag

```text
Avoid:  Welcome to Our Blog Where We Discuss Many Topics About Software and More
Avoid:  Project Management Tips, Project Management Tools, Project Management Software   (stuffed)
Prefer: 7 Project Management Tips for Remote Teams — Acme
```

## §3 — meta description

```text
Avoid:  Read our blog post about project management for remote teams. Click here to learn more.
Prefer: Managing a remote team? These 7 project-management tactics cut status meetings and keep
        delivery on track — with templates you can copy today.
```

## §8 — image alt text

```text
Avoid:  alt="seo tips seo guide seo best practices marketing"
Avoid:  alt="image"
Prefer: alt="Kanban board showing a remote team's sprint split into To-Do, Doing, and Done"
```

## §9 — URL slug

```text
Avoid:  /blog/2026/01/the-top-10-best-project-management-tips-for-remote-teams-this-year
Prefer: /blog/remote-team-project-management
```

## A note on `llms.txt`

You may be asked about `/llms.txt` (a proposed Markdown file mapping a site's key content for LLMs). Be honest: **adoption is low and no major answer engine has confirmed using it for ranking or citation** — Google has publicly said it does *not* use it. Its clearest real value today is giving **coding assistants** clean context for API/library docs. For editorial content, **schema.org structured data is the established, higher-priority way** to be machine-readable (a `[[seo-standards]]` concern). Treat `llms.txt` as optional/nice-to-have, not a core tactic — and don't oversell it.
