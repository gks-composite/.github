# trivy-scan (composite action)

Org-owned composite action that wraps [Trivy](https://github.com/aquasecurity/trivy) for
**filesystem (source/dependency)**, **container image**, and **IaC/config** scanning with
consistent severity gating. It replaces the personal `Xander-Rudolph/trivy-scan` dependency so
the whole enterprise consumes a single, reviewed action from the `gks-composite` org.

The underlying `aquasecurity/trivy-action` is pinned by commit SHA (v0.36.0) for supply-chain
integrity.

> Lives inside the `gks-composite/.github` repository and is referenced by path:
> `uses: gks-composite/.github/actions/trivy-scan@main`

## Usage

```yaml
- name: Scan source (fs)
  uses: gks-composite/.github/actions/trivy-scan@main
  with:
    scan-type: fs
    scan-ref: .
    severity: HIGH,CRITICAL
    exit-code: '1'          # fail the job on HIGH/CRITICAL

- name: Scan a built image
  uses: gks-composite/.github/actions/trivy-scan@main
  with:
    scan-type: image
    scan-ref: ghcr.io/facefirst-engineering/myservice:sha-abc1234
    severity: CRITICAL
    exit-code: '1'
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `scan-type` | `fs` | `fs`, `image`, `config`, or `repo`. |
| `scan-ref` | `.` | Path (fs/config/repo) or image reference (image). |
| `severity` | `HIGH,CRITICAL` | Comma-separated severities to report and gate on. |
| `exit-code` | `0` | `0` = report only; `1` = fail on any finding of the given severity. |
| `ignore-unfixed` | `false` | Skip vulnerabilities without an available fix. |
| `scanners` | `vuln,secret` | Trivy scanners: `vuln,secret,misconfig,license`. |
| `format` | `table` | Output format (`table`, `sarif`, `json`, ...). |
| `output` | `` | Optional output file (e.g. `trivy-results.sarif`). |
| `trivyignores` | `` | Comma-separated `.trivyignore` file paths. |
| `working-dir` | `` | Back-compat alias; overrides `scan-ref` for fs/config scans. |
| `summary` | `true` | Append a short summary line to the job summary. |

## Tiered gating convention

The enterprise standard (implemented by the `Docker-Build-Push` reusable workflow) is:

| Trigger | `severity` | `exit-code` |
|---|---|---|
| Pull request | `CRITICAL` | `1` (fail on CRITICAL) |
| Push to default branch / release tag | `HIGH,CRITICAL` | `1` (fail on HIGH+CRITICAL) |
| Other branches | `HIGH,CRITICAL` | `0` (report only) |

Call this action directly with those values, or use the shared reusable workflows
(`Docker-Build-Push.yml`, `Global-Scan.yml`) which apply them automatically.

## Related

- `gks-composite/.github` → `Docker-Build-Push.yml`, `Global-Scan.yml` reusable workflows.
- `gks-composite/global-version` → uniform version numbering.
