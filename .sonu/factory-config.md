---
tracker: github
---
Factory tracker configuration for this repo. Workflows read only the YAML
frontmatter above; this prose is for humans.

Tickets live in this repo's GitHub Issues. Triggers are the labels
`factory-ready-for-spec`, `factory-ready-to-implement`, and
`factory-ready-to-ship` — applying one is a human's one-shot authorization for
a single factory pass, and the ship label is merge authority: gate who can
apply it as tightly as who can merge. Agents never apply triggers. Credentials
come from the authenticated `gh` CLI; nothing secret is stored here.
