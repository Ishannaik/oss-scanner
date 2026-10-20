# Threat model

## What this project does and where untrusted input enters
- Warp sends files directly between devices over WebRTC. It has three parts:
  - `server/`: a Cloudflare Worker with Durable Objects (`server/src/index.js`). It is only a signaling relay. It mints 6-character room codes, groups peers into rooms (max 8), offers nearby-device discovery by public IP, and relays SDP/ICE blobs.
  - `web/`: a React app. The transfer engine is in `web/src/lib/warp/` (signaling client, peer, transfer, receive controller and guards, resume, piece manifest and SHA-256 hashing, OPFS/IndexedDB staging, zip download).
  - `cli/`: a Rust CLI (`webrtc` crate) that speaks the same protocol (`cli/src/protocol.rs`, `signaling.rs`, `peer.rs`).
- Design docs in the repository: `docs/THREAT-MODEL.md` and `docs/ARCHITECTURE.md`.
- Untrusted input enters through:
  - Any WebSocket client of the signaling server: room codes, aliases (`shared/codewords.js`), peer ids, device names and signal frames. Frames are capped at 64 KiB (`MAX_SIGNAL_BYTES`).
  - The remote peer over the data channel: file metadata (names, sizes, MIME types), piece manifests, chunks, resume tokens and control messages. The receiver must treat all of it as hostile.

## Security goals
- The signaling server never sees file bytes. File data travels only over the peers' DTLS-encrypted data channel.
- A client must not read or inject messages in a room it has not joined (cross-room isolation), and must not see nearby devices outside its own IP group.
- A device in the room must not hijack or corrupt another peer's transfer. Resume needs the sender's per-file `resumeToken`.
- A malicious peer must not cause XSS in the web app (file names, device names, text), path traversal or file overwrite in the CLI, or unbounded memory or disk use on the receiver.
- Received files must match the sender's piece hashes.

## Components that matter most / least
- Most: `server/src/index.js`; `web/src/lib/warp/{signaling,peer,transfer,receiveController,receiveGuards,receiveResume,pieceManifest,hash,sha256,opfsStage,idbStage,zipDownload,uniqueName}.ts`; `cli/src/{protocol,peer,signaling,main}.rs`.
- Less: landing page, hero animation, brand kit, theory pages under `web/src/theory/`, `promo/` and `docs/`.

## How to exercise it
- `pnpm --filter @warp/web check:engine` runs the engine harnesses (`web/src/lib/warp/*.check.mjs`).
- `pnpm --filter @warp/server test` boots `wrangler dev` (local workerd) and drives real WebSocket clients against the Worker.
- `cd cli && cargo test --lib --offline` runs the CLI unit tests. `cli/target/debug/warp` is the built CLI.

## How you rate severity
- Critical: file bytes reaching the server or a third party; a peer outside the room reading or injecting signaling or data; remote code execution or arbitrary file write on a receiver.
- High: XSS from peer-controlled data; path traversal in CLI saves; hijacking another peer's transfer; a receiver accepting data that fails the piece hash.
- Medium: DoS of the signaling server or a receiver (memory, disk, CPU) by one client; leaking nearby-device names across IP groups.
- Low: metadata leaks that the docs already say the server can see.

## Anything to leave alone
- Room codes carry about 29.7 bits and anyone with a code can join. `docs/THREAT-MODEL.md` already documents this, and the small modulo bias in `makeCode()`. Report only a practical attack beyond it.
- Failing to connect through restrictive NATs is a known limit (no TURN relay), not a vulnerability.
- A peer you chose to share a room code with can send you files. That is the product.
