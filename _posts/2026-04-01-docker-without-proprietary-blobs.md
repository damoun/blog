---
layout: single
title: "Removing Proprietary JARs from Git: Build-Time Fetching in Docker"
categories: [dev]
tags: [docker, github-actions, open-source]
---

[docker-omada-controller](https://github.com/damoun/docker-omada-controller) is an unofficial Docker image for the TP-Link Omada SDN controller. The controller ships as a `.deb` package that includes a mix of open-source libraries (available on Maven Central) and proprietary JARs that TP-Link does not publish publicly. To build the image, those JARs had to come from somewhere — and for years, the answer was: commit them to git.

That worked until it didn't. Omada v6.0.0.25 added `omada-web-6.0.0.25-local.jar`, a single file weighing 100.57 MB. GitHub's hard file size limit is 100 MB. Automated releases broke. The repo had accumulated ~173 MB of binary blobs. It was time to fix this properly.

## The problem with committing binaries

Committing binaries to git is a well-known anti-pattern, but it's tempting when you don't control the upstream source. The JARs needed to be _somewhere_ to build the image. The simplest answer — just put them in the repo — works until it doesn't scale.

Beyond the size issue, binary blobs in git make history review meaningless, bloat every fresh clone, and create redistribution ambiguity. The blobs had to go.

## The approach: release assets as a build-time source

The solution splits the problem in two:

1. **At release time**: package the proprietary JARs into a `lib.tar.gz` archive and upload it as a GitHub release asset.
2. **At build time**: the Dockerfile downloads and extracts that archive via a `LIB_URL` build argument.

The release workflow now does this instead of committing JARs directly:

```bash
mkdir -p lib-staging
while IFS= read -r jar_path; do
  cp "$jar_path" lib-staging/
done < lib-keep.txt
tar -czf lib.tar.gz -C lib-staging .
```

Then `gh release create` uploads `lib.tar.gz` as a release asset alongside the changelog. The URL gets passed forward to the Docker build workflow.

## A dedicated download stage in the Dockerfile

The key change is a new `download` stage that handles fetching before the build stage runs:

```dockerfile
FROM debian:bookworm-slim AS download

ARG LIB_URL
RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
RUN mkdir -p /lib-jars && \
    curl -fsSL "${LIB_URL}" | tar -xz -C /lib-jars
```

The `build` stage then copies from it:

```dockerfile
COPY --from=download /lib-jars /opt/tplink/EAPController/lib
COPY --from=build target/dependency /opt/tplink/EAPController/lib
```

Maven Central dependencies (managed via `pom.xml`) are resolved separately in the `build` stage, so they can still be upgraded independently when CVEs appear in public libraries.

## The CI wiring

The Docker workflow no longer triggers on `push` to `main`. It only runs as a `workflow_call` from the release workflow, which passes the `lib_url` input:

```yaml
# release.yaml
- name: Get lib asset URL
  id: lib_asset
  run: |
    echo "url=https://github.com/${{ github.repository }}/releases/download/v${NEW_VERSION}/lib.tar.gz" >> "$GITHUB_OUTPUT"
```

```yaml
# docker.yaml (workflow_call input)
lib_url:
  description: "URL to lib.tar.gz release asset containing proprietary JARs"
  required: true
  type: string
```

This makes the build fully reproducible: any given Docker image tag corresponds to a specific release asset URL, and the JARs that went into it are permanently archived there.

## The result

- ~173 MB removed from git history (via `git filter-repo`)
- No more GitHub file size rejections
- The repo is now cloneable without downloading binary blobs
- Future releases are fully automated again
- Proprietary and open-source JARs are still both present in the final image, just fetched from different sources

If you're in a similar situation — packaging software that ships proprietary binaries you can't redistribute in source form — this pattern works well. Use GitHub release assets as a controlled, versioned binary store, and fetch at build time rather than commit time.

The full changes are in [PR #192](https://github.com/damoun/docker-omada-controller/pull/192).
