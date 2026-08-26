---
name: gate
description: Run the full local CI gate for Konnect — workspace tests, doctests, clippy, rustfmt, plus the schematic-viewer tests that --workspace skips. Use before opening a PR, before declaring work done, or when the user asks to "run the gate", "check CI locally", or "verify this passes".
---

Run the exact commands CI runs. If these pass locally, CI should be green.

`protoc` must be resolvable first — either `PROTOC` is set or `protoc` is on PATH.
Check with `protoc --version`; if it's missing, stop and tell the user to install it
(`apt install protobuf-compiler` / `brew install protobuf` / `choco install protoc`)
rather than reporting a build failure as a code problem.

Run in this order, stopping to fix rather than pushing past a failure:

```bash
cargo fmt --all -- --check
cargo clippy --workspace --locked -- -D warnings
cargo test --workspace --locked --lib --tests
cargo test --workspace --locked --doc
```

`--doc` cannot be combined with `--lib`/`--tests`, which is why it is a separate run.
Doctests broke silently once because of exactly this.

Then, **only if the change touched `crates/schematic-viewer`** (it is excluded from the
workspace, so nothing above tests it):

```bash
cd crates/schematic-viewer && cargo test --locked
```

If the change added or removed MCP tools, also verify the counts are in sync — see the
`add-tool` skill's checklist. A count drift is not caught by any of the commands above.

Report each command's outcome plainly. If a command fails, show the relevant output and
fix the cause; do not summarize a failure as a pass.
