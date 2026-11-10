---
name: infra-standards
description: >-
  House rules for infrastructure-as-code, Dockerfiles and compose files, CI pipelines, deploy config, secrets and feature-flag rollout. Use when creating or editing those files. Not for application code ([[code-standards]]), telemetry ([[observability]]) or schema changes ([[safe-migrations]]).
---

# Infra standards

The project's own instructions win: where its AGENTS.md, CLAUDE.md or established conventions say otherwise, follow them and skip that part of this.

One line of infrastructure can delete or replace a whole environment, and no unit test catches it. For any infra change, know what it can delete or replace, where its secrets live, and whether it gives the same result when run twice.

## 1. Everything is code

A change made by hand in a cloud console is drift: nobody reviewed it, nobody can reproduce it, and the next apply overwrites it or depends on it without saying so. Infrastructure changes go through the files in the repo and are reviewed like code. An emergency hand-fix is allowed, and writing it into code is part of the same incident. When code and reality disagree, reconcile toward the code by importing the resource or fixing the code.

## 2. Read the plan before applying

A one-line diff can mean "destroy and recreate". Changing an immutable attribute such as a name, zone or SKU replaces the resource, and for a database that is data loss.

- Do not apply a plan you have not read. Explain each `destroy` and `replace` line or stop.
- On protected environments the plan is posted for human review before apply. Auto-apply is for dev sandboxes.
- Terraform state lives in a remote backend with locking. It is not hand-edited and not committed. Use `terraform state` commands or `import` and `moved` blocks to change it.

## 3. Secrets and config stay out of the artifact

- Config comes from the environment. The file layout is: `.env.example` committed as the template; `.env` not committed; `.env.test` committed only with no real secrets; CI secrets in the platform vault; production secrets in the deployment platform.
- CI does not hold production secrets. A CI-only test database gets its own secret, not a hardcoded value.
- Secrets come from a secret store. They do not go in source, committed `.tfvars`, Dockerfile `ENV` or `ARG`, pipeline YAML or logs. `set -x` in a CI script prints the secret it just interpolated.
- Each environment has its own secrets, so a staging leak is not a production leak.
- When a secret reaches a remote anyway, [[security]] section 4 gives the order: revoke and reissue before purging history.

## 4. Dockerfiles

- Pin the base image to a version, and to a digest for supply-chain-sensitive builds. Do not use `:latest`.
- Use multiple stages so build tools and dev dependencies stay out of the runtime image.
- Run as a non-root user in the runtime stage.
- Copy dependency manifests and install before copying source, so a source change does not reinstall dependencies.
- No secrets in any layer. Build `ARG`s and deleted files persist in layer history, so use build-time secret mounts or fetch at runtime.
- Keep a `.dockerignore` that excludes `.git`, `node_modules`, `.env*` and local config.
- Where images are deployed (compose files, container-service config), reference a unique tag or digest per release. Rolling back to `:latest` rolls back to nothing.

## 5. CI/CD pipelines

- Do not make a pipeline green by weakening it: skipping the failing test, `continue-on-error: true`, commenting out the step, `|| true`. This is the suppression rule of [[code-standards]] section 12. Fix the cause, or justify a narrow suppression with a stated reason and removal condition.
- Run cheap checks before expensive ones, and key dependency caches on the lockfile hash.
- When a pipeline gets slow, improve it in this order, cheapest first: cache dependencies, run independent jobs in parallel, path-filter so only what changed runs, shard the tests, move slow tests to a schedule, and only then use larger runners.
- Build once and promote that artifact. Rebuilding per environment deploys something nothing tested.
- Scope CI credentials to the job. In GitHub Actions set `permissions:` explicitly, defaulting to `contents: read`, and prefer OIDC federation to stored cloud keys.
- Pin third-party actions to a version, or to a commit SHA for supply-chain-critical workflows.
- Pipeline secrets go through the platform's secret mechanism and are not echoed.

## 6. Least privilege

Every service principal, role, managed identity and deploy key gets the narrowest scope that does the job. Name the exact operations when granting. A wildcard grant needs a written justification.

## 7. Environments

- Staging uses the same modules as production with smaller sizes. Differences are parameters, not separate templates.
- Preview deployments have their own env-var scope, separate from production, and are not indexable ([[seo]]).
- Applying twice with no changes shows no diff. If a second apply shows changes, something is non-deterministic (a timestamp, an unpinned version, drift); fix it before it hides a real diff.

## 8. Feature flags and staged rollout

[[code-standards]] section 14 covers flag defaults. This section covers the lifecycle.

- Every flag has an owner and an expiry date set at creation, and is removed within a couple of weeks of full rollout.
- Do not nest flags, because the combinations go untested. CI exercises both states of any flag that gates behaviour.
- Write the rollback plan before the deploy: the trigger conditions (for example error rate more than double baseline, or p95 latency up by half), the mechanism, and how long it takes. A flag flip takes seconds, redeploying the previous artifact takes minutes, and a database rollback is its own [[safe-migrations]] plan. Confirm soon after a risky rollout that the rollback mechanism works.

## Maintenance

Tool syntax (Terraform plan and state behaviour, GitHub Actions `permissions:` and OIDC) was last checked in 2026-07. Check current documentation before relying on exact syntax.
