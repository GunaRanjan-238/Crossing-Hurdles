# Deno project (kvdex). Use the official Debian-based Deno image so the sandbox
# has full GNU coreutils, tar and gzip available (required by Daytona to archive
# the verifier directory). The Alpine/busybox base was the cause of the
# "sandbox images must provide tar and gzip" Daytona build failure.
FROM denoland/deno:debian-2.4.0

# Harbor/Terminus runtime tools + GNU tar/gzip. Installed early so the layer is
# cached across rebuilds. Verify both harness tools are on PATH.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        tmux \
        asciinema \
        tar \
        gzip \
        git \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && command -v tmux && command -v asciinema && command -v tar && command -v gzip

WORKDIR /app

# Cache Deno dependencies at build time so tests run fully offline.
ENV DENO_DIR=/deno-dir

# Copy manifest/lock first for better layer caching of the dependency download.
COPY repo/deno.json repo/deno.lock ./

# Bring in the full source so cache resolution sees every module/import.
COPY repo/ .

# Warm the module cache for the app entrypoint, all test modules and benchmarks.
# This downloads every jsr:/npm:/https: dependency during `docker build` so the
# offline test run needs no network.
RUN deno cache mod.ts \
    && find tests benchmarks src -name '*.ts' -print0 | xargs -0 deno cache

# Default: run the test suite exactly as `deno task test` does.
CMD ["deno", "test", "--allow-write", "--allow-read", "--allow-ffi", "--allow-sys", "--unstable-kv", "--trace-leaks"]
RUN mkdir -p /installed-agent
RUN command -v patch >/dev/null 2>&1 || ( (apt-get update && apt-get install -y --no-install-recommends patch && rm -rf /var/lib/apt/lists/*) 2>/dev/null || apk add --no-cache patch 2>/dev/null || yum install -y patch 2>/dev/null || true )
