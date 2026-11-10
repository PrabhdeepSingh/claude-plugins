---
name: security
description: >-
  Build-time security rules: which changes need a human's approval first, outbound fetches, leaked secrets, dependencies, model output and privacy. Use when building anything that touches auth, personal data, uploads, outbound requests, dependencies or model output. Not the after-the-fact review. Validation: [[code-standards]]; infra secrets: [[infra-standards]].
---

# Security — decide what can go wrong before deciding what to build

Explicit project instructions (its AGENTS.md, CLAUDE.md or other written rules) override this skill's workflow and style; follow them and skip that part of this. Existing code or habit alone never waives a trust boundary, a disclosure rule, a required human authorization or a data-safety rule: where the code conflicts with one of those, say so instead of following it.

These rules cover the decisions that are cheapest to get right while the trust boundaries are still being drawn. Parameterized queries and boundary validation live in [[code-standards]] section 9. Secret storage and CI credentials live in [[infra-standards]] sections 3 and 5.

## 1. Name the boundaries and the abuse cases

Before choosing defences, list where outside data enters (requests, uploads, webhooks, third-party responses, queue messages and model output) and what is reachable through each (credentials, personal data, payment data, admin capabilities, money movement). Then write the abuse case beside each use case and make it a test ([[tdd]]). "Upload a profile photo" pairs with "upload a 2GB file", "upload an SVG with a script tag" and "upload to someone else's profile".

## 2. Three tiers: always, ask first, never

- **Always**, with no permission needed: parameterized queries, boundary validation, passwords hashed with a current algorithm, TLS on, secrets from the store, least-privilege scopes.
- **Ask first.** A human approves before the change ships, because each of these widens the attack surface more than the diff suggests: a new authentication flow or a change to auth logic; storing a new category of sensitive data; a new external service integration; a CORS change; a file-upload handler; a rate-limiting or throttling change; granting an elevated permission or role.
- **Never**: secrets in source or logs, personal data in telemetry, disabled TLS verification, auth checks only on the client, security controls that fail open.

The ask-first tier is the same authorization posture as a merge: a widening needs a human's yes. In a non-interactive run, write the precise widening into the hand-off or ticket as a blocker and stop that part. Do not proceed on an approval nobody gave.

## 3. Outbound requests to user-influenced destinations

Any fetch whose URL a user can influence (webhooks, link previews, importers, "fetch my avatar from a URL") can be aimed at the internal network.

- Allowlist the scheme and host when the set of destinations is knowable.
- When arbitrary hosts are the feature, resolve every DNS record and reject unless every resolved address is globally routable. That one check rejects loopback, link-local (including the cloud metadata endpoint), private and unique-local ranges.
- Forbid redirects or re-validate every hop, because a public URL can redirect to an internal one.
- The HTTP client resolves DNS again after your check, so a short-TTL record can rebind to an internal address in between. For high-risk surfaces, resolve once and connect to the pinned address, or route through a filtering egress proxy. State this limit when it is left open.

## 4. A committed secret is a rotated secret

Once a credential reaches a remote (a pushed commit, a gist, a pasted log), treat it as compromised however fast the line is deleted, because clones, forks, caches and scrapers already have it. The order is fixed: revoke and reissue first, then purge it from history, then find how it got there and close that path.

## 5. Supply chain

A dependency is code that runs with your privileges.

- Install new dependencies with lifecycle scripts disabled, read what the pending scripts do, approve the minimum set and record the policy. Install scripts run before anything is imported.
- Do not auto-apply forced remediation such as `npm audit fix --force`. It crosses declared version ranges, which is an unannounced major upgrade.
- Triage advisories by whether the vulnerable function is reachable on your code path at runtime, not by severity alone. A deferred fix gets a written reason and a review date.
- Audits catch known advisories only. They do not catch a freshly malicious release or a typosquat. Read the package name character by character against the one you meant: a substituted character, an added or dropped hyphen, a look-alike letter from another alphabet. Check that the manifest and lockfile agree on the package manager, and stop on competing lockfiles.
- Before adopting a package, state roughly how many projects depend on it, how old its first release is and how many maintainers it has, and why it is still worth it. Do not take a release published in the last few days when an older one will do, because a hijacked publish is usually caught within days.
- Upgrade mechanics are in [[code-standards]] section 15.

## 6. Model and agent output is untrusted input

A model's output depends on retrieved documents, user prompts and tool results, so it has their trust level.

- Model output gets the same validation and encoding as user input before it reaches `eval`, SQL, a shell, `innerHTML` or a file path.
- The system prompt is not a security boundary. Enforce permissions in code.
- Anything in the context can be echoed back, so keep secrets and other tenants' data out of prompts.
- Give agent tools the minimum scope, gate destructive actions on confirmation, and validate every tool argument as boundary input. Bound tokens, request rate and loop depth so crafted input cannot run up cost or hang the system.
- In retrieval systems, partition embeddings per tenant and validate documents before indexing, so one tenant cannot retrieve another's documents and a poisoned page cannot steer later answers.

## 7. Privacy

Security asks whether an attacker can read the data. Privacy asks whether to hold it at all, and for how long.

- Collect a field only against a stated use.
- Set retention when the field is created, and delete on schedule from backups, caches, search indexes and analytics copies as well.
- Design the schema so one user's data can be found, exported and erased. A "delete my account" that flips a flag while personal data stays in five stores is the case to test for.
- Sending personal data to any vendor, including an analytics service or a model API, is sharing, and the user's consent gates it.
- Classify what is held as non-personal, personal or sensitive (health, finance, precise location, biometrics, government IDs, anything about minors), and make the tier visible in the schema.

## Maintenance

The package-manager flags and the attack patterns in sections 3, 5 and 6 were last checked in 2026-08. Check exact mechanisms against current documentation before relying on them.
