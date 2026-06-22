# Reel CLI

Kubernetes continuous compliance. Captures container state — SBOMs, cryptographic inventories, vulnerability scans, and malware detection — before pods disappear.

This repository distributes the **`reel` CLI binary** (Linux, macOS, Intel + Apple Silicon) and tracks its [changelog](./CHANGELOG.md). Related artifacts live in dedicated repos:

| Artifact | Repository |
|---|---|
| **CLI binary** + changelog (this repo) | [`getreeldev/reel-cli`](https://github.com/getreeldev/reel-cli) |
| **GitHub Action** | [`getreeldev/reel-action`](https://github.com/getreeldev/reel-action) |
| **Helm chart** | [`getreeldev/helm`](https://github.com/getreeldev/helm) |
| **Homebrew formula** | [`getreeldev/homebrew-tap`](https://github.com/getreeldev/homebrew-tap) |
| **Docker images** | `getreel/agent`, `getreel/init-criu`, `getreel/init-trivy` on Docker Hub |

## Install

### Linux — direct download

```bash
ARCH=$(uname -m); case "$ARCH" in aarch64|arm64) ARCH=arm64 ;; *) ARCH=amd64 ;; esac
curl -sL https://github.com/getreeldev/reel-cli/releases/latest/download/reel_linux_${ARCH}.tar.gz \
  | tar xz && sudo mv reel /usr/local/bin/
```

Each release publishes `reel_linux_amd64`, `reel_linux_arm64`, `reel_darwin_amd64`, and `reel_darwin_arm64` tarballs.

### macOS — Homebrew

```bash
brew install getreeldev/tap/reel
```

Verify:

```bash
reel version
reel status
```

## Quick Start

```bash
# Generate an SBOM
reel export sbom --image nginx:latest -o sbom.json

# Vulnerability scan (SARIF output, ready for GitHub Code Scanning)
reel export sarif --image nginx:latest -o results.sarif

# Annotate scan output with vendor VEX statements (new in v1.5.0)
reel export sbom --image redhat/ubi9-minimal --scanners vuln,vex

# Cryptographic bill of materials
reel export cbom --image nginx:latest -o cbom.json

# Malware detection
reel export malware --image nginx:latest -o malware.json
```

## Documentation

- Full docs: [getreel.dev/docs](https://getreel.dev/docs)
- GitHub Action usage: [getreel.dev/docs/github-action](https://getreel.dev/docs/github-action)
- VEX integration: [vex.getreel.dev](https://vex.getreel.dev)

## License

Proprietary. See [LICENSE](LICENSE) for terms.
