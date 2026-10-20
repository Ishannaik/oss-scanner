# Threat model

## What this project does and where untrusted input enters
- CloakBin is a zero-knowledge encrypted pastebin built with SvelteKit. The browser encrypts each paste with AES-256-GCM (`src/lib/crypto.ts`). For a random paste the key lives only in the URL `#fragment`. For a password paste the browser derives the key with PBKDF2 (100,000 iterations) from the password and a salt that the server stores. The server only stores and returns ciphertext.
- Full design: `docs/SECURITY-MODEL.md` in the repository.
- Untrusted input enters through:
  - `POST /api/paste` (`src/routes/api/paste/+server.ts`): ciphertext, expiry, salt, burn flag, language. Size is capped by `MAX_PASTE_BYTES` (`src/lib/server/maxPasteBytes.ts`). Rate limiting is in `src/lib/server/rateLimit.ts`.
  - `GET /api/paste/[id]` and `POST /api/paste/[id]/burn`: the paste id.
  - The view pages under `src/routes/p` and `src/routes/r`: ciphertext and the URL fragment, decrypted and rendered in the browser (CodeMirror, highlight.js, Markdown).
  - The admin panel under `src/routes/admin`: login form and an HMAC-signed session cookie (`src/lib/server/admin-auth.ts`, checked in `src/hooks.server.ts`).
  - The CLI in `cli/`: encrypts locally, uploads, and decrypts server responses.

## Security goals
- The server, its logs, and third parties must never see plaintext, the key, the URL fragment, or the paste password.
- Decrypted content must not run script in the viewer's page (no XSS through paste content, language name or other metadata).
- A burn-after-read paste must be returned to at most one reader (`src/lib/server/burnAfterRead.ts`, adapter `burnPaste`).
- Only a holder of a valid admin session may reach admin routes. Forged, tampered or expired tokens must fail.

## Components that matter most / least
- Most: `src/lib/crypto.ts`, the paste API routes, `hooks.server.ts`, `admin-auth.ts`, burn-after-read, and the paste view pages.
- Less: the memory and MongoDB storage adapters. Other adapters under `src/lib/db/adapters/` (Redis, Postgres, Turso, Cloudflare KV) are unfinished and out of scope unless `src/lib/db/index.ts` can select them.
- Out of scope: `mockups/`, `static/` assets, and marketing pages.

## How to exercise it
- `pnpm run test` runs the vitest unit tests (`src/**/*.test.ts`).
- `cd cli && npm test` runs the CLI round-trip tests.
- `pnpm run preview` serves the built app with `DB_TYPE=memory` (already set in the image).

## How you rate severity
- Critical: any path where plaintext, the key or the password reaches the server or a third party; stored XSS on the paste view page; admin auth bypass.
- High: a crypto flaw that weakens confidentiality or integrity (IV reuse, missing authentication, weak derivation); burn-after-read returning a paste twice.
- Medium: rate-limit or size-cap bypass leading to DoS; reflected XSS needing user interaction.
- Low: information leaks that reveal only metadata (expiry, language, size).

## Anything to leave alone
- A full random-paste URL decrypts the paste by design. Do not report that the fragment is in the link.
- Weak user-chosen passwords and compromised end-user browsers are out of scope.
