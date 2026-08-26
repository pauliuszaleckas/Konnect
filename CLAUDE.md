# CLAUDE.md

This file guide Claude Code (claude.ai/code) when work with code in this repo.

## Build requirements

`protoc` must be on hand — `konnect-ipc/build.rs` generate protobuf code at build time. Without it, whole workspace fail to build.

`protoc` on PATH enough. `PROTOC` override binary, `PROTOC_INCLUDE` override well-known-types include dir.

Well-known `.proto` files ship separate from compiler on some distros — Debian need both `protobuf-compiler` and `libprotobuf-dev`, Fedora both `protobuf-compiler` and `protobuf-devel`. Missing them fail with `google/protobuf/any.proto: File not found`.

Rust toolchain pinned in `rust-toolchain.toml` (1.96.0), and that pinned version IS the MSRV. Bump deliberate, own commit, after full gate run.

## The gate

Exactly what CI run. Run all four before call work done:

```bash
cargo test --workspace --locked --lib --tests
cargo test --workspace --locked --doc      # --doc cannot be combined with --lib/--tests
cargo clippy --workspace --locked -- -D warnings
cargo fmt --all -- --check
```

`crates/schematic-viewer` **excluded from workspace** (Tauri app). `--workspace` never touch it, break there ship silent — test explicit when you change it:

```bash
cd crates/schematic-viewer && cargo test
```

## Adding or removing an MCP tool

Four counts must move together or they drift (happened before):

1. `tool!(...)` definition in toolset's `tools()` vec, handler named `handle_<tool_name>` below it.
2. `tool_count` in `crates/konnect-core/src/router/registry.rs` (`ALL_TOOLSETS`).
3. Matching section and Overview totals in `tool-directory.md`.
4. "Current Stats" in `DEV.md` and tool count in `README.md`.

Add toolset to `registry.rs::STARTER_KIT` only if tool must be available without explicit `load_toolset` call — starter kit deliberate tiny to keep `tools/list` around 2K tokens.

## Public API surface

MCP tool names, toolset names, tool arguments, CLI flags, environment variables, config keys, JSON keys, documented filesystem paths — all public API. Never silent rename or repurpose one — add alias/deprecation period and migration note.

## Naming

Full rules: @docs/NAMING_CONVENTIONS.md

Two easy to get wrong:

- New prose use `KiCad`, but existing codebase use `KiCAD` in ~400 places and is **not** to be mass-renamed. Match surrounding style when edit existing text.
- Konnect-owned JSON keys and tool arguments are `snake_case`. Protocol-defined MCP and JSON-RPC fields keep spec spelling (`jsonrpc`, `serverInfo`, `tools/list`). KiCad plugin manifests keep KiCad's field names.

## Schematic file writes

Writes to existing `.kicad_sch` files go through revision-checked atomic replacement: read exact source, take cooperative lock, reject any intervening KiCad or Konnect change, write sibling scratch file, fsync, rename.

`Schematic::save(new_path)` is create-only. `save(loaded_path)` and `overwrite()` replace only when file still exactly match what was loaded. Do not delete destination to work around conflict, and do not weaken `save()` into unconditional overwrite — KiCad autosave producing conflict is mechanism working, and callers resolve by reload and reapply.

`.konnect-transaction-*.json` journals hold complete before/after file images. Treat as sensitive project data.

## Repo etiquette

- Branches: `fix/indent-safe-wire-delete`, `feat/linux-pcm-support`, `docs/naming-conventions`.
- PR titles: imperative Conventional Commit style — `fix(schematic): preserve tab-indented wire blocks`. Types: `fix`, `feat`, `docs`, `test`, `refactor`, `build`, `ci`, `chore`.
- One reviewable outcome per PR. Split unrelated platform, protocol, feature, doc changes into series.

## Architecture

@DEV.md cover crate layout, KiCAD 10 IPC vs. S-expression editing, structured error taxonomy (`ToolErrorKind`), observability, toolset router.