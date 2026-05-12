# Changelog

All notable changes to Reel are documented here.

## v1.5.2

CLI quality-of-life release. Three focused improvements; no API or behavioural changes.

### CLI

- **`reel schedule` is now scannable.** Default output is a compact table — one row per scheduled command, with cron expressions translated to plain English (`every 5 minutes` instead of `*/5 * * * *`). The previous per-pod block format is still available via `--detailed` for cases that need full command lines and dependency wiring.
- **`reel license` reports remaining time in human-readable units.** A 10-year license used to render as `85223h0m47s remaining`; it now reads `9 years 8 months remaining`. Sub-day durations stay in the native form (`23h14m`) since they're already legible.
- **`reel status` output: "Capabilities" → "Features".** Renamed the section header and the command's `--help` description to remove a reference to a removed `reel capabilities` subcommand. JSON output is unchanged.

## v1.5.1

Bug-fix release for K8s agent deployments. No new features.

### Fixed

- **Binary forensic uploads (`reel --agent upload layer/memory/checkpoint/frame`) now honor the `aws-credentials` K8s secret on non-EC2 clusters.** Previously these four upload types bypassed the secret and fell through to the default AWS credential chain — on GKE / AKS / on-prem K8s this meant timing out on IMDS. SBOM, CBOM, and other JSON exports were unaffected. If you scheduled binary-blob uploads to S3 from a non-EC2 cluster, you'll want to upgrade.

### Companion fixes elsewhere

- **Helm chart v1.5.1** ships two chart-side fixes that align the templates with what the agent binary has expected since Feb 2026: the shared volume now lands at `/opt/trivy/bin` (init-trivy was previously writing the binary to a path that wasn't on the shared volume, so the agent re-downloaded Trivy on every cold start), and the ClamAV virus DB persists across pod restarts via a `/var/tmp/reel/clamav` hostPath (was re-downloading ~110 MB on every restart). See the [chart changelog](https://github.com/getreeldev/helm/blob/main/CHANGELOG.md).

## v1.5.0

Adds VEX support to `reel export sbom`. Annotate vulnerability scans with vendor "not affected" / "fixed" / "exploitable" statements served by [vex.getreel.dev](https://vex.getreel.dev) — Red Hat, SUSE, Ubuntu, and Debian coverage — so downstream tooling (Trivy `--vex`, Dependency-Track, GitHub Code Scanning) suppresses noise automatically.

### CLI

- **New `--scanners vex` flag on `reel export sbom`** — opt-in scanner that POSTs the Trivy-generated CycloneDX SBOM to `vex.getreel.dev/v1/analyze` and merges vendor VEX statements back into the output. Vendor sources currently aggregated: Red Hat (CSAF + OVAL), SUSE (CSAF), Ubuntu (OpenVEX + OVAL), Debian (OVAL).
  - **CycloneDX output** (default): annotates `vulnerabilities[].analysis` blocks in place with `state`, `justification`, and `response` per the CycloneDX VEX schema. Trivy `--vex` and Dependency-Track pick up vendor `not_affected` / `fixed` / `affected` automatically.
  - **SARIF output** (`--format sarif`): vendor `not_affected` CVEs land in GitHub Code Scanning as pre-dismissed findings via `results[].suppressions[]`. Hidden by default, surfaceable on demand from the Security tab.
- **New `--vex-url` flag** to point `--scanners vex` at a self-hosted vex-hub (default: `https://vex.getreel.dev`). For air-gapped environments running their own `reel-vex` instance.
- **Fail-soft on hub unreachability** — network error or non-2xx from vex-hub → CLI exits 0 with the un-annotated SBOM and stamps `metadata.properties` with `reel:vex_hub_status=unreachable`. A transient hub outage never breaks a CI pipeline that asked for VEX annotation; consumers can detect the degraded mode by checking the metadata flag.

### Examples

```bash
# Annotate a UBI image scan with vendor VEX (CycloneDX, default format)
reel export sbom --image redhat/ubi9-minimal --scanners vuln,vex

# SARIF for GitHub Code Scanning, with vendor not_affected pre-suppressed
reel export sbom --image redhat/ubi9-minimal --scanners vuln,vex --format sarif -o reel.sarif

# Point at a self-hosted hub
reel export sbom --image redhat/ubi9-minimal --scanners vex --vex-url https://vex.example.internal
```

### Compatibility

- `--scanners vex` is **not included in `--scanners all`** — the hub call is an outbound network dependency, opt-in by design.
- Supported output formats with `--scanners vex`: `cyclonedx` (default), `sarif`. SPDX is rejected at parse time pending vex-hub spec support.
- `--scanners vex` auto-enables `vuln` (you need the underlying vulnerability data for VEX to annotate).
