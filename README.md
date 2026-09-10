# rayls-sovereign-workflows

Shared CI for the Rayls Sovereign repositories.

Two workflows:

| | Trigger | Does | Never does |
|---|---|---|---|
| `ci-container.yml` | your PR | builds every architecture, then Trivy | touches ECR |
| `release-container.yml` | a `v*` tag | builds and publishes | deploys |

## `ci-container.yml`

Proves the images still build, and scans them. No AWS credentials at all.

Call it on `pull_request`, **never `pull_request_target`**. These repos are
public: `pull_request_target` would run a fork's code with the base repo's
secrets. `pull_request` gives a fork a read-only token and no secrets.

```yaml
name: CI
on:
  pull_request:
    branches: [main]

jobs:
  images:
    uses: raylsnetwork/rayls-sovereign-workflows/.github/workflows/ci-container.yml@v1
    with:
      images: >-
        [{"image": "ops-api", "dockerfile": "Dockerfile"}]
```

| Input | Default | |
|---|---|---|
| `images` | — | JSON array of `{image, dockerfile}` |
| `platforms` | `linux/amd64,linux/arm64` | architectures the build must succeed for |
| `scan_platform` | `linux/amd64` | the one loaded and scanned |
| `scan_severity` | `CRITICAL` | what fails the build |

Both architectures are built on purpose. A Dockerfile that hardcodes an
architecture builds fine on amd64 and only fails when someone runs the arm64
image — which is exactly the bug this caught in `rayls-sovereign-contracts`.

## `release-container.yml`

A maintainer pushes a `v*` tag; the workflow builds every image the caller
lists and publishes it to ECR Public. **It ends at publish** — deployment is
manual and lives outside CI.

A tag is rejected before anything is built unless it:

1. points at a commit on `main`,
2. is newer than the last release in its own `X.Y` line, and
3. contains that release — so a version can never ship without a fix an
   earlier version already carried.

### Using it

```yaml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  release:
    permissions:
      contents: read
      id-token: write
    uses: raylsnetwork/rayls-sovereign-workflows/.github/workflows/release-container.yml@v1
    with:
      images: >-
        [{"image": "ops-api", "dockerfile": "Dockerfile"}]
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

| Input | Default | |
|---|---|---|
| `images` | — | JSON array of `{image, dockerfile}` |
| `namespace` | `rayls-sovereign` | registry namespace |
| `registry_alias` | `w0k9o1t3` | ECR Public alias |
| `platforms` | `linux/amd64,linux/arm64` | |

`AWS_ROLE_ARN` must be an OIDC role allowed to push to ECR Public.

### Notes

- Pin callers to a tag (`@v1`), not `@main` — `@main` changes every caller at once.
- The ECR Public repository is created on first release; nothing to request up front.
- ECR Public has no immutable-tag setting, so the workflow refuses to push over an
  existing tag. If one image of several fails, re-running the tag finishes the
  release instead of burning the version.
- Dockerfiles should cross-compile from `$BUILDPLATFORM` rather than be emulated.
  Go with `CGO_ENABLED=0` does this cleanly and makes arm64 nearly free.
