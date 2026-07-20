# Changelog

All notable changes to Reel are documented here.

## v1.12.0

Official Talos Linux support, node-OS awareness, and checkpoint moved to opt-in.

**reel runs on Talos Linux.** Talos has no SSH, no shell, and a read-only root filesystem — so traditional "SSH in and run forensic tooling" is impossible by design. reel's agent, a DaemonSet, captures container state there like on any other node: SBOM, CBOM, malware, filesystem layers, and live memory dumps all work on **stock chart values**. It's continuously tested on a real Talos cluster each release. The same applies to other immutable-OS nodes such as Bottlerocket and Flatcar.

**The agent now knows what OS its node runs.** `reel --agent status` shows a `Node OS:` line (Talos, Bottlerocket, Flatcar, Ubuntu, RHEL, Amazon Linux, and more, detected from the Kubernetes node), and the agent reports this in telemetry so a fleet's node-OS mix is visible at a glance.

**Checkpoint/restore is now opt-in.** CRIU checkpointing installs a patched CRIU binary onto the host and relies on Kubernetes' still-beta `ContainerCheckpoint` feature, so it's now off by default — one less privileged host operation unless you use it. **To keep checkpoint, install the agent with `--set initCriu.enabled=true`** on nodes with a writable host filesystem; it is unavailable on immutable-OS nodes, where `reel status` says so plainly. Layer capture and live memory dumps don't use CRIU and are unaffected. This also means immutable-OS nodes need no special configuration — they run on the defaults.

Chart-side changes (init-criu disabled by default; the separate Talos values profile retired) are in the [helm changelog](https://github.com/getreeldev/helm/blob/main/CHANGELOG.md).

## v1.11.2

Cryptographic Bill of Materials (CBOM) accuracy fixes.

`reel export cbom` no longer mislabels modern cryptography. TLS 1.2 and 1.3 were being flagged as deprecated, ECDSA was flagged through a substring collision with DSA, and RSA keys weaker than 2048 bits weren't flagged at all — all now classified correctly. SSH public-key sizes are read from the key rather than assumed to be 2048 bits. CBOM output is now deterministic (re-scanning the same container yields identical algorithm ordering), and the scanner skips oversized files that only coincidentally match a crypto filename pattern.

**In-cluster malware scanning, fixed properly.** `reel --agent export malware` was reporting zero detections even for known malware. Three things were wrong and are now fixed: (1) the ClamAV sidecar ships with a **baked-in signature database**, so it has signatures immediately on a fresh node even when ClamAV's rate-limited CDN can't be reached; (2) the agent now **hands each file to the ClamAV daemon by file descriptor** (it opens the file itself, since the daemon can't reach container paths under `/proc`), instead of asking the daemon to open a path it can't see; and (3) the silent CLI fallback is **removed** — if the scanner can't run, `reel export malware` now **returns an error instead of a clean result**, so a broken scanner can never quietly report "no malware found." Standalone `reel export malware --image` is unchanged. See the [helm changelog](https://github.com/getreeldev/helm/blob/main/CHANGELOG.md) for the chart-side change.

## v1.11.0

reel now runs on arm64 Linux — Graviton, Ampere, and other 64-bit Arm hardware — in both standalone and agent mode.

**arm64 Linux support.** Releases now ship a `reel_linux_arm64.tar.gz` tarball alongside amd64, and the install snippet auto-detects your architecture. In Kubernetes, the agent and init images are multi-arch manifests: `helm install` on amd64, arm64, or mixed-architecture clusters pulls the right image per node — checkpoint/restore included. Malware scanning works on arm64 too, via a new multi-arch ClamAV sidecar image (the official ClamAV image is amd64-only).

**Image scans follow the host architecture.** `reel export sbom --image …` scans your machine's architecture by default; use `--platform` to override (for example `--platform linux/amd64` from an Arm machine).

**Security refresh.** ClamAV engine 1.5.3 (July 2026 patch), containerd 2.2.5, Go 1.25.11. The agent's Trivy database download now retries transient failures instead of blocking pod startup.

Chart-side changes (multi-arch ClamAV sidecar image swap) are in the [helm changelog](https://github.com/getreeldev/helm/blob/main/CHANGELOG.md).

## v1.10.0

Agent telemetry now reaches a live endpoint, a privacy-vetted metering heartbeat is added, the no-license grace period is reframed, and a checkpoint/layer/frame/memory upload bug is fixed.

**Agent telemetry now reaches its endpoint.** Earlier startup telemetry pointed at a sink that no longer existed, so every event was silently dropped. Telemetry now posts to PostHog and adds a periodic metering heartbeat — both carry only counts and privacy-vetted identifiers (hashed cluster UID, node, license id/status, agent version, capability flags), never workload names or vulnerability data. Opt out with `REEL_TELEMETRY_DISABLED=true`.

**Licensing: the in-binary period is a short grace, not the evaluation.** Running without a token now grants a 3-hour in-memory grace (was 30 minutes) that resets on restart. The real evaluation is a free, self-serve 30-day license you issue yourself; it persists across restarts. All features remain available during the grace period; the API returns 403 once it expires.

**Fixed: checkpoint, layer, frame, and memory uploads to S3.** These artifacts stage to a local file before the S3 push, but the staging directory wasn't created — so scheduled `upload checkpoint` (and layer/frame/memory) failed with "no such file or directory" and never reached the evidence vault. SBOM/CBOM uploads, which stream directly, were unaffected.

## v1.9.1

Security hardening for the in-cluster agent, two fixes, and the removal of `reel upload sarif`.

**The agent's HTTP API now binds to localhost (`127.0.0.1`) by default.** The agent runs as a privileged node-root daemon and its API has no caller authentication, so exposing it across the pod network was unnecessary surface. Nothing changes for normal use — the in-pod scheduler, the MCP loopback, and `reel --agent` (which runs inside the agent pod) all talk to it over localhost. If you deliberately front the API with your own authentication, the new `--bind` flag opts back into a wider interface: `reel start server --bind 0.0.0.0`.

**`reel upload sarif` has been removed.** SARIF is a code-scanning report format, not forensic evidence, so it no longer has an evidence-vault upload path. Generating SARIF is unchanged — `reel export sbom --scanners vuln,vex --format sarif` (with vendor VEX suppressions) and `reel export sarif --image …` both work as before.

**Fixes:**
- IPv6 connections in `volatile` dumps were byte-swapped (loopback `::1` showed as `::100`); they now render correctly.
- SBOM scans no longer fail when Trivy emits a newer CycloneDX version than the bundled parser recognises — the document is passed through intact.

Companion chart release: the Helm chart tightens the agent's secret permissions to read-only-by-name and ships an opt-in default-deny NetworkPolicy — see the [Helm changelog](https://github.com/getreeldev/helm/blob/main/CHANGELOG.md).

## v1.9.0

Reel can now act as an MCP (Model Context Protocol) server, exposing its container-extraction capabilities to Claude Code, Cursor, Continue, and other MCP-aware AI clients.

**`reel start mcp`** launches a stdio MCP server you can wire into your AI client's config:

```json
{
  "mcpServers": {
    "reel": { "command": "reel", "args": ["start", "mcp"] }
  }
}
```

Seven tools are exposed in standalone mode: `whoami`, `health`, `list_workloads`, `list_images`, `sbom`, `cbom`, `malware`. Each artifact tool offers three destinations — `inline` for short responses the AI reads directly, `local` for writes to `~/.reel/cache/`, `s3` for evidence persistence. Discovery tools support `summary=true` to return aggregate counts cheaply before pulling full rows.

VEX annotation is wired into the MCP `sbom` tool: pass `scanners: ["vuln","vex"]` to get vendor VEX statements attached to Trivy findings, same path the CLI uses against vex.getreel.dev.

**CLI rename:** the long-running `reel server` command is now `reel start server`. The `start` verb is the new parent for long-running processes (currently `server` and `mcp`). No backward-compat shim — pre-customer rename.

**Fixes:**
- `malware` tool reported scan duration in nanoseconds under a field named `_ms`. Now correctly in milliseconds.
- MCP `sbom`/`cbom` tools now reject `pod`/`namespace` args in standalone mode with a clear error instead of falling through to a confusing low-level message.

## v1.8.0

Architectural refactor. CLI commands are unchanged from a user point of
view — same flags, same outputs. The change is in *where* the work
happens: the agent's HTTP API now owns S3 uploads end-to-end, which sets
up the upcoming MCP gateway and AI-agent integrations.

### Changed

- **`reel --agent upload <artifact>` commands now POST to the agent
  server.** Same flags, same behaviour. Previously the CLI ran the scan
  + S3 upload pipeline in-process; now it sends one request to the
  agent and the agent does the work. The user-visible CLI surface is
  identical.

### Added

- **Inventory `--hash` / `--hash-algorithms` flags now actually work.**
  Before this release the flags parsed but were silently ignored —
  inventories produced via `reel --agent upload inventory --hash` had
  no per-file cryptographic hashes. The flags now propagate end-to-end
  and the resulting CycloneDX manifest carries hashes from the
  requested algorithms (`md5,sha1,sha256,sha512`).

### Breaking — scheduler only

- **`reel.io/schedule: "... | export <artifact>"` annotations now
  error.** The scheduler's `export` verb (local-only artifact
  production on agent-pod disk) has been removed. It was never
  documented on the website — only the `upload` verb is — and the use
  case was never real (pod restart evaporates local disk).
  Existing annotations using `export` should be rewritten as `upload`
  with an S3 destination:

  ```diff
  - reel.io/schedule: "*/5 * * * * | export sbom --scanners vuln"
  + reel.io/schedule: "*/5 * * * * | upload sbom --scanners vuln --s3-bucket vault"
  ```

  Standalone `reel export <artifact> ...` CLI commands are unaffected —
  the `export` verb stays on the CLI side, only the scheduler stopped
  accepting it.

## v1.7.2

Security patch + quality-of-life. Upgrade is recommended for anyone running
`reel export malware --image` or `reel export cbom --image` against
untrusted images, or running the agent with scheduled `upload malware` /
`upload cbom` jobs.

### Security

- **Tar extraction now rejects symlinks with absolute or `..`-bearing
  targets.** Before this release, a malicious container image could craft
  a tar layer containing a symlink (e.g. `evil → /etc/passwd`) followed by
  a regular-file entry of the same name; extraction would follow the
  symlink and overwrite the host file. Affected paths: standalone image
  extraction for malware and CBOM scans (`reel export malware --image`,
  `reel export cbom --image`) and every scheduled `upload malware` /
  `upload cbom` job in agent mode. Agent runs as root in its pod — any
  host file writable from the pod (incl. mounted hostPath volumes) was
  reachable. Patched at the extractor; all six call paths in the codebase
  now share a single hardened implementation.

### Fixed

- **Pipe-keepalive extended to `inventory`, `malware`, `metadata`, and
  `volatile` exports.** v1.7.1 fixed `sbom | claude "..."` etc.; the other
  JSON-shaped exports had the same stdin-readiness timeout problem when
  the agent or scanner took >3 s to produce its first byte. Same shim,
  same byte-identical behavior for TTY and `-o file` paths.

### Changed

- **One tar extractor codebase-wide.** Internal refactor — no user-visible
  CLI change. Three extractors were collapsed to one hardened
  implementation. Forensic-friendly defaults: extracted files preserve
  ownership and timestamps on a best-effort basis (silent no-op as
  non-root, like GNU tar).

## v1.7.1

Quality-of-life release. The only user-visible change is a pipe-keepalive
shim that lets long-running `reel export sbom|sarif|cbom` survive
downstream stdin-readiness timeouts.

### Fixed

- **`reel export sbom|sarif|cbom ... | claude "..."`** (and any other pipe
  consumer with a stdin-readiness timeout) no longer fails when Trivy's
  cold-start takes longer than the consumer's grace window. `claude`'s
  default is 3 seconds; cold image scans can take 4–8 s before the first
  byte. New behaviour: when stdout is a pipe AND the output format is
  whitespace-tolerant JSON (`cyclonedx`, `spdx-json`, `sarif`, `json`,
  default), the CLI emits a single `\n` every second until the real
  payload starts streaming. JSON parsers tolerate leading whitespace per
  RFC 8259, so the keepalive bytes are invisible to downstream consumers.
  TTY output, regular files (`-o`, `> file`), and unsafe formats (`text`,
  `table`, tag-value `spdx`) are byte-identical to v1.7.0.

## v1.7.0

Major CRIU detection and installation overhaul. Fixes the segfault class
of bugs we kept hitting on hosts with different glibc versions than the
init-criu build base, and replaces the brittle env-var-trust capability
check with a marker-file contract between init-criu and the agent.

### Fixed

- **Checkpoint no longer segfaults on hosts with a different glibc than
  the init-criu build base.** init-criu previously bundled `libc.so.6` in
  `/opt/reel/lib`, which the host's `ld-linux` would load via
  `LD_LIBRARY_PATH` and segfault on symbol/layout mismatch. The bundle
  now blacklists glibc-coupled libs (libc, libpthread, libdl, librt,
  libm, libresolv, libutil, libnss_*, ld-linux); the host's libc loads
  cleanly and CRIU runs. Symptom was `criu swrk failed: signal:
  segmentation fault (core dumped)` from every checkpoint attempt on
  newer Ubuntu hosts (OrbStack questing, e.g.).

- **CRIU's net-lock no longer dies on `iptables-restore: undefined
  symbol: xt_xlate_set_get`.** The previous install used an
  `LD_LIBRARY_PATH=/opt/reel/lib` wrapper script that exported the env
  to every CRIU child process — including host's `iptables-restore`,
  which then picked up our older bundled `libxtables.so.12` and crashed
  on its first symbol lookup. The new install bakes
  `DT_RPATH=/opt/reel/lib` into the binary via `patchelf --force-rpath`;
  RPATH resolves the binary's own deps without propagating to child
  processes' lib resolution.

- **`reel status` no longer reports `✓ Checkpoint` when CRIU is
  actually broken.** Capability detection switched from trusting the
  `CRIU_ENABLED` env var (set by Helm) to reading a status marker file
  written by init-criu. The marker says one of: `source=reel` (reel's
  MPTCP-enabled CRIU installed at host's PATH), `source=host` (our
  install failed but the host already has a CRIU), or `source=none`
  (no CRIU available). False-positive ✓ Checkpoint reports are gone.

### Changed

- **`reel status` CRIU line gains a source qualifier.** Examples:
  - `CRIU: v4.2 (reel, MPTCP-enabled)` — happy path
  - `CRIU: vX.Y (host, no MPTCP — Go 1.24+ workloads may fail)` — reel's
    install failed but the host has a CRIU; checkpoints will work for
    most workloads but Go 1.24+ services may fail at the MPTCP socket
    layer (the patch isn't in mainline CRIU yet).
  - Line omitted when init-criu was disabled in the chart; the Features
    section instead shows `✓ Checkpoint (init-criu disabled in chart;
    CRIU presence not verified, checkpoint is best-effort)`.

- **`initCriu.enabled=false` in Helm no longer artificially blocks
  checkpoint.** The agent used to require `CRIU_ENABLED=true` and refuse
  the operation otherwise — even if the host actually had a CRIU
  installed via some other mechanism. Now `initCriu.enabled=false` means
  "skip init-criu's image pull and install"; the agent reports the
  state as unverified but allows checkpoint attempts. If the host has
  a CRIU, kubelet → runc → criu finds it via PATH and the checkpoint
  succeeds; if not, the runtime error surfaces cleanly. Lets users opt
  out of the init-criu apparatus (e.g., to avoid pulling an extra image)
  without losing the checkpoint feature.

- **init-criu's `install.sh` is probe-first.** It verifies the bundled
  binary works before writing anything to the host. If verification
  fails, the host is left untouched (no half-broken `/usr/local/sbin/criu`
  symlink pointing at a segfaulting binary), and the marker reports
  `source=none` (or `source=host` if a pre-existing host CRIU is
  detected). Replaces the previous `chroot $DEST /usr/local/sbin/criu
  --version || echo "Version check skipped"` silent-failure pattern.

### Added

- **`/uninstall.sh` bundled in the init-criu image.** Removes the
  symlink + `/opt/reel/` files this image's `install.sh` writes.
  Idempotent. Run manually via `kubectl run` to clean a node:

  ```bash
  kubectl run reel-cleanup --rm -it --restart=Never \
    --image=getreel/init-criu:v1.7.0 \
    --overrides='{"spec":{"containers":[{"name":"reel-cleanup","image":"getreel/init-criu:v1.7.0","securityContext":{"privileged":true},"command":["/uninstall.sh","/host"],"volumeMounts":[{"name":"host","mountPath":"/host"}]}],"volumes":[{"name":"host","hostPath":{"path":"/"}}]}}'
  ```

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
