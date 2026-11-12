# syntax=docker/dockerfile:1
#
# Build on an arm64 host (mtl0) with the project root as the build context:
#
#   cd /path/to/slacking-off
#   docker build -f /path/to/k3s-homelab/apps/slacking-off/Dockerfile \
#     -t slacking-off:arm64 .
#
# The context must also contain `loo-linux-x64`, the x86_64 CLI built from the
# same commit (the node itself is arm64 and cannot run an x86_64 cargo build
# natively). The image serves both Linux CLI binaries from /app/downloads.
#
# The web bundle and the Rust binaries are built natively; this Dockerfile is
# intentionally not a cross-build.

FROM node:24-bookworm-slim AS web
WORKDIR /src/web
COPY web/package.json web/package-lock.json ./
RUN npm ci
COPY web/ ./
# The SPA imports `docs/art/sprites.svg?raw`, which sits outside the Vite root.
COPY docs/art/sprites.svg /src/docs/art/sprites.svg
RUN npm run build

FROM rust:1-bookworm AS build
RUN apt-get update \
    && apt-get install -y --no-install-recommends pkg-config \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /src
COPY Cargo.toml Cargo.lock ./
COPY crates ./crates
COPY --from=web /src/web/dist ./web/dist
RUN cargo build --release -p loo-server -p loo

FROM debian:bookworm-slim
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd --gid 10001 loo \
    && useradd --uid 10001 --gid 10001 --no-create-home --home-dir /nonexistent --shell /usr/sbin/nologin loo \
    && mkdir -p /app/maps /app/downloads /data \
    && chown 10001:10001 /data

COPY --from=build /src/target/release/loo-server /usr/local/bin/loo-server
COPY --from=build /src/target/release/loo /app/downloads/loo-linux-arm64
# Prebuilt on the amd64 workstation from the same commit.
COPY loo-linux-x64 /app/downloads/loo-linux-x64
COPY maps/default.txt /app/maps/default.txt
RUN chmod 755 /app/downloads/loo-linux-arm64 /app/downloads/loo-linux-x64

ENV TMPDIR=/tmp \
    MALLOC_ARENA_MAX=2
VOLUME ["/data"]
EXPOSE 18080
STOPSIGNAL SIGTERM

USER 10001:10001
ENTRYPOINT ["/usr/local/bin/loo-server"]
CMD ["--official", "--addr", "0.0.0.0:18080", "--map", "/app/maps/default.txt", "--data-dir", "/data", "--downloads-dir", "/app/downloads"]
