# ci-actions

Composite GitHub Actions shared by kernaxis repositories.

Pin every action by full commit SHA with the release as a comment, so Dependabot can propose upgrades:

```yaml
uses: kernaxis/ci-actions/govulncheck@<sha> # v1.0.0
```

Tool versions (actionlint, govulncheck, zizmor) are pinned in [`tools/`](tools) and updated by Dependabot here,
not in consumer repositories.

## Actions

| Action | Purpose | Requirements |
|---|---|---|
| `govulncheck` | Reachable Go vulnerability scan, optional normalized report | Go on PATH (`actions/setup-go`) |
| `workflow-lint` | actionlint and zizmor, optional normalized zizmor report | Go on PATH for actionlint, `python3` |
| `security-issues-sync` | Open, update and close bot-owned issues from the reports of the run | `issues: write`, `gh` |
| `publish_docker` | Push local images with skopeo when the remote digest differs | Docker, skopeo, jq |

## Security findings as issues

Scanners fail the job on findings in every mode. With `report: "true"`, they also upload a
`security-report-*` artifact; `security-issues-sync` turns the reports of the current branch head into issues
and closes the issues of findings that disappeared. Reports of a commit that is no longer the branch head
never change issues. Issue markers (`<!-- security-finding:v1:… -->`) are compatible with the per-repository
scripts these actions replace, so existing issues are adopted rather than duplicated.

```yaml
jobs:
  go-vulnerabilities:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<sha> # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-go@<sha> # v7.0.0
        with:
          go-version-file: go.mod
      - uses: kernaxis/ci-actions/govulncheck@<sha> # v1.0.0
        with:
          report: ${{ github.ref == 'refs/heads/main' }}

  workflows:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<sha> # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-go@<sha> # v7.0.0
        with:
          go-version-file: go.mod
      - uses: kernaxis/ci-actions/workflow-lint@<sha> # v1.0.0
        with:
          report: ${{ github.ref == 'refs/heads/main' }}

  finding-issues:
    if: always() && github.ref == 'refs/heads/main'
    needs: [go-vulnerabilities, workflows]
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      issues: write
    concurrency:
      group: security-finding-issues
      cancel-in-progress: false
    steps:
      - uses: kernaxis/ci-actions/security-issues-sync@<sha> # v1.0.0
```

`security-issues-sync` does not need a checkout. Its `label`, `bot-login` and `branch` inputs default to
`security`, `github-actions[bot]` and `main`.

## Development

```bash
python3 -I -m unittest discover -s tests -p 'test_*.py'
```

The CI runs the unit tests, lints this repository with `./workflow-lint` and smoke-tests `./govulncheck`.
A new action directory must also be added to `.github/dependabot.yml`.
