---
name: security
description: >-
  Build-time security discipline — threat-model before controls, Always/Ask-First/Never boundary tiers, SSRF, supply chain, LLM/agent security, privacy. INVOKE PROACTIVELY when building anything touching auth, user data, uploads, outbound fetches, dependencies, or model output — even when nobody says "security". The after-the-fact review pass belongs to /security-review; validation mechanics: [[code-standards]]; infra secrets: [[infra-standards]].
---

# Security — decide what can go wrong before deciding what to build

Security added after the design is a patch; security added at build time is a property. The cheapest moment to prevent a vulnerability is while the trust boundaries are still being drawn — before the upload handler exists, before the fetch reaches the network, before the dependency is installed. These rules make that moment explicit: name what can go wrong, gate the changes that widen the attack surface, and treat the classes of failure that recur everywhere — retried secrets, poisoned packages, model output in a sink — as design constraints, not review findings.

## When to apply this

Any feature touching authentication or authorization, personal or payment data, file uploads, outbound requests to user-influenced destinations, new dependencies, webhooks or queues, or LLM/agent output feeding anything executable. The scoped mechanics stay with their owners — parameterized queries and boundary validation in [[code-standards]] §9, secret storage and CI credentials in [[infra-standards]] §3/§5 — this skill owns the decisions those mechanics assume were already made.

---

## 1. Threat-model first — five minutes, before any control

Before choosing defenses, name what's being defended. Three steps, time-boxed to minutes not meetings:

- **Map the trust boundaries** — every place outside data enters: HTTP requests, form fields, file uploads, webhooks, third-party API responses, queue messages, and **LLM output** (it's derived from inputs you don't control, so it's a boundary like any other).
- **Name the assets** — credentials, PII, payment data, admin capabilities, money movement. A boundary matters in proportion to what's reachable through it.
- **Write abuse cases next to use cases.** For each feature ask "how would I misuse this?" — then make that abuse the first test you write ([[tdd]]'s bug-fix reflex, applied before the bug exists). "Upload a profile photo" pairs with "upload a 2GB file", "upload an SVG with a script tag", "upload to someone else's profile".

If you can't name a feature's trust boundaries, you're not ready to secure it — which means you're not ready to build it.

## 2. Three boundary tiers: Always, Ask First, Never

Most security decisions are pre-made; the discipline is knowing which tier a change sits in.

- **Always** (no permission needed — these are the floor): parameterized queries, boundary validation, hashed passwords with a current algorithm, TLS on, secrets from the store, least-privilege scopes.
- **Ask First** (a human approves before the change ships, because each widens the attack surface in a way the diff understates): a new authentication flow or a change to auth logic; storing a **new category** of sensitive data; a new external service integration; a CORS change; a file-upload handler; a rate-limiting or throttling change; granting an elevated permission or role. The list is the point: these look like ordinary features and are not.
- **Never**: secrets in source or logs, PII in telemetry, disabled TLS verification, auth checks only on the client, security controls that fail open.

The Ask-First tier is the house authorization posture applied to security: the same reason a merge needs a human's trigger, an attack-surface widening needs a human's yes. In a non-interactive run, "ask first" means write the precise widening into the hand-off or ticket as a blocker and stop that part — never proceed on an approval nobody gave.

## 3. Outbound requests to user-influenced destinations (SSRF)

Any fetch whose URL a user can influence — webhooks, link previews, importers, "fetch my avatar from a URL" — can be aimed at your own internal network. The defense, in order:

- **Allowlist the scheme and host** when the destination set is knowable; an allowlist beats any amount of blocklist cleverness.
- When arbitrary hosts are the feature: **resolve every DNS record and reject unless every resolved address is globally routable** (a public IPv4 address or IPv6 global unicast) — that one check rejects loopback, link-local (the cloud metadata endpoint, the single most-targeted SSRF destination), private, and unique-local ranges alike. **Forbid redirects**, or re-validate every hop — a public URL that 302s to an internal one passes the naive check.
- **Know the honest limit: the check-then-fetch gap.** The HTTP client resolves DNS again after your validation, so a short-TTL record can rebind to an internal address between the check and the connection. For high-risk surfaces, resolve once and connect to the pinned address, or route through a filtering egress proxy. A defense that doesn't state this limit reads as complete and isn't.

## 4. A committed secret is a rotated secret

The moment a credential reaches a remote — a pushed commit, a public gist, a pasted log — assume it's compromised, no matter how fast the line is deleted. Deleting the line or rewriting history is cleanup, not response: clones, forks, caches, and scrapers already have it. The order is fixed: **revoke and reissue first**, then purge from history, then find how it got there (a missing `.gitignore` entry, a hardcoded default) and close that path. [[infra-standards]] §3 owns where secrets live so this never happens; this rule owns the day it happens anyway.

## 5. Supply chain: packages run code on your machine

A dependency is code you execute with your privileges, chosen by someone you've never met.

- **Block install scripts before first execution.** Bootstrap new dependencies with lifecycle scripts disabled, inspect what the pending scripts do, approve the minimum set, and record the policy — never blanket-approve. Install scripts are arbitrary code that runs at `install`, before you've imported anything.
- **Never auto-apply forced remediation** (`npm audit fix --force` and equivalents) — forced fixes cross declared version ranges, which is a silent major upgrade wearing a security patch.
- **Triage advisories by reachability × severity**, not severity alone: is the vulnerable function actually called on your code path, at runtime or only in dev tooling, exploitable in your deployment shape? A critical advisory in code you never call is a scheduled fix; a moderate one on a hot input path is today's work. A deferred fix gets a written reason and a review date.
- **Audits catch known advisories only.** They do not catch a freshly-malicious release or a typosquat (`crossenv` for `cross-env`) — read the name you're installing, check the manifest and lockfile agree on the package manager, and stop on competing lockfiles.
- Upgrade mechanics — one dependency per change, changelog over version number, lockfile diff — live in [[code-standards]] §15.

## 6. LLM and agent output is untrusted input

A model's output is a function of inputs you don't fully control — retrieved documents, user prompts, tool results — so it inherits their trust level.

- **Never pass model output straight into a sink**: `eval`, SQL, a shell, `innerHTML`, a file path. It gets the same validation and encoding any user input gets at that sink ([[code-standards]] §9).
- **The system prompt is not a security boundary.** Enforce permissions in code, where they can't be talked out of; a prompt that says "never reveal X" is a request, not a control.
- **Anything in the context can be echoed back.** Keep secrets and other tenants' data out of prompts entirely, rather than instructing the model to withhold them.
- **Scope agent tools to the minimum and gate destructive actions on confirmation**; validate every tool argument as boundary input. **Bound consumption** — tokens, request rate, recursion/loop depth — so a crafted input can't run up cost or hang the system.
- **In retrieval systems, the vector store is a trust boundary**: partition embeddings per tenant so one user's query cannot retrieve another's documents, and validate documents before indexing so a poisoned page can't steer every future answer.

## 7. Privacy is a different question from security

Security asks "can an attacker read this data?" Privacy asks "**should we hold it at all, and for how long?**" — hardening never answers the second question, and the cheapest data to protect, breach, and comply over is the data never collected.

- **Collect a field only against a stated use.** "It might be useful later" is not a purpose — it's latent breach scope.
- **Set retention when the field is born, then actually delete** — including backups, caches, search indexes, and analytics copies. Data with no expiry is a breach scheduled for later.
- **Design the schema so one user's data is findable and erasable** — deletion and export are engineering features, and a "delete my account" that flips a flag while the PII lingers in five stores is the anti-pattern to test for.
- **Sending PII to any vendor is sharing** — analytics, ads, or an LLM API alike — and the user's consent gates it, not convenience.
- Classify what you hold: non-personal, personal, and sensitive (health, finance, precise location, biometrics, government IDs, anything about minors) — the tiers carry different legal weight and the schema should make the tier visible.

---

## Self-check before the feature ships

- Can you name the feature's trust boundaries and assets — and does each abuse case you wrote have a test?
- Did anything in the Ask-First tier ship without a human's explicit yes?
- Does every outbound fetch with a user-influenced destination validate scheme/host or resolved addresses, handle redirects, and state (or close) the check-then-fetch gap?
- If a secret ever touched a remote this change is cleaning up after: was it revoked and reissued *before* the history purge?
- New or upgraded dependencies: install scripts inspected before first execution, advisory triage by reachability with deferrals dated, package names read for typosquats?
- Does any model output reach a sink unvalidated, any secret or cross-tenant data sit in a prompt, or any agent tool lack an argument check or a consumption bound?
- For every new field of personal data: a stated purpose, a retention decision, and an erasure path that reaches the copies?

## Provenance and maintenance

The disciplines here (threat-model-first, boundary tiers, rotate-on-commit, reachability triage, privacy-as-retention) are durable. The ecosystem specifics are not — last verified 2026-08: the cloud-metadata endpoint as the leading SSRF target, package-manager script-disabling flags and forced-remediation behavior (`npm audit fix --force` and peers), and the LLM/agent attack patterns in §6, which track a fast-moving field. Re-verify against current platform and package-manager docs before leaning on an exact mechanism; per-ecosystem commands deliberately don't live here so they can't rot here.
