# Threat model

## What this project does and where untrusted input enters
- MergeID is a self-hosted Discord bot (TypeScript, discord.js, Fastify, Prisma on PostgreSQL, BullMQ on Redis). It links a Discord account to a GitHub account through GitHub OAuth, then grants or revokes Discord roles from rules such as "member of GitHub org X". A background worker re-verifies links on a schedule.
- Full design in the repository: `docs/security-model.md` and `docs/architecture.md`.
- Untrusted input enters through:
  - HTTP: `GET /oauth/callback` (`src/api/routes/oauth.ts`, `src/api/server.ts`) with `code` and `state` from the browser. HTML responses are built in `src/lib/html.ts`.
  - Discord interactions: slash commands and their options from any guild member (`src/discord/events/interaction-create.ts`, `src/discord/commands/*`). Admin commands (`/mergeid ...`) need `ManageGuild`.
  - GitHub API responses about the linked user and their orgs (`src/github/*`).
  - Environment configuration (`src/config/env.ts`). This is trusted operator input.

## Security goals
- GitHub access tokens are stored only encrypted (AES-256-GCM, versioned key, `src/crypto/token.ts`) and never appear in logs, URLs or error messages.
- OAuth `state` is single-use, bound to the initiating Discord user, and expires (`src/oauth/state.ts`, PKCE in `src/oauth/pkce.ts`). An attacker must not bind their GitHub account to a victim's Discord account, or the reverse.
- Authorization uses the numeric GitHub user id, never the login. One GitHub account links to one Discord account.
- The bot only grants roles from the guild's allowlist, never a role above its own or a protected role (`src/discord/roles.ts`, `src/discord/rule-roles.ts`, `src/verification/engine.ts`).
- Non-admin members cannot change rules or settings, and cannot read other members' links.

## Components that matter most / least
- Most: `src/api/`, `src/oauth/`, `src/crypto/`, `src/services/links.ts`, `src/verification/engine.ts`, `src/discord/roles.ts`, `src/discord/rule-roles.ts`, `src/discord/commands/`.
- Less: `src/sync/rate-budget.ts` and the scheduler (availability only), `src/lib/logger.ts`.
- Out of scope: `docs/`, `scripts/`, `docker/compose*.yml`.

## How to exercise it
- `pnpm test` runs the vitest suite in `test/`. Without `DATABASE_URL` and `REDIS_URL` the e2e suites skip.
- The image has a local Postgres with the migrations applied and a Redis server. Start them with `service postgresql start && redis-server --daemonize yes`, then run `DATABASE_URL=$MERGEID_DB_URL REDIS_URL=$MERGEID_REDIS_URL pnpm test`.
- Discord and GitHub are not reachable offline. The tests mock them.

## How you rate severity
- Critical: reading or decrypting stored GitHub tokens; OAuth CSRF or link hijack; a non-admin granting themselves a role or changing rules.
- High: token or secret leakage into logs or responses; XSS on the OAuth callback page; granting a role outside the allowlist or above the bot.
- Medium: DoS of the callback endpoint or the sync worker; wrong revocation of roles.
- Low: information leaks about which accounts are linked, without tokens.

## Anything to leave alone
- Misconfigured self-hosted deployments (exposed database or Redis ports, weak env secrets, no TLS) are out of scope, as `SECURITY.md` says.
- The placeholder values in the image's environment are fake on purpose.
