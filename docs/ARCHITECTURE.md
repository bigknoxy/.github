# Architecture

## Topology

```mermaid
flowchart LR
  subgraph repo[Any bigknoxy repo]
    caller[.github/workflows/security.yml<br/>~10 lines]
  end
  subgraph central[bigknoxy/.github]
    reusable[security-reusable.yml]
    dep[dependabot.yml<br/>bumps pinned SHAs]
  end
  caller -- "uses: @main<br/>with: language, has_docker" --> reusable
  dep --> reusable
  fleet[fleet-audit.yml<br/>weekly] -- "gh api" --> repo
  fleet --> issue[(tracking issue)]
  reusable --> gl[gitleaks]
  reusable --> dr[dependency-review<br/>PR only]
  reusable --> cq[CodeQL]
  reusable --> da[dep-audit<br/>per language]
  reusable --> tf[trivy-fs]
  reusable --> ti[trivy-image]
  gl & cq & da & tf & ti --> sarif[(Code scanning<br/>SARIF)]
  dr --> pr[(PR comment / block)]
```

## Job selection

```mermaid
flowchart TD
  A[inputs] --> B{language == none?}
  B -- yes --> C[gitleaks + dependency-review only]
  B -- no --> D[gitleaks, dependency-review, CodeQL, dep-audit]
  A --> E{has_docker?}
  E -- yes --> F[+ trivy-fs, trivy-image]
```

| language | CodeQL id | audit tool |
|---|---|---|
| go | go | govulncheck |
| node | javascript-typescript | npm / pnpm / yarn / bun audit |
| python | python | pip-audit |
| rust | rust | cargo audit |
| dotnet | csharp | dotnet list package --vulnerable |

## Design decisions

- **Pinned SHAs, bumped centrally.** Only this repo's Dependabot touches action versions.
- **Blocking by default.** Every scanner exits non-zero on findings. Suppress per-repo with
  `.gitleaksignore`, `.trivyignore`, or CodeQL config, never with `continue-on-error`.
- **dependency-review only on pull_request.** The action needs a base/head diff.
- **`@main` reference.** Trade-off: instant rollout of fixes vs. a bad push breaking every
  repo. Mitigated by branch protection on this repo.

## Security controls

| control | where |
|---|---|
| least-privilege `permissions: contents: read` | top of reusable + caller |
| `security-events: write` scoped to SARIF jobs only | codeql, trivy-* |
| `pull-requests: write` scoped to dependency-review | dependency-review |
| `--ignore-scripts` on installs | node audit |
| secrets: only `GITHUB_TOKEN` via `secrets: inherit` | gitleaks |

Last updated: 2026-09-04
