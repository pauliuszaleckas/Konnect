---
name: add-tool
description: Add or remove an MCP tool in Konnect, keeping the tool definition, handler, registry tool_count, tool-directory.md, DEV.md, and README counts in sync. Use when adding a new MCP tool, removing one, or moving a tool between toolsets.
---

Adding or removing a tool touches six places. The counts have drifted apart before
because only one of them got updated — work the list to the end.

## 1. Tool definition

Add the `tool!(...)` entry to the `tools()` vec in the owning toolset module,
`crates/konnect-core/src/tools/<toolset>.rs`. Tool names are lowercase `snake_case`,
begin with a concrete verb, and are **public API** — never silently rename or reuse one.
Prefer `get_` for a single object, `list_` for a collection, `create_` for a new
persisted object, `set_` for replacement.

Do not encode the transport (IPC vs. file editing) in the name; state the requirement in
the tool description instead.

## 2. Handler

Write `async fn handle_<tool_name>()` below the tools vec in the same module. The name
must mirror the tool name exactly — `place_component` → `handle_place_component`.

Use `require_str` / `require_f64` from `tools/mod.rs` for required arguments; they emit
`ToolErrorKind::InvalidArgument` automatically. For other failures prefer
`CallToolResult::error_kind(...)` with a `ToolErrorKind` variant over free text — see
`project.rs::handle_get_project_info` for the pattern.

Argument names are `snake_case`, carry units where ambiguous (`_mm`, `_nm`, `_degrees`,
`_ms`, `_bytes`), use `_path` for files and `_dir` for directories, and `_count` /
`_index` for quantities and positions.

## 3. `tool_count` in the registry

Update the toolset's `tool_count` in `crates/konnect-core/src/router/registry.rs`
(`ALL_TOOLSETS`). This is the declared count `list_toolboxes` reports — a registry
invariant test will catch a mismatch against the actual `tools()` vec.

Add the toolset to `STARTER_KIT` only if the tool must be callable with no
`load_toolset` hop. The starter kit is deliberately tiny (`project`, `config`) to keep
baseline `tools/list` near 2K tokens.

## 4. `tool-directory.md`

Update the toolset's table section **and** the Overview totals at the top — read the
registered-tool and total counts already there and adjust them; do not write a
remembered number.

## 5. `DEV.md`

Update the "Current Stats" section's tool counts.

## 6. `README.md`

Update the headline tool count in the same way — adjust the number the file already
carries.

## Verify

Run `cargo check --workspace`, then the `gate` skill. Confirm the counts in
`registry.rs`, `tool-directory.md`, `DEV.md`, and `README.md` all agree before finishing,
and say so explicitly — nothing in the test suite checks the docs.
