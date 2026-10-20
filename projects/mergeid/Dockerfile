# Node 24 matches the repo's CI (.github/workflows/ci.yml) and docker/Dockerfile.
FROM node:24.20.0-bookworm-slim

# openssl: Prisma's schema engine. postgresql + redis-server: local datastores so
# the offline agent can run the app and the integration tests (prod uses
# Postgres 17 / Redis 8; Debian bookworm ships Postgres 15 / Redis 7).
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git openssl postgresql redis-server \
    && rm -rf /var/lib/apt/lists/*
# Same pnpm as package.json "packageManager".
RUN npm --no-update-notifier --no-fund --global install pnpm@10.27.0

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Install from pnpm-lock.yaml. The Prisma and esbuild postinstalls fetch their
# engines here, while the network is still up.
RUN pnpm install --frozen-lockfile
# `build` runs `prisma generate` (client into src/generated/prisma) and `tsc`.
RUN pnpm build
# Unit tests, as CI runs them (no env, so the Postgres/Redis e2e suites skip).
# A failure must not fail the build.
RUN pnpm test || echo "WARNING: MergeID unit tests reported failures"

# Local-only placeholder settings so src/config/env.ts validates when the agent
# runs the app offline.
# None of these are real credentials; Discord and GitHub are unreachable offline.
ENV NODE_ENV=test \
    DISCORD_TOKEN=oss-scanner-placeholder \
    DISCORD_CLIENT_ID=000000000000000000 \
    GITHUB_CLIENT_ID=oss-scanner-placeholder \
    GITHUB_CLIENT_SECRET=oss-scanner-placeholder \
    OAUTH_REDIRECT_URI=http://localhost:3000/oauth/callback \
    PUBLIC_BASE_URL=http://localhost:3000 \
    TOKEN_ENCRYPTION_KEY=000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f \
    MERGEID_DB_URL=postgresql://mergeid:mergeid@127.0.0.1:5432/mergeid \
    MERGEID_REDIS_URL=redis://127.0.0.1:6379

# Create a local database and apply the migrations, so `prisma migrate deploy`
# works offline later.
# To run the integration tests too: `service postgresql start && redis-server --daemonize yes`, then
# `DATABASE_URL=$MERGEID_DB_URL REDIS_URL=$MERGEID_REDIS_URL pnpm test`.
RUN service postgresql start \
    && su postgres -c "psql -c \"CREATE USER mergeid WITH PASSWORD 'mergeid' CREATEDB;\"" \
    && su postgres -c "createdb -O mergeid mergeid" \
    && DATABASE_URL=$MERGEID_DB_URL pnpm exec prisma migrate deploy \
    && service postgresql stop
