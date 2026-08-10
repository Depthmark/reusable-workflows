# Depthmark Reusable Workflows

Security-hardened reusable GitHub Actions workflows for Depthmark projects.
Callers should pin workflows to a release tag or commit SHA rather than `main`.

## Release Workflows

<!-- markdownlint-disable MD013 -->

| Workflow | Publishes | Signing and provenance |
| --- | --- | --- |
| `go-release.yml` | Architecture-specific Go archives and checksums on a GitHub release | Optional Cosign checksum signature and GitHub SLSA provenance for every checksummed artifact |
| `docker-build-push.yml` | Multi-platform OCI image | Cosign signature, BuildKit SBOM/provenance, and GitHub SLSA provenance for the image index digest |
| `helm-package-push.yml` | Helm chart in an OCI registry and optionally a GitHub release | Cosign signature and GitHub SLSA provenance |

<!-- markdownlint-enable MD013 -->

The Go and Docker workflows are designed to run in parallel for the same
immutable release tag. They create separate attestations because a release
archive and an OCI image have different artifact digests.

See [Go and container releases](docs/go-and-container-releases.md) for:

- The normative caller contract and required permissions.
- A complete `.goreleaser.yaml` example.
- Combined Go and Docker release workflow examples.
- `Depthmark/github-sts` integration guidance.
- Signature and attestation verification commands.
- Failure modes and troubleshooting.

## Quick Example

```yaml
jobs:
  go:
    uses: Depthmark/reusable-workflows/.github/workflows/go-release.yml@<commit-sha>
    with:
      tag_name: v1.2.3
    permissions:
      contents: write
      id-token: write
      attestations: write
      artifact-metadata: write

  docker:
    uses: Depthmark/reusable-workflows/.github/workflows/docker-build-push.yml@<commit-sha>
    with:
      tag_name: v1.2.3
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
      artifact-metadata: write
```

The caller must provide `.goreleaser.yaml` and make its checksum filename match
the Go workflow's `checksum-file` input. The default contract is
`dist/checksums.txt`.
