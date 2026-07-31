# Agent guidelines

This file describes the conventions that govern this repository. Read it before making any change.

## Purpose

This repository holds standalone tools shared across [EDOT SDK](https://www.elastic.co/docs/reference/opentelemetry) demo applications. Tools are kept here so consuming repos can reference a released artifact instead of copying source. Each tool is independent: its own directory, its own build, its own release lifecycle.

## Repository layout

```
shared-otel-sdk-demo/
├── README.md                          # root index — one row per tool
├── AGENTS.md                          # this file
├── .gitignore
├── .github/
│   └── workflows/
│       └── release-<tool>.yml         # one workflow per tool
└── <tool>/                            # one directory per tool
    ├── README.md
    ├── Dockerfile                     # if the tool ships a Docker image
    └── ...                            # tool-specific build files and source
```

There is no root-level build system. Each tool directory is entirely self-contained.

## Adding a new tool

1. **Create `<tool>/`** as a self-contained project. It must build and run without depending on any other directory in this repo.

2. **Add `<tool>/README.md`** — see [Tool README requirements](#tool-readme-requirements).

3. **Add `.github/workflows/release-<tool>.yml`** — see [Release workflow conventions](#release-workflow-conventions).

4. **Update the root `README.md`** — add one row to the tools table:
   ```markdown
   | [<tool>](<tool>/) | One-line description |
   ```
   Do not leave the root README out of sync with the actual tool directories.

5. **Verify the build** — run the tool's build locally before committing. For Gradle-based tools: `cd <tool> && ./gradlew build`.

## Release workflow conventions

Each tool has exactly one workflow file: `.github/workflows/release-<tool>.yml`.

**Trigger** — push of a tag matching `<tool>/v[0-9]+.[0-9]+.[0-9]+`:
```yaml
on:
  push:
    tags:
      - '<tool>/v[0-9]+.[0-9]+.[0-9]+'
```

**Required permissions:**
```yaml
permissions:
  contents: write   # create GitHub Releases
  packages: write   # push to ghcr.io
```

**Version extraction** — strip the `<tool>/v` prefix from the tag:
```yaml
- name: Extract version from tag
  run: echo "VERSION=${GITHUB_REF_NAME#<tool>/v}" >> "$GITHUB_ENV"
```

**Every release must publish two artifacts:**

| Artifact | Location | Example |
|----------|----------|---------|
| Docker image | `ghcr.io/elastic/shared-otel-sdk-demo/<tool>:<version>` | `ghcr.io/elastic/shared-otel-sdk-demo/backend:1.0.0` |
| Executable archive + SHA-256 | GitHub Release assets | `backend-1.0.0.jar`, `backend-1.0.0.jar.sha256` |

Also push a moving `:latest` Docker tag alongside the versioned tag for interactive use. Consumers must pin the versioned tag, not `:latest`.

**GitHub Release** — use the tag name as the release tag and attach the archive and its checksum:
```yaml
- name: Create GitHub Release
  env:
    GH_TOKEN: ${{ github.token }}
  run: |
    gh release create "${{ github.ref_name }}" \
      --title "<Tool> v${{ env.VERSION }}" \
      --generate-notes \
      "<tool>/<tool>-${{ env.VERSION }}.jar" \
      "<tool>/<tool>-${{ env.VERSION }}.jar.sha256"
```

**Checksum format** — generate with `sha256sum` on Linux CI so the file contains the bare filename (not a full path). Consumers verify with `shasum -a 256 -c`.

See `.github/workflows/release-backend.yml` as the reference implementation.

## Tool README requirements

Every `<tool>/README.md` must cover, in order:

1. **What it is** — one paragraph.
2. **Endpoints / API** — a table of exposed HTTP endpoints (for HTTP services).
3. **Configuration** — all supported environment variables, with examples.
4. **Running with Docker** — exact `docker run` command using a pinned version tag.
5. **Running without Docker** — download-and-verify steps using the JAR release asset. Include the `shasum` verification step. This is required because GitHub-hosted macOS runners cannot run Docker.
6. **Building locally** — minimum steps to build and run from source.
7. **Releasing** — the exact `git tag` and `git push` commands that trigger the workflow.

## Build conventions

- Each tool manages its own build toolchain. There is no shared root build.
- Gradle-based tools must ship their own `gradlew` and `gradle/wrapper/` so consumers never need a pre-installed Gradle.
- The Gradle `bootJar` (or equivalent archive) task must produce a **fixed filename** (`<tool>.jar`) regardless of the project version, so the Dockerfile and CI steps do not need to glob or version-match the output path. Set this explicitly — for Spring Boot:
  ```kotlin
  tasks.named<org.springframework.boot.gradle.tasks.bundling.BootJar>("bootJar") {
      archiveFileName.set("<tool>.jar")
  }
  ```
- The Dockerfile copies from that fixed path (`build/libs/<tool>.jar`). CI then copies the file to the versioned release name (`<tool>-1.0.0.jar`) before attaching it to the GitHub Release.

## Docker conventions

- Use a JDK base image, not a JRE. The `elastic-otel-runtime-attach` library uses the JVM attach
  API at startup, which is only available in a JDK. Example: `eclipse-temurin:17-jdk`.
- Match the base image Java version to the build toolchain version.
- The Docker build context is the tool's own directory. Do not reference files outside it.

## HTTP service conventions

HTTP services must expose:
- `GET /v1/health` — returns `200 OK` with no body; used by CI to confirm the service started.
- All other endpoints under `/v1/`.

## Keeping documentation in sync

Every code change that affects behaviour visible to consumers must be accompanied by a documentation
update in the same commit. Specifically:

- **Dockerfile base image changed** → update the base image example in the Docker conventions
  section of this file and in `<tool>/README.md` if it shows a `FROM` line.
- **New or removed environment variable** → update the configuration table in `<tool>/README.md`.
- **New or removed HTTP endpoint** → update the endpoints table in `<tool>/README.md`.
- **Build task or output filename changed** → update the build conventions section of this file and
  the "Building locally" section of `<tool>/README.md`.
- **Release workflow logic changed** → update the release workflow conventions section of this file.
- **New tool added or removed** → update the root `README.md` tools table and this file if any
  shared convention changes.

If a change breaks something that was documented as a convention (e.g. a Docker convention),
correct the convention text at the same time as the fix — do not leave the docs describing the
wrong approach.

## What NOT to do

- Do not add a root-level build file that aggregates tools. Tools are independent.
- Do not share source or compiled artifacts between tool directories.
- Do not use a single workflow to release multiple tools. One workflow per tool.
- Do not update the root `README.md` tools table without also having the corresponding `<tool>/` directory and `<tool>/README.md` in the same commit, and vice versa.
- Do not push a release tag without verifying the build succeeds locally first.
- Do not make a code change that affects documented behaviour without updating the relevant docs in
  the same commit.
