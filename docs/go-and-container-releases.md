# Go And Container Releases

This guide is both human documentation and a normative integration contract for
automation agents. Requirements use **MUST**, **SHOULD**, and **MAY** in their
usual RFC-style meaning.

## What Gets Released

A Go service commonly has three distinct forms:

1. The Go module is released by pushing a semantic-version tag such as
   `v1.2.3`. Consumers can use `go get` or `go install` against that tag.
2. Executable archives are built by GoReleaser for each supported operating
   system and architecture, then attached to the GitHub release.
3. A multi-platform container image is published to an OCI registry such as
   GHCR.

The Go module tag does not require a separate package registry. The reusable Go
workflow publishes executable distributions; the tag itself makes the Go
module version available.

## Attestation Model

Attestations bind an artifact name and digest to provenance. Artifacts with
different digests MUST be represented as different subjects.

<!-- markdownlint-disable MD013 -->

| Distribution | Attestation subject | Storage |
| --- | --- | --- |
| Go archives | Every filename and digest in the GoReleaser checksum file | GitHub artifact attestation API |
| OCI image | Fully qualified image name and multi-platform index digest | GitHub API and OCI registry |
| OCI platform manifests | Referenced by the attested image index | OCI registry |

<!-- markdownlint-enable MD013 -->

`go-release.yml` passes the GoReleaser checksum manifest to `actions/attest`.
GitHub creates one signed in-toto/SLSA statement containing every listed
archive as a subject. A downloaded archive can therefore be verified directly.

`docker-build-push.yml` passes the digest returned by
`docker/build-push-action` to `actions/attest`. For a multi-platform build, this
is the OCI index digest that references the platform manifests. Buildx also
publishes its native provenance and SBOM because `provenance` and `sbom` are
enabled.

Signing and attestation are complementary:

- Cosign signing proves that a configured workflow identity approved an
  artifact or checksum file.
- SLSA provenance records where and how the subject was built.
- A checksum file maps downloadable archive names to their immutable digests.

## Go Workflow Contract

Workflow path:

```text
.github/workflows/go-release.yml
```

### Required Caller Files

The repository MUST contain:

- A valid Go version file, `go.mod` by default.
- A GoReleaser v2 configuration, `.goreleaser.yaml` by default.
- A GoReleaser checksum configuration that emits the file named by
  `checksum-file`.

The default checksum contract is:

```yaml
checksum:
  name_template: checksums.txt
  algorithm: sha256
```

This creates `dist/checksums.txt`. If a project uses another name, it MUST pass
the corresponding repository-relative path through `checksum-file`.

### Inputs

<!-- markdownlint-disable MD013 -->

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `tag_name` | Yes | None | Tag checked out and released, such as `v1.2.3` |
| `go-version-file` | No | `go.mod` | Repository-relative Go version file |
| `goreleaser-config` | No | `.goreleaser.yaml` | Repository-relative GoReleaser v2 configuration |
| `goreleaser-version` | No | `v2.15.2` | Exact GoReleaser version |
| `install-cosign` | No | `true` | Installs Cosign for GoReleaser signing pipes |
| `attest` | No | `true` | Creates provenance for checksummed artifacts |
| `checksum-file` | No | `dist/checksums.txt` | GoReleaser checksum manifest used as attestation subjects |
| `environment` | No | `release` | GitHub environment applied to the publishing job |

### Outputs

| Output | Meaning |
| --- | --- |
| `version` | Version without the leading `v` or component prefix |
| `artifacts` | GoReleaser artifact metadata JSON |
| `attestation-url` | GitHub URL for the generated provenance statement; empty when attestation is disabled |

<!-- markdownlint-enable MD013 -->

### Required Permissions

```yaml
permissions:
  contents: write
  id-token: write
  attestations: write
  artifact-metadata: write
```

`contents: write` uploads assets to the GitHub release. `id-token: write` is
used by Cosign and GitHub attestation signing. `attestations: write` persists
provenance, and `artifact-metadata: write` records the attested subjects.

### GoReleaser Example

This configuration suits a Go service with a command at `./cmd/my-service`:

```yaml
version: 2
project_name: my-service

before:
  hooks:
    - go mod verify

builds:
  - id: my-service
    main: ./cmd/my-service
    binary: my-service
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
    goarch:
      - amd64
      - arm64
      - arm
    goarm:
      - "7"
    ignore:
      - goos: darwin
        goarch: arm
    flags:
      - -trimpath
    ldflags:
      - -s -w

archives:
  - formats: [tar.gz]
    name_template: >-
      {{ .ProjectName }}_
      {{- title .Os }}_
      {{- if eq .Arch "amd64" }}x86_64
      {{- else if eq .Arch "386" }}i386
      {{- else }}{{ .Arch }}{{ end }}
      {{- if .Arm }}v{{ .Arm }}{{ end }}

checksum:
  name_template: checksums.txt
  algorithm: sha256

signs:
  - cmd: cosign
    signature: "${artifact}.sigstore.json"
    args:
      - sign-blob
      - "--bundle=${signature}"
      - "${artifact}"
      - --yes
    artifacts: checksum
    output: true

release:
  mode: keep-existing

changelog:
  use: github-native
```

Use `release.mode: keep-existing` when Release Please creates the GitHub release
before GoReleaser runs. This preserves the Release Please notes while
GoReleaser uploads assets.

Set `install-cosign: false` only when the GoReleaser configuration contains no
Cosign signing pipe. Set `attest: false` only when GitHub artifact attestations
are intentionally unavailable, for example under an unsupported private
repository plan.

## Docker Workflow Contract

For the default `linux/amd64,linux/arm64` build, the caller MUST grant:

```yaml
permissions:
  contents: read
  packages: write
  id-token: write
  attestations: write
  artifact-metadata: write
```

The relevant outputs are:

| Output | Meaning |
| --- | --- |
| `image` | Fully qualified registry and image name |
| `digest` | Immutable OCI index digest |
| `version` | Version derived from the release tag |
| `attestation-url` | GitHub URL for the image provenance statement |

Consumers SHOULD deploy by digest rather than by mutable version or `latest`
tag:

```text
ghcr.io/owner/my-service@sha256:<verified-index-digest>
```

## Combined Release Workflow

The jobs can run in parallel after a CI gate because they publish different
subjects. Callers MUST pin reusable workflows to an immutable release tag or
commit SHA.

```yaml
---
name: Release

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      tag_name:
        description: "Release tag for manual recovery"
        required: true
        type: string

permissions: {}

jobs:
  ci:
    uses: ./.github/workflows/ci.yml
    permissions:
      contents: read

  go:
    needs: ci
    uses: Depthmark/reusable-workflows/.github/workflows/go-release.yml@<commit-sha>
    with:
      tag_name: ${{ github.event.release.tag_name || inputs.tag_name }}
    permissions:
      contents: write
      id-token: write
      attestations: write
      artifact-metadata: write

  docker:
    needs: ci
    uses: Depthmark/reusable-workflows/.github/workflows/docker-build-push.yml@<commit-sha>
    with:
      tag_name: ${{ github.event.release.tag_name || inputs.tag_name }}
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
      artifact-metadata: write
```

Keep `workflow_dispatch` as a recovery path. Re-running a release against an
existing tag MUST be treated as a controlled operation because release assets
and registry tags may already exist.

## Depthmark/github-sts Adoption

`Depthmark/github-sts` already has the first half of the required release
chain:

1. Its Release Please workflow requests a short-lived GitHub App installation
   token through `Depthmark/github-sts-action`.
2. The request is scoped to `Depthmark/github-sts`, identity `release`, and app
   `depthmark-release-bot`.
3. Release Please uses that token to create the release PR, tag, and GitHub
   release.

This matters because GitHub suppresses downstream workflow events caused by a
repository `GITHUB_TOKEN`. A release created with the STS-issued GitHub App
token can emit `release.published`, allowing the combined release workflow to
start automatically.

For `github-sts`, implementation consists of:

1. Add `.goreleaser.yaml` using `project_name: github-sts`,
   `main: ./cmd/github-sts`, and `binary: github-sts`.
2. Name the checksum file `checksums.txt` so it satisfies the default reusable
   workflow contract.
3. Add a `go` job beside the existing `docker` job in
   `.github/workflows/release.yml`.
4. Add `release: types: [published]` while retaining the current manual trigger.
5. Add `artifact-metadata: write` to both publishing jobs and
   `attestations: write` to the Go job.
6. Pin the new reusable workflow release by commit SHA.
7. Pin `Depthmark/github-sts-action` by commit SHA rather than `main`.

The STS-issued token SHOULD remain limited to release orchestration. It does not
need to cross job boundaries:

- GoReleaser uses the Go job's scoped `GITHUB_TOKEN` to upload assets.
- Docker uses the Docker job's scoped `GITHUB_TOKEN` to push to GHCR.
- Cosign and `actions/attest` independently request short-lived Sigstore
  credentials through GitHub OIDC.

This avoids propagating a GitHub App token between jobs and preserves least
privilege.

### Existing Release Considerations

Release Please creates the GitHub release before GoReleaser uploads binaries.
The caller's GoReleaser configuration therefore MUST use
`release.mode: keep-existing`.

If GitHub immutable releases are enabled, assets cannot be appended to an
already published release. In that case the project MUST instead create a draft
release, upload and attest all assets, then publish the release as the final
step. Do not enable a `release.published` trigger for that draft-based topology,
because publication occurs after artifact generation.

The `github-sts` release pipeline depends on the deployed STS service to create
the release. Keep the manual release trigger as an operational recovery path
for an existing tag if the STS service is unavailable.

## Verification

Download and verify one Go archive:

```bash
gh release download v1.2.3 \
  --repo Depthmark/github-sts \
  --pattern "github-sts_Linux_x86_64.tar.gz"

gh attestation verify \
  github-sts_Linux_x86_64.tar.gz \
  --repo Depthmark/github-sts
```

Validate all downloaded files against the signed manifest:

```bash
sha256sum --check checksums.txt
```

Verify the container provenance:

```bash
gh attestation verify \
  oci://ghcr.io/depthmark/github-sts:1.2.3 \
  --repo Depthmark/github-sts
```

Verify the keyless container signature against the expected repository workflow
identity:

```bash
cosign verify \
  --certificate-identity-regexp \
  '^https://github.com/Depthmark/github-sts/.github/workflows/' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  ghcr.io/depthmark/github-sts@sha256:<index-digest>
```

After resolving the index digest, deploy the immutable reference:

```text
ghcr.io/depthmark/github-sts@sha256:<verified-index-digest>
```

## Failure Guide

<!-- markdownlint-disable MD013 -->

| Failure | Likely cause | Resolution |
| --- | --- | --- |
| `Checksum file not found or empty` | GoReleaser filename differs from `checksum-file` | Set `checksum.name_template: checksums.txt` or pass the actual path |
| Attestation permission denied | Caller omitted a required permission | Grant `id-token`, `attestations`, and `artifact-metadata` write access |
| GoReleaser cannot create the release | Release Please already created it with incompatible settings | Set `release.mode: keep-existing`; check immutable release settings |
| Release assets exist on a manual rerun | The same tag was already published | Inspect existing assets before enabling controlled replacement |
| `release.published` does not start publication | Release was created with `GITHUB_TOKEN` | Use the STS-issued GitHub App token in Release Please |
| Container verification resolves a different digest | A mutable tag moved | Verify and deploy the digest emitted by the Docker workflow |

<!-- markdownlint-enable MD013 -->

## Validation Checklist

Before merging a caller integration:

1. Run `goreleaser check` against `.goreleaser.yaml`.
2. Run `goreleaser release --snapshot --clean` and inspect `dist/`.
3. Confirm every intended archive is present in `dist/checksums.txt`.
4. Run `actionlint`, Zizmor, and Poutine against workflow changes.
5. Test with a prerelease tag before publishing a stable version.
6. Download and verify at least one archive per operating system.
7. Verify the OCI image and inspect its platform list.
8. Confirm Release Please notes remain intact after GoReleaser uploads assets.
