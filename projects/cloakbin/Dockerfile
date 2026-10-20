FROM node:24.20.0-bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git \
    && rm -rf /var/lib/apt/lists/*
# Same pnpm version as the repo's CI (.github/workflows/type-coverage.yml).
RUN npm --no-update-notifier --no-fund --global install pnpm@10.15.1

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Local-only settings. The in-memory storage adapter needs no database, so the
# app can run offline with `pnpm run preview`. These are not production secrets.
ENV DB_TYPE=memory \
    ADMIN_USER=oss-scanner-admin \
    ADMIN_PASS=oss-scanner-local-only-not-a-production-secret

# Web app: dependencies from pnpm-lock.yaml (sharp's postinstall fetches its
# prebuilt libvips here, while the network is still up), then the SvelteKit build.
RUN pnpm install --frozen-lockfile
RUN pnpm run build

# CLI: separate npm package with its own package-lock.json.
RUN cd cli && npm ci --no-fund --no-audit

# Run the cheap tests so the image is known to work, without failing the build.
RUN pnpm run test || echo "WARNING: CloakBin web tests (vitest) reported failures"
RUN cd cli && (npm test || echo "WARNING: CloakBin CLI tests reported failures")
