# Changelog

All notable changes to Reel are documented here.

## v1.6.0

Two new features and a scheduler verb cleanup.

### Added

- **`--socket <path>` for non-default runtime sockets.** Override the socket path when containerd / CRI-O / Docker lives somewhere other than the standard location. Pair with `--runtime`:

  ```bash
  reel list containers --runtime containerd --socket /var/run/containerd/containerd.sock
  ```

  Replaces the previous "no container runtime detected" wall when the runtime socket exists but at a non-default path.

- **Actionable runtime-detection errors.** When `--runtime X` is requested but unavailable, the error now tells you what went wrong and what to try next: EACCES (suggests `sudo` or group membership), ENOENT (suggests checking the daemon and pointing `--socket` at the right path), not-a-socket (verify path correctness). Previously all three rendered as the same generic "X socket is not accessible / check that the daemon is running".

- **VEX annotation now works in the scheduler.** Scheduled `upload sbom --scanners vuln,vex` annotations produce vex-hub-annotated CycloneDX SBOMs uploaded to S3, same as `reel export sbom --scanners vex` on the CLI. Previously the scheduler passed "vex" through to Trivy as a literal scanner name; Trivy silently ignored it and the SBOM uploaded to S3 had zero annotations. Fail-soft: on vex-hub error, ships un-annotated SBOM with `reel:vex_hub_status=unreachable` in the CycloneDX metadata.

### Changed

- **Scheduler verb namespace mirrors the CLI's export/upload split.** Before v1.6, `reel.io/schedule: "*/5 * * * * | export sbom --s3-bucket foo"` would silently upload to S3 in the scheduler, even though `reel export sbom --s3-bucket foo` on the CLI rejected that exact flag combination. Now both honor the split:
  - `export` = local-only artifact production (e.g., `export checkpoint -o /tmp/backups/`). Rejects `--s3-bucket` / `--s3-region` / `--s3-secret-name`.
  - `upload` = S3 upload. Requires an S3 destination (either `--s3-bucket` arg or `reel.io/s3-bucket` annotation).
  - Unknown verbs (typos) are rejected at parse time with a clear error.

  Existing annotations that upload to S3 via `export` need to migrate to `upload`:

  ```diff
  - reel.io/schedule: "*/5 * * * * | export sbom --scanners vuln --s3-bucket evidence"
  + reel.io/schedule: "*/5 * * * * | upload sbom --scanners vuln --s3-bucket evidence"
  ```

### Fixed

- **`reel -h` no longer ships two redundant command lists.** The hand-written list in the top section drifted out of sync with the cobra-auto list at the bottom — still mentioned the long-renamed `capabilities` subcommand and missed the (still-current) `upload` verb. The cobra-auto list is now the only one. Same cleanup applied to `reel export -h`, `reel upload -h`, `reel list -h`, etc.
- **`reel upload -h` example used the wrong annotation key and was missing a pipe separator.** Replaced with the correct shape: `reel.io/schedule: "0 */4 * * * | upload layer --s3-bucket vault"`.

## v1.5.3

Release-pipeline fix. No CLI behaviour or output changes.

### Fixed

- **`reel version` from inside an agent pod now reports the GA version, not the release-candidate suffix.** Before v1.5.3, `kubectl exec … reel version` returned strings like `v1.5.2-rc.3` even though the agent image was tagged `v1.5.2` and the Helm chart reported `appVersion: v1.5.2`. The CLI binary baked into the image was built during the RC phase and never re-stamped at GA promotion. The release pipeline now builds the CLI once with the GA marketing version and promotes the same bytes through to GA — so the version string in the image matches the image tag and the chart appVersion.

The CLI tarballs published to GitHub releases and Homebrew also flow through the same build-once-promote path now; tarball downloaders were unaffected by the previous bug but benefit from the pipeline-time speed-up (~3 min faster per release).

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
