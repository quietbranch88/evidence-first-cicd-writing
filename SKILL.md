---
name: evidence-first-cicd-writing
description: Turn real CI/CD measurements, workflow changes, deployment incidents, and release receipts into evidence-bounded Traditional Chinese case-study articles. Use for CI/CD writing and article review, not for modifying pipelines, publishing content, or claiming deployment without separate authorization and live evidence.
---

# Evidence-first CI/CD writing

Write technical case studies from repository and provider evidence rather than
generic CI/CD advice.

## Before writing

Inspect the relevant evidence when it is available:

- workflow definitions;
- exact commit and pull request;
- CI job names, conclusions, queue time, execution time, and retained timings;
- deployment provider identity;
- readiness and database results;
- production marker and asset readback;
- sanitized release receipt.

Treat commit messages, old summaries, and provider dashboards as leads rather
than current proof. Re-read mutable state before describing it as current.

## Keep states separate

Do not collapse these states:

1. implemented;
2. locally tested;
3. exact-head CI green;
4. merged;
5. exact-main CI green;
6. provider deployed;
7. production readback verified;
8. customer flow verified.

CI success does not establish deployment. Provider success and health do not
establish exact source identity. Production readback does not establish a
complete customer journey.

## Label measurements

Identify every number as a local sample, CI sample, production sample, repeated
result, stable percentile, or estimate. One run is not a stable percentage or
percentile. A no-runtime-change verification is not comparable to a full
deployment.

## Choose the article mode

- For CI cost, self-hosted runners, path-aware checks, parallel lanes, virtual
  time, and cache operations, read
  [references/article-01-ci-cost-and-speed.md](references/article-01-ci-cost-and-speed.md).
- For Exact-SHA, provider identity, deployment ordering, runtime/control
  digests, production readback, and release receipts, read
  [references/article-02-exact-sha-release.md](references/article-02-exact-sha-release.md).
- Before finalizing any factual claim, read
  [references/evidence-and-claim-boundaries.md](references/evidence-and-claim-boundaries.md).

## Writing shape

Lead with the real incident, cost, failure, or surprising result. Then explain:

- what was happening;
- what the evidence showed;
- what changed;
- which safety conditions stayed;
- what improved;
- what remains unverified.

Prefer short Traditional Chinese sentences and compact flow diagrams. Explain
jargon when first used. Preserve the author's voice and correct obvious errors,
but do not turn a project-specific choice into a universal best practice.

## Boundaries

Writing or reviewing an article does not authorize changing a repository,
merging a pull request, deploying a service, modifying a database, or publishing
the article externally.
