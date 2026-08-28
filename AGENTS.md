# Inkwash Workspace — e-ink calendar/alarms/todos: firmware, server, desktop tool, MCP

**Generated:** 2026-08-28
**Commit:** bfee251
**Branch:** main

## OVERVIEW
Umbrella repo for the four-repository Inkwash system: an ESP32-S3 e-paper notebook (Zectrix Note 4) showing calendar/alarms/todos, ringing alarms offline via the PCF8563 RTC. This repo tracks only `README.md`, `LICENSE`, `.gitignore` — the product repos live in gitignored sibling directories, each an independent git repo. No code here.

## STRUCTURE
```
inkwash-workspace/
├── inkwash-firmware/   # Rust ESP-IDF firmware + host-testable logic crate + USB/BLE/sync protocols
├── inkwash-server/     # Rust axum backend + SQLite/Postgres + embedded Vue admin-ui
├── inkwash-desktop/    # Tauri 2 PC tool (Rust + Vue): configure device over USB/BLE
├── inkwash-mcp/        # TypeScript/Bun MCP server: agents push notifications to device
├── README.md           # system overview, architecture diagram, getting started
├── inkwash.code-workspace  # VS Code multi-root
└── LICENSE             # Apache-2.0
```

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Cross-repo sync contract (alarms/todos JSON, ETag) | `inkwash-firmware/docs/sync-api.md` | frozen by server `models.rs` fixtures + pinned `inkwash-logic` rev |
| USB/BLE command protocol | `inkwash-firmware/docs/control-protocol.md` | contract with inkwash-desktop |
| Shared wire types | `inkwash-logic` crate (git-pinned from firmware repo) | deserialized by firmware, re-exported by server |
| Device-side state/alarm behavior | `inkwash-firmware/rust-firmware/src/` | module map in its AGENTS.md |
| Backend API + storage | `inkwash-server/src/` | see its AGENTS.md |
| Device registration/config | `inkwash-desktop/` | USB serial/BLE, admin API client |
| Notification webhook / MCP | `inkwash-mcp/` | `notify` tool posts to the server webhook |
| Hardware/board reference | `inkwash-firmware/docs/development-guide.md` | GPIO/power rails/EPD; safety section |

## CONVENTIONS
- Each subdir is an independent git repo with its own remotes; the umbrella never tracks code.
- Conventional Commits, English subjects (`feat:`/`fix:`/`docs:`/`chore:`/`build:`), lowercase type. Firmware `docs/` are written in Chinese.
- Wire contracts change only by bumping the pinned `inkwash-logic` rev (server Cargo.toml; checked by `scripts/check-logic-pin.sh`) — both sides move together, and `sync-api.md` updates in the same change.
- ts-rs codegen: server + desktop DTOs regenerate `*.ts` bindings on `cargo test` — commit refreshed bindings with the model change.
- Rust style is `cargo fmt` defaults; clippy `-D warnings` enforced in firmware `logic/` CI; desktop keeps clippy at zero locally.

## ANTI-PATTERNS (THIS PROJECT)
- **Breaking the wire contracts casually** — `Repeat` is serde externally tagged (`"Daily"` / `{"Once":…}`); the UI sends `repeat: "Daily"`. Enforced: server `models.rs` fixture tests + `docs/sync-api.md`.
- **Committing secrets** — server `.env` (gitignored; holds `ADMIN_TOKEN`), device tokens, logs with credentials. Never log `DATABASE_URL` (server `main.rs:25`); desktop redacts secrets (`logs.rs`).
- **Cross-flashing NOTE4 / NOTE4C** — incompatible hardware/waveforms; restore only from this unit's own backup (firmware red line #1).
- **Retagging to re-trigger releases** — GitHub Actions runs the workflow at the tagged commit; a retag must point at a commit with the latest workflow, after deleting the old release + tag (firmware/server/desktop `release.yml`).

## UNIQUE STYLES
- Four independent repos, no monorepo tooling: no workspace configs, no shared lockfile; cross-repo sharing is a git-pinned crate (`inkwash-logic`) + documented protocols.
- Server embeds `admin-ui/` (Vue 3) into the binary at compile time (rust-embed; `build.rs` runs `npm run build` — needs `node_modules`).
- Firmware builds/flashes locally (`scripts/build-rust.sh` sources ESP-IDF; CI only covers `logic/`); releases built locally and published via `scripts/release.sh`.
- E-paper UI preview: `tools/preview` renders real firmware screens to PNG on PC before flashing.
- Desktop binary is dual-mode: headless CLI (`--status`/`--sync`/`--ble-scan`/`--ble-list`) plus Tauri GUI.

## COMMANDS
```bash
# server (backend + embedded admin UI)
cd inkwash-server && ./scripts/start.sh        # first-run setup: .env, npm install, cargo run --release
# desktop
cd inkwash-desktop && npm run tauri dev        # dev app (vite :1420)
# firmware (needs ESP-IDF toolchain)
cd inkwash-firmware && ./scripts/build-rust.sh --release
# mcp
cd inkwash-mcp && bun install && bun run src/index.ts   # needs INKWASH_CHANNEL_ID + INKWASH_WEBHOOK_TOKEN
```

## NOTES
- `inkwash.code-workspace` opens the four repos; `.claude/` and `.omo/` are tooling dirs, gitignored.
- Server `build.rs` panics without `admin-ui/node_modules` — run `npm install --prefix admin-ui` once after cloning.
- Desktop log dir must stay outside the project tree (`~/Library/Logs/inkwash-desktop`) or `tauri dev` restart-loops.
- Firmware verification is flash + monitor (`espflash`) — "it compiles" is not vouch; see `development-guide.md` §13 smoke checklist.
- Not yet verified on device: full alarm-ringing flow and BLE end-to-end pairing (firmware).
