# Changelog

All notable changes to Reel are documented here.

## v1.4.0

Overhauls the GitHub Action's output and closes the SARIF loop with GitHub Code Scanning.

### GitHub Action

- **Scan summary on the run page** — every run now appends a markdown summary to `$GITHUB_STEP_SUMMARY` covering SBOM, SARIF, CBOM, and malware scans. Shows per-scan findings, status (`[PASS]` / `[FAIL]` / `[WARN]` / `[INFO]`), and the top vulnerabilities by severity when findings exist. Runs on every invocation — no more bare green-check pass with zero visibility, no more grep-the-artifact JSON on fail.
- **SARIF → Code Scanning** — when `scan-types` includes `sarif`, findings now upload to the repo's Security tab via `github/codeql-action/upload-sarif@v3`. Enables dismissal (persists across runs via GCS fingerprinting), PR-level gating through the "Code Scanning results" status check (configurable severity threshold in repo settings + branch protection), and cross-scanner visibility alongside CodeQL. **Requires callers to grant `security-events: write` permission** on the job.
- **SBOM severity now reflects max across all rating sources** — CycloneDX emits per-source severity ratings (nvd, alma, redhat, ubuntu, …). Summary previously implicitly picked the first source, which was non-deterministic and under-reported (e.g. a CVE rated `medium` by one source and `critical` by another showed as `medium`). Now shows the max severity across sources.
- **`scanners` input default flipped** from `''` to `'vuln'`. Without an explicit scanner, reel's SBOM emits zero vulns regardless of image content; the previous default silently produced ungated pipelines. Consumers who set `scanners:` explicitly are unaffected.
- **New `cbom-file` output** for symmetry with `sbom-file` / `sarif-file` / `malware-file`.
- **`vuln-count` and `malware-count` outputs now emit on every run** regardless of `fail-on-findings`. Previously they were only emitted when `fail-on-findings: true`.

### README

- Required permissions documented for SARIF consumers.
- Two-layer gating (workflow gate + PR gate) explained.
- Outputs table added.

### Compatibility

- Callers using `scan-types: sarif` must add `security-events: write` to their job permissions. Without it, the new upload step fails.
- Callers relying on `scanners: ''` (explicit empty) to skip vuln scanning should now pass the scan types they want (e.g. `scanners: secret,license`); the default behavior is now to scan vulns.

## v1.3.0

Ships alongside reel CLI v1.3.0. Two new pass-through inputs surface CLI flags that just landed on the reel side, plus a dead-script cleanup.

### GitHub Action

- **`local` input** — pass-through for `reel export --local`. When `true`, image scans are restricted to the runner's container daemon and fail fast if the image isn't present. Default is `false`. With no flag set, reel CLI v1.3.0's new local-first-with-fallback default already finds images built into the runner's daemon, so CI workflows that build-then-scan no longer need any special configuration.
- **`ignore-unfixed` input** — pass-through for `reel export --ignore-unfixed`. Applied to `sbom` and `sarif` scan types only (cbom and malware don't carry a "fixed" concept). When `true`, unfixed HIGH/CRITICAL CVEs don't trip the `fail-on-findings` gate — useful for release pipelines that want to block on fixable issues without breaking on upstream patch lag.

### Compatibility

- Requires reel CLI v1.3.0 or later in the runner (delivered via `reel-version: latest` by default). Earlier CLIs on `cbom` / `malware` don't recognize `--local`; setting `local: true` against a pinned older version will error.

### Removed

- `test-action.sh` deleted. The jq-based gate tests duplicated trivial expressions and the reel-backed tests depended on drifting upstream CVE state; nothing in CI invoked the script. End-to-end validation now rides on downstream callers (e.g. vex-hub's release workflow).

## v1.2.0

### Standalone CLI — macOS container scanning

- **Scan running containers on macOS.** `reel export sbom|cbom|malware <container>` now works against Docker Desktop, OrbStack, Colima, and Rancher Desktop. Previously the container's overlay filesystem lived inside the Linux VM and wasn't reachable from the host. The CLI now uses the Docker API as a fallback when direct filesystem access isn't available — `docker commit` + `docker save` + `trivy image --input` for SBOMs (preserves the writable layer); `CopyFromContainer(/)` + safe tar extract for CBOM and malware. Linux behavior is unchanged: GraphDriver / `/proc/{pid}/root` remain the fast path.
- **Auto-detection of Docker socket on macOS.** Probes `~/.orbstack/run/docker.sock`, `~/.docker/run/docker.sock`, `~/.colima/default/docker.sock`, `~/.rd/docker.sock`. `DOCKER_HOST=unix://...` is now honored on all platforms.
- **Intel Mac (darwin/amd64) binary.** `reel_darwin_amd64.tar.gz` ships alongside `reel_darwin_arm64.tar.gz`. `brew install getreeldev/tap/reel` now works on both Intel and Apple Silicon Macs.

### GitHub Action

- **`fail-on-findings` input** — fail the workflow step if any vulnerabilities or malware survive the scan filters. Scans always complete and produce artifacts; the gate evaluates after.
- **`scanners` input** — passthrough to Trivy `--scanners` (vuln, secret, license, config, all).
- **`severity` input** — passthrough to Trivy `--severity` (LOW, MEDIUM, HIGH, CRITICAL).
- **Structured outputs** — `sbom-file`, `sarif-file`, `malware-file` (paths), `vuln-count`, `malware-count` (counts).

### Notes

- `reel export checkpoint|frame|memory` still require agent mode on Linux — they need CRIU and kernel access that cannot be emulated via the Docker API.
- Runtime tar extraction is conservative: zip-slip rejected, absolute/`..` symlinks skipped, device nodes skipped, setuid bits stripped, 5 GiB size cap.

## v1.1.0

### New Features

- **Cron shorthand in schedule annotations** — `@every 1h`, `@daily`, `@hourly`, `@weekly`, `@monthly`, `@midnight` now work alongside standard 5-field cron expressions.
- **Linux arm64 CLI binary** — standalone CLI now ships for both `linux/amd64` and `linux/arm64`.
- **Automated Helm chart publishing** — chart is now versioned, tagged, and pushed to `oci://docker.io/getreel/helm` automatically on release.

### Improvements

- **ClamAV auto-download on macOS** — downloads the official universal `.pkg` from ClamAV GitHub releases. No Homebrew required. Binaries are patched with the correct rpath and ad-hoc re-signed automatically.
- **ClamAV auto-download on Linux arm64** — downloads the official `aarch64.deb`. Previously required manual installation.
- **CVD certificate handling** — uses `CVD_CERTS_DIR` environment variable instead of writing to system paths. No sudo required for ClamAV database updates.
- **Homebrew formula** — now includes `linux/arm64` alongside `darwin/arm64` and `linux/amd64`.
- **Test fixture optimization** — pre-pull images at test start, `imagePullPolicy: IfNotPresent` on all fixtures, CRI-O registry config fix.

### Bug Fixes

- **Helm chart image tags** — init container tags now use `v` prefix to match Docker Hub (was causing `ImagePullBackOff` on `init-criu`).
- **Helm chart publish** — only updates `getreel/*` image tags, no longer overwrites `clamav/clamav` tag.

## v1.0.1

### Bug Fixes

- **macOS ClamAV PKCS7 errors** — fixed certificate verification noise during `freshclam` database updates by setting `CVD_CERTS_DIR` to the extracted cert from the official package.
- **Release pipeline** — `fetch-depth: 0` for tag checkout, idempotent release creation, explicit token auth for Helm chart push.

## v1.0.0

Initial release.

### Standalone CLI

- `reel export sbom` — SBOM generation via Trivy (CycloneDX, SPDX)
- `reel export cbom` — Cryptographic bill of materials
- `reel export malware` — ClamAV malware scanning
- `reel export sarif` — Vulnerability scan with SARIF output
- `reel list images` — Local container images
- `reel list workloads` — Running containers

### Agent Mode

- Kubernetes DaemonSet deployment via Helm (`oci://docker.io/getreel/helm`)
- Annotation-driven scheduling (`reel.io/schedule`)
- Container checkpoints via CRIU
- Filesystem layer capture and restore
- Frame archives
- Volatile data and metadata capture
- S3 evidence vault archival
- ClamAV sidecar for daemon-based malware scanning

### Distribution

- `brew install getreeldev/tap/reel` (macOS arm64)
- `curl` from `getreeldev/releases` (Linux amd64)
- `uses: getreeldev/releases@v1` (GitHub Action)
- `helm install reel oci://docker.io/getreel/helm` (Kubernetes agent)

### Infrastructure

- Netavark stub for buildah on Alpine
- Raised `fs.inotify.max_user_instances` for CRIU checkpoint support
- Runtime detection fix for CRI-O (`crio` without hyphen)
