---
layout: single
title: "A Single 100 MB JAR Broke My Docker Build — Here's How I Fixed It"
categories: [dev]
tags: [docker, github-actions, open-source]
---

**TL;DR:** Store proprietary binaries as GitHub release assets. Fetch them at Docker build time via a `LIB_URL` build arg. Never commit blobs to git again.

---

I run a few TP-Link Omada devices at home: a router, a managed switch, and a couple of APs. The Omada controller is what ties it all together, and I run it on a Raspberry Pi in a Docker Compose stack, with MongoDB split into its own container. Since TP-Link does not publish an official Docker image, I have been maintaining [docker-omada-controller](https://github.com/damoun/docker-omada-controller) since April 2023 to handle that.

Omada ships as a `.deb` that bundles two kinds of dependencies: open-source libraries available on Maven Central, and proprietary JARs that TP-Link does not publish anywhere. For years the repo committed those JARs directly to git. The repo accumulated 173 MB of binary blobs. Then Omada v6.0.0.25 added a single 100.57 MB JAR, and GitHub's 100 MB file size limit broke automated releases entirely. That was the forcing function.

## What "before" looked like

The old Dockerfile was simple: copy the JARs from the `lib/` directory that lived in the repo.

```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
COPY pom.xml .
RUN mvn dependency:copy-dependencies

FROM eclipse-temurin:17-jre-jammy
COPY lib /opt/tplink/EAPController/lib
COPY --from=build target/dependency /opt/tplink/EAPController/lib
```

And `lib/` was just tracked in git:

```
lib/
  anomaly-api-5.15.24.19.jar
  api-gateway-core-5.15.24.19.jar
  omada-web-6.0.0.25-local.jar   # 100.57 MB — the one that broke everything
  ... 249 more JARs
```

This is a well-known anti-pattern, but it is tempting when you do not control the upstream source. No extra infrastructure, no secrets, no external storage. It just works, until the files get large enough to hit platform limits. Beyond the size issue, binary blobs make diffs meaningless, bloat every fresh clone, and create licensing ambiguity for anyone forking the project.

## Separating public from proprietary

Before anything else, the repo needed a reliable way to know which JARs were genuinely proprietary and which ones TP-Link had just bundled for convenience. The `sync-deps.sh` script handles this: it computes the SHA-1 of each JAR and queries Maven Central's search API. Any JAR found there gets added to `pom.xml` instead. Anything that comes back with no match is kept as proprietary.

The output is `lib-keep.txt`, a plain list of JAR paths that must stay in the image but cannot be sourced from Maven Central. That file drives the packaging step. In practice, every open-source JAR in the bundle matched — SHA-1 lookups on Maven Central are exact (no false positives), and the index is complete enough that manual cleanup was not needed.

## The approach: release assets as a versioned binary store

The fix splits the problem in two:

1. **At release time**: package the JARs listed in `lib-keep.txt` into a `lib.tar.gz` archive and upload it as a GitHub release asset.
2. **At build time**: the Dockerfile fetches and extracts that archive via a `LIB_URL` build argument.

GitHub release assets support files up to 2 GB when uploaded via the API, so size is no longer a concern. They are also versioned: every release gets its own set of assets, so the exact JARs that went into any given image tag are permanently archived and traceable.

The release workflow now does this instead of committing JARs:

```bash
mkdir -p lib-staging
while IFS= read -r jar_path; do
  cp "$jar_path" lib-staging/
done < lib-keep.txt
tar -czf lib.tar.gz -C lib-staging .
```

Then `gh release create` uploads `lib.tar.gz` alongside the changelog, and the URL is passed forward to the Docker build as a workflow output.

## A three-stage Dockerfile

The image now uses three stages: `download`, `build`, and the final runtime.

```dockerfile
FROM debian:bookworm-slim AS download

ARG LIB_URL
RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
RUN mkdir -p /lib-jars && \
    curl -fsSL "${LIB_URL}" | tar -xz -C /lib-jars

FROM maven:3.9-eclipse-temurin-17 AS build
COPY pom.xml .
RUN mvn dependency:copy-dependencies

FROM eclipse-temurin:17-jre-jammy
COPY --from=download /lib-jars /opt/tplink/EAPController/lib
COPY --from=build target/dependency /opt/tplink/EAPController/lib
```

Keeping the two JAR sources in separate stages matters: you can patch CVEs in public Maven dependencies by bumping `pom.xml` without touching the proprietary JARs, and vice versa. Each layer is also cached independently, so a rebuild that does not change the JAR version reuses the `download` layer without re-fetching.

The `LIB_URL` argument is required and has no default. If you try to build locally without passing it, `curl` fails loudly. That is intentional.

## The git history rewrite

Removing the JARs from tracking and adding `lib/` to `.gitignore` is not enough. The blobs are still referenced in history and downloaded on every fresh clone. To actually reclaim the space, you need to rewrite history with `git filter-repo`:

```bash
git filter-repo --path lib/ --invert-paths
```

This rewrites every commit that touched `lib/`, changing commit SHAs from that point forward. For a personal project with a handful of forks it is a one-time disruption, but the clone size savings are permanent.

## The result

- Around 173 MB removed from git history
- No more GitHub file size rejections on release
- Fresh clones no longer download binary blobs
- Future releases are fully automated again
- Both proprietary and open-source JARs are still present in the final image, just fetched from different sources

The full changes are in [PR #192](https://github.com/damoun/docker-omada-controller/pull/192).
