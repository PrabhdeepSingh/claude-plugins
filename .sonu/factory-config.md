---
tracker: github
---
Factory tracker configuration for this repo. Workflows read only the YAML
frontmatter above; this prose is for humans.

Tickets live in this repo's GitHub Issues. Triggers are the labels
`factory-ready-for-spec` and `factory-ready-to-implement` — applying one is a
human's one-shot authorization for a single factory pass. Agents never apply
them. Credentials come from the authenticated `gh` CLI; nothing secret is
stored here.
