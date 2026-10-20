# Warp has two stacks: a pnpm workspace (web/ React app + server/ Cloudflare Worker)
# and a Rust CLI (cli/, Cargo.lock). Start from a pinned Rust image and copy in a
# pinned Node 22 (.nvmrc says 22) from the official Node image.
FROM node:22.23.2-bookworm-slim AS node

FROM rust:1.98.1-bookworm
COPY --from=node /usr/local/ /usr/local/

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git pkg-config \
    && rm -rf /var/lib/apt/lists/*
# Same pnpm as package.json "packageManager".
RUN npm --no-update-notifier --no-fund --global install pnpm@10.27.0

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

ENV WRANGLER_SEND_METRICS=false \
    CI=1

# JS workspace: install from pnpm-lock.yaml. The workerd and esbuild postinstalls
# fetch their native binaries here, while the network is still up.
RUN pnpm install --frozen-lockfile
RUN pnpm --filter @warp/web build

# Rust CLI: fetch every crate from Cargo.lock, then build the binary and the unit
# test binaries so `cargo test --lib --offline` works without a network.
RUN cd cli && cargo fetch --locked \
    && cargo build --locked \
    && cargo test --locked --lib --no-run
ENV CARGO_NET_OFFLINE=true

# Cheap tests. None of them may fail the build.
#  - engine checks: dependency-free harnesses in web/src/lib/warp/*.check.mjs
#  - signaling server: boots `wrangler dev` (local workerd) and drives WebSockets
#  - CLI unit tests (the live_signaling test needs the Internet and is skipped)
# The Playwright e2e suite is not set up: it downloads browsers and needs a relay.
RUN pnpm --filter @warp/web check:engine || echo "WARNING: warp engine checks reported failures"
RUN pnpm --filter @warp/server test || echo "WARNING: warp signaling server test reported failures"
RUN cd cli && (cargo test --locked --lib || echo "WARNING: warp CLI unit tests reported failures")
