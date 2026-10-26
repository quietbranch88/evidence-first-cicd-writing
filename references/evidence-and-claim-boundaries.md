# Evidence and claim boundaries

Use this reference before finalizing a CI/CD article.

## Evidence ladder

| Evidence | Supported claim | Does not prove |
|---|---|---|
| Source inspection | The workflow or guard exists | It executed successfully |
| Local test | The tested behavior passed locally | CI or production passed |
| Exact-head CI | The PR head passed the named checks | The merge SHA passed |
| Exact-main CI | The merge SHA passed the named checks | It deployed |
| Provider success | The provider reports a completed deployment | Exact source identity unless read back |
| Readiness | The checked dependencies were ready at that time | Complete customer behavior |
| Production marker and assets | The public artifact matches the expected source | Authenticated or paid flows work |
| Customer-flow E2E | The exercised path completed | Untested failure paths or reliability percentiles |

## Performance wording

- State the environment, revision, workload, and sample count.
- Say `single sample` when there is one observation.
- Do not convert one result into a stable percentage, percentile, capacity, or
  cost claim.
- Keep runner queue time separate from execution time.
- Keep a no-op or no-runtime-change verification separate from a full build and
  publication.
- Keep local virtual-time improvements separate from production latency.

## Release wording

- `Merged` means the pull request entered the target branch.
- `CI green` means the named checks passed for the named SHA.
- `Deployed` requires provider completion for the named SHA.
- `Live readback verified` requires the named production marker, readiness, and
  relevant asset or dependency checks.
- `Customer flow verified` requires the real user-facing chain and observable
  outcome; it is not implied by health endpoints.

When evidence is narrower than the intended sentence, narrow the sentence.
