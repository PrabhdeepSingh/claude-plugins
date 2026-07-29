---
name: infra-standards
description: >-
  Infrastructure, container, and CI/CD discipline — IaC, Dockerfiles and compose files, CI pipelines, platform deploy config, env/secrets handling. INVOKE PROACTIVELY when creating or editing any such file (*.tf, *.bicep, Dockerfile, docker-compose*, .github/workflows/*, vercel.json) — even when nobody says "infra". Not for application code ([[code-standards]]), telemetry ([[observability]]), or schema changes ([[safe-migrations]]).
---

# Infra standards — the outage-shaped mistakes are all preventable at review time

Infrastructure changes fail differently from code changes: there's no unit test that catches a deleted database, and the blast radius of one line can be the whole environment. These rules exist because every one of them maps to a class of incident — drift nobody can reproduce, a plan nobody read, a secret in an image layer, a pipeline greened by disabling its checks. The theme throughout: **infrastructure is code — reviewed, planned, least-privileged, and boring.**

## How to apply this

When touching any infra surface, hold three questions: *what is the blast radius of this change* (what can it delete or replace), *where do the secrets live* (never in the artifact), and *would this survive being run twice* (idempotency). Run the self-check before shipping.

---

## 1. Everything is code — no clickops

Any change made by hand in a cloud console/portal is **drift**: invisible to review, unreproducible, and silently overwritten (or silently load-bearing) the next time the IaC applies. The rules:

- Infrastructure changes go through the IaC files in the repo, PR-reviewed like any code.
- A genuine emergency hand-fix is allowed — and **codifying it is part of the same incident**, not a follow-up ticket that dies in the backlog. Until it's in code, the fix doesn't exist.
- If IaC and reality disagree, reality is the bug: reconcile toward the code (import the resource or fix the code), never by editing the console to match.

## 2. Read the plan before you apply — every time

`terraform plan` / `az deployment ... what-if` / CloudFormation change sets exist because IaC's most dangerous property is that **a one-line diff can mean "destroy and recreate."** Changing an immutable attribute (a resource name, an AZ, a SKU) doesn't edit the resource — it replaces it, and for a database that's data loss wearing a green checkmark.

- Never apply a plan you didn't read. In the plan output, `destroy` and `replace` lines are the whole review — explain each one or stop.
- In CI, the plan is posted for human review before apply on protected environments; auto-apply is for dev sandboxes only.
- **State is sacred** (Terraform): remote backend with locking, never hand-edited, never committed to the repo. Use `terraform state` commands (or `import`/`moved` blocks) for surgery — a corrupted state file makes the plan lie to you.

## 3. Secrets and config: the artifact never contains them

Config varies per environment; secrets are config with consequences. The discipline (the runtime-code side of this lives in [[code-standards]] sections 8–10):

- **Config comes from the environment** — env vars or platform config (App Service settings, Vercel project env), never hardcoded per-environment values inside the artifact.
- **Secrets come from a secret store** — Azure Key Vault, AWS Secrets Manager, Vercel encrypted env, CI's secret mechanism. Never in: source, `.tfvars` committed to the repo, Dockerfile `ENV`/`ARG`, pipeline YAML, or logs (`set -x` in a CI script echoes everything — including the secret you just interpolated).
- **Each environment gets its own secrets.** A staging leak must not be a production leak.

```bicep
// Avoid: the connection string is now in git history forever
param dbConnectionString string = 'Server=prod-db;User=admin;Password=hunter2;'

// Prefer: the template references the store; the value never touches the repo
param keyVaultName string
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = { name: keyVaultName }
// app setting references: '@Microsoft.KeyVault(SecretUri=${kv.properties.vaultUri}secrets/db-connection)'
```

## 4. Dockerfile standards

Images are built once and inspected never — so the mistakes ship silently. The baseline:

- **Pin the base image** — `node:22.12-alpine`, never `:latest` (unreproducible builds that change under you). For supply-chain-sensitive builds, pin the digest.
- **Multi-stage**: build tools, compilers, and dev dependencies stay in the build stage; the runtime stage carries only the artifact and production deps. This is both size and attack surface.
- **Run as non-root**: create and `USER` an unprivileged user in the runtime stage. A container escape from root is a much worse day.
- **Layer order = cache strategy**: copy dependency manifests and install *before* copying source, so a source-only change doesn't reinstall the world.
- **No secrets in any layer** — build `ARG`s and deleted files persist in layer history; use build-time secret mounts or fetch at runtime from the store (section 3).
- **`.dockerignore` exists** and excludes `.git`, `node_modules`, `.env*`, local config — both for size and to keep secrets from being COPY'd by accident.
- **Pinning applies where images are *deployed* too** (compose files, App Service/container-service config): reference a unique tag or digest per release, never `:latest` — rolling back to "latest" is rolling back to nothing.

```dockerfile
# Avoid: latest tag, root, single stage, source copied before deps, .env baked in
FROM node:latest
COPY . .
RUN npm install
CMD ["node", "server.js"]

# Prefer
FROM node:22.12-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

FROM node:22.12-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=build --chown=app:app /app/dist ./dist
COPY --from=build --chown=app:app /app/node_modules ./node_modules
USER app
CMD ["node", "dist/server.js"]
```

## 5. CI/CD pipeline discipline

The pipeline is code: reviewed, least-privileged, and honest.

- **Never green the pipeline by weakening it.** Skipping the failing test, `continue-on-error: true`, commenting out the failing step, `|| true` — these are [[code-standards]] section 12's suppression rule wearing a hard hat: the signal goes away, the problem ships. A red pipeline is telling you something; fix the cause or genuinely justify (narrow scope, stated reason, removal condition) — never bare-suppress.
- **Fail fast, cache honestly**: cheap checks (lint, types, unit) before expensive ones; dependency caches keyed on the lockfile hash so a lockfile change busts the cache instead of poisoning it.
- **Build once, promote the artifact.** The image/bundle that passed staging is byte-for-byte what reaches production. Rebuilding per environment means deploying something nothing ever tested.
- **Least-privilege tokens**: scope CI credentials to what the job does — in GitHub Actions, set the `permissions:` block explicitly (default it to `contents: read` and grant upward per job) and prefer OIDC federation to cloud providers over long-lived stored keys.
- **Pin third-party actions/tasks** to a version (for supply-chain-critical workflows, a commit SHA) — an unpinned action is someone else's push access to your pipeline.
- Pipeline secrets go through the platform's secret mechanism and are never echoed; remember `set -x` and debug logging both leak.

## 6. Least privilege, everywhere

Every identity — service principals, IAM roles, managed identities, deploy keys — gets the narrowest scope that does the job. `*:*` and owner-role-because-it-was-easier are how a leaked dev credential becomes a company incident. When granting, name the exact operations; when reviewing, treat any wildcard as a finding that needs a justification.

## 7. Environments: parity, previews, and idempotency

- **Staging mirrors production's shape** — same IaC modules, smaller SKUs. Config differences are parameters, not divergent templates; the thing you tested should be structurally the thing you ship.
- **Ephemeral previews** (Vercel preview deploys, PR environments) are production-shaped too: preview env vars are a separate scope from production's ([[seo-standards]] already requires previews to be non-indexable).
- **Idempotency is the IaC contract**: applying twice with no changes = zero diff. If a second apply shows changes, something is non-deterministic (a timestamp, an unpinned version, drift) — fix it before it hides a real diff.

---

## Self-check before you ship an infra change

- Is every change expressed in code and reviewed — no console/portal edits left uncodified?
- Did you read the plan/what-if, and can you explain every `destroy`/`replace` line in it?
- Zero secrets in source, tfvars, Dockerfiles, pipeline YAML, images, or logs — everything from a secret store, scoped per environment?
- Dockerfile: pinned base, multi-stage, non-root, deps-before-source layers, `.dockerignore` present — and deploy config references a unique tag per release, never `:latest`?
- Pipeline: no skipped/suppressed checks without a stated justification; artifact built once and promoted; `permissions:` scoped; third-party actions pinned?
- Any wildcard permission grants that lack a written justification?
- Would applying this twice produce zero diff?

## Provenance and maintenance

The disciplines are durable; the tool surfaces drift. Last verified 2026-07:

- Tool syntax in examples (Terraform plan/state behavior, Bicep Key Vault references, GitHub Actions `permissions:`/OIDC, `npm ci`/prune flags) — re-verify against each tool's current docs before leaning on exact syntax.
- Base-image tags in examples are illustrative — pin to whatever is current-LTS at the time of writing, not to this file's example.
