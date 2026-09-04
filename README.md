# bigknoxy/.github

Org-wide GitHub Actions for bigknoxy repositories.

## security-reusable.yml

One reusable workflow every repo calls instead of maintaining its own scanner list.
Jobs: gitleaks, dependency-review (PRs), CodeQL, language-native dependency audit,
trivy filesystem + image scan (when `has_docker`). Every third-party action is pinned
to a commit SHA and bumped by Dependabot here, so a bump lands once for every repo.

### Caller (copy into `.github/workflows/security.yml`)

```yaml
name: Security
on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'
  workflow_dispatch:
permissions:
  contents: read
jobs:
  security:
    uses: bigknoxy/.github/.github/workflows/security-reusable.yml@main
    permissions:
      contents: read
      security-events: write   # SARIF upload (CodeQL, trivy)
      pull-requests: write     # dependency-review PR comment
    with:
      language: node        # go | node | python | rust | dotnet | none
      has_docker: false
    secrets: inherit
```

### Inputs

| input | default | notes |
|---|---|---|
| `language` | required | selects CodeQL language and the audit tool |
| `has_docker` | `false` | enables trivy-fs and trivy-image |
| `dockerfile` | `Dockerfile` | |
| `docker_context` | `.` | |
| `working_directory` | `.` | where the manifest lives |
| `codeql_language` | derived | override CodeQL language id |
| `node_audit_level` | `high` | |
| `fail_on_severity` | `high` | dependency-review threshold |

The job-level `permissions` block is required: a called workflow can never hold more than
the caller grants it, and GitHub fails the run at startup if it asks for more.

Then add the job names (`gitleaks`, `codeql`, `dep-audit (<lang>)`, `dependency-review`,
`trivy-fs`, `trivy-image`) as required status checks in branch protection.

Architecture and rollout: see [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## fleet-audit.yml

Weekly sweep of every public, non-archived repo: does it call the reusable workflow,
are secret scanning, push protection and Dependabot alerts on, and how many open
critical/high Dependabot alerts. Upserts one tracking issue here titled
"Fleet security audit". Requires the `FLEET_AUDIT_TOKEN` secret (fine-grained PAT,
read on all repos plus issues write here); without it the job skips with a warning.
Run on demand from the Actions tab.
