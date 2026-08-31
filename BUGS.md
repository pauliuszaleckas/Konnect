# Known bugs

Found while dogfooding Konnect on a real board (ch347t-swd-debugger: 25 footprints,
99 nets, KiCad 10.0, Fedora). Entry 1 came from the PCB side, entry 2 from a second
redraw of the same board's schematic against the same design record, and both are fixed
on this branch. Entries 3 and 4 are not from the board at all — they came out of
reading the code while splitting this branch up. Entry 3 is fixed here too; entry 4 is
not.

The tracker has been searched (2026-08-16, again 2026-08-17 for entry 2, 2026-08-26 for
entries 3 and 4, and again 2026-08-31 for entry 3), and each **Status** line says what
it found. None of the four is filed, and none of the three fixes exists anywhere but
this branch.

An entry is deleted once its fix is merged, and the rest are renumbered, so nothing
outside this file cites one by number.

---

## 1. IPC socket path is not auto-detected

**Status:** FIXED on `paulius-next` — not filed; [#18] and [#39] are the closed
issues this friction produced, both resolved by documentation and by making the env
var apply on every load path.

**Severity:** low — one-time setup friction, but the failure is opaque.

`crates/konnect/src/config.rs:75` resolves the socket from `KICAD_API_SOCKET` and
nothing else. That variable is only set for plugins KiCad launches itself, so a
standalone server started by an MCP client sees an empty address and every IPC call
falls back to file editing — with no message saying why. On this machine the socket was
at the predictable `/tmp/kicad/api.sock` and had to be written into `settings.json` by
hand.

**Fix:** when the env var is absent, probe the platform default (`/tmp/kicad/api.sock`
on Linux, the documented equivalents elsewhere) before giving up, and log at `warn` when
an IPC-capable tool silently falls back to the file path.

**What landed:** `konnect_ipc::socket` resolves the address KiCad would be listening on
— `<temp dir>/kicad/api.sock`, plus `/tmp/kicad/api.sock` on macOS, where the temp dir
resolves under `/var/folders` — and `Config::resolve_ipc_address` consults it after the config file and
`KICAD_API_SOCKET`, reporting which of the three answered. Verified end to end: with no
config and no env var, `check_kicad_ui` now returns `ipc_responsive: true` against a
running KiCad.

A candidate counts only when something is **listening** on it, not when the path exists:
KiCad does not unlink `api.sock` on exit, so a socket from a session two days ago sits
exactly where a live one would. One `connect()` separates them, bounded by a 250 ms
timeout: `connect()` on an `AF_UNIX` stream blocks while the listener's backlog is full,
and this resolution runs before tracing is initialized, so an unbounded wait would hang
startup with nothing on stderr. Only a socket this user owns qualifies — a detected one
is adopted as the board endpoint unread, and the shared `/tmp` candidate sits in a
world-writable directory.

Windows probes nothing: NNG's `ipc://` is a named pipe with nothing at that path to
connect to, so `is_listening` answers `false` there and detection finds nothing. Probing
the pipe properly (`CreateFileW` against the name NNG derives) is left for a change that
can be tested on Windows.

Detecting nothing deliberately leaves `ipc_address` empty rather than guessing, which
keeps the "socket path not configured" error and its setup steps in place instead of
replacing them with a dial failure against an address nobody chose.

The diagnostics half:

- Startup logs the resolved address and its source, and `warn`s with the candidate list
  when there is none.
- Every IPC failure that reaches KiCad through `with_ipc`/`with_ipc_classified` and
  never gets through now `warn`s with the address it tried. KiCad-*rejected* calls stay
  silent: they fail closed and report themselves. Five sites still build a
  `KiCadIpcClient` directly and bypass both seams (`project.rs:298` and `:370`,
  `verification.rs:822` and `:880`, `pcb_routing.rs:1735`), so their failures remain
  unlogged — worth routing through the gate on its own.
- The dial error names the three things that produce it, a stale socket included.

Wiring that warning wanted one seam per failure shape. `with_ipc_classified` was
already one, and every board tool reaches it through `with_board_ipc_classified`.
`pcb_export` still had a private `with_ipc` — the last of the four per-toolset copies,
the other three having been folded upstream — and it is now `tools::with_ipc` beside
the classified gate, so the next toolset gets the warning for free and the drift
`with_ipc_classified`'s doc comment complains about is gone.

Also fixed on the way: `ffi.rs` resolved nothing at all, so a cdylib loaded by KiCad with
`--config` never picked up `KICAD_API_SOCKET` — the gap [#39] closed for `main.rs` only.
Both entry points now load through one `Config::load_resolved`, so a future third cannot
reopen it.

**Known limitations.** Windows gains no auto-detection — it keeps exactly the setup it
had before, an address from config or `KICAD_API_SOCKET` or nothing. That is deliberate:
answering `true` unconditionally would make `IpcAddressSource::Unresolved` unreachable
there and retire the two messages a Windows user needs most — the "socket path not
configured" error with its settings-dialog steps, and the startup `warn` listing the
candidates — in favour of a dial failure against an address nobody chose.

And detection runs once, at startup. The usual MCP launch order is
client → server → *then* the user opens KiCad, and in that order the address stays
unresolved for the whole session — `launch_kicad_ui(wait_ready)`
(`verification.rs:854`) snapshots the empty address before spawning KiCad, so it can only
time out. Covering it means re-probing at the read site while the address is empty
(a `ToolContext` accessor rather than a plain field), which is worth doing on its own.

[#18]: https://github.com/mixelpixx/Konnect/issues/18
[#39]: https://github.com/mixelpixx/Konnect/issues/39

---

# Schematic-side bugs

Found redrawing the same board's schematic from an empty file through the MCP tools.
The first redraw's own findings have since been fixed and merged upstream, and their
entries deleted — including the four-way connectivity divergence they left behind,
which [#323] closed by giving the tools one shared index.

The entry below comes from drawing that schematic a second time, from an empty A3
sheet, against the same design record: 27 symbols, 31 power symbols, 26 nets, 21 wires,
39 labels, 9 no-connect flags. The sheet passes `kicad-cli` ERC with zero violations and
its netlist is node-for-node identical to the first redraw's, so where a tool disagrees
below, the tool is wrong. That pass's other finding — every schematic audit resolving
nets by a 0.5 mm text scan for labels — is merged as [#336].

Observed against a `paulius-next` build carrying the schematic fixes since merged
upstream: this pass hit none of them. `find_orphan_items` reported 0 orphans,
`validate_wire_connections` 0 floating ends, `validate_component_connections` 0
unconnected pins, and `list_schematic_nets` all 26 nets including the two power-symbol
rails.

[#323]: https://github.com/mixelpixx/Konnect/pull/323
[#336]: https://github.com/mixelpixx/Konnect/pull/336

---

## 2. `find_single_pin_nets` counts labels, not pins

**Status:** FIXED on `paulius-next` — not filed; the tracker has nothing on this tool.
[#249] is the same pin blindness one tool over — `find_orphan_items` — and closed
without reaching this one.

**Severity:** medium — 11 false positives out of 11 nets reported, and the real defect
it advertises can pass through unreported.

`sch_analysis.rs::handle_find_single_pin_nets` counts *label instances* per net name and
reports every net with exactly one:

```rust
for l in &labels { *counts.entry(l.net.clone()).or_insert(0) += 1; }
let singles = counts.iter().filter(|(_, &c)| c == 1)
```

Pins, wires and junctions are never consulted, so a net drawn the normal way — one label
on a wire that reaches two or more pins — is reported as a single-pin net. Every net on
this sheet that is wired rather than labelled at both ends came back: `VBUS` (five pins),
`ACT`, `ACT_A`, `CC1`, `CC2`, `PWR_LED_A`, and all five `*_INT` nets between U1 and its
series resistors.

It misses the case it exists for just as readily. A net that really does reach one pin
is invisible as soon as it carries two labels, and a one-pin net named by a power symbol
never appears at all — `extract_all_net_labels` includes power symbols since that fix, so
the rail *is* counted, but a rail with one pin on it is not what this tool is looking
for.

**Introduced by** `dd49a86` (2026-07-05, the initial public release); the logic is
untouched since, and the power-symbol fix only changed which extractor feeds it.

**Fix:** count pins on the net — `sch_connectivity`'s `ConnectivityIndex` and
`seed_net_graph`, which [#323] gave the other connectivity tools — and report a net whose
pin count is 1. The label count is worth keeping as a separate field, since a net with no
label at all is a different smell.

**What landed:** the handler counts connection points on the shared net graph and
reports a net that reaches at most one. Each entry carries `pin_count`, `label_count`
and the `pins` that were counted, so the label count is still there to read and is no
longer the answer.

Zero counts as well as one, which the fix above did not say. A label whose net reaches
nothing is the orphan label and the deleted-component stub the review skill sends this
tool looking for, and the label count reported it; `find_orphan_items` names the
dangling wire end but never the net. Reporting only `1` would have retired a documented
behaviour, so `pin_count` carries the real number.

Two connection points are deliberately treated unalike:

- A hierarchical sheet pin counts. A net leaving the sheet through one reaches whatever
  is on the other side, and not counting it would report every such net.
- A power symbol's own pin does not. It names the rail rather than consuming it, and
  counting it would hide the rail that reaches exactly one component pin — which is the
  whole reason a rail-named net is worth reporting. KiCad marks those symbols,
  `PWR_FLAG` included, with a `#`-prefixed reference.

The label-to-root walk moved into `sch_connectivity` as `label_roots`, since
`find_shorted_nets` reads the same relation from the other end and the two drifting
apart is what that module exists to prevent. The `#`-prefix rule is
`is_power_symbol_reference`, one home for a heuristic `design_review` had inline too.

The report is sorted by net name; it came out of a `HashMap` in whatever order it held
them, which is the same coin toss entry 4 is about.

**Known limitation.** The answer is per sheet, so a net named by a global or
hierarchical label may well continue on another one — on a hierarchical child sheet,
every hierarchical label wired to a single pin is the ordinary way to draw it, and is
reported. Each net carries the kind of label that named it and the tool description says
so, which is as far as a single-sheet reader can go; ranking a `NetLabel` net above a
hierarchical one is the follow-up.

[#249]: https://github.com/mixelpixx/Konnect/issues/249
[#323]: https://github.com/mixelpixx/Konnect/pull/323

---

# Review-side bugs

Not from the board either. Both came out of a review pass over this branch on
2026-08-26, after the audit net-resolution fix ([#336]) landed. Entry 3 is fixed on this
branch; entry 4 is not.

---

## 3. A reachable KiCad without this board open refuses the write

**Status:** FIXED on `paulius-next` — not filed. [#240] was the same classification
seam from the other side — whether `Unreachable` is safe to fall back on when KiCad
*just died* holding the board — and has since closed. This direction was untouched:
`Rejected` is not safe to refuse on when KiCad never had the board at all. [#241]
tracks the untestability of the neighbouring refusal branch.

**Severity:** high — it takes a headless board edit that worked and turns it into a
refusal that writes nothing, and the trigger is "KiCad happens to be open".

`attempt_ipc_write` (`pcb_board.rs:213`) calls `ensure_board_is_active` and then
classifies whatever comes back:

```rust
Err(konnect_ipc::IpcFailure::Rejected(message)) => … BoardWrite::Refused(…)
Err(konnect_ipc::IpcFailure::Unreachable(_))    => … BoardWrite::File
```

[#240] has since narrowed that second arm — the file fallback is refused once this
server has observed the board live — and left the first one alone.

`find_open_board` (`client.rs:566`) bails with a plain `anyhow` error — `"requested
board 'B' is not open in KiCAD (open boards: A)"`, or `"No PCB document is open in
KiCAD. Open a board file first."` — and neither carries the `TransportUnreachable`
marker, so `IpcFailure::from_error` classifies both as **`Rejected`**. Every
`attempt_ipc_write` caller then answers:

> KiCAD rejected the … over IPC … The board file was not modified — KiCAD is reachable
> and may hold this board open, so editing the file directly could be silently
> overwritten.

KiCad never rejected anything. It was asked about a board it does not have, and the
premise of the message — "may hold this board open" — is the one thing
`find_open_board` has just established to be false.

Two ordinary situations hit it: KiCad open on project A while Konnect lays out project
B, and KiCad running with no board open at all, which is what a user who has just
launched it has. `add_board_outline`, `set_board_size`, `add_zone`, `delete_graphics`
and the rest of the `attempt_ipc_write` callers all refuse and write nothing.

`refuse_if_board_open_in_kicad` (`pcb_board.rs:263`), just below, draws the
distinction correctly — `Ok(()) => refuse`, `Err(_) => proceed with the file` — and its
doc comment states the rule this one breaks: "a reachable KiCAD holding a *different*
board … does not interfere".

**Made reachable by entry 1.** Before socket auto-detection a standalone server had an
empty `ipc_address`, so the dial failed, every call classified `Unreachable`, and the
file fallback ran. Now the address resolves whenever KiCad is up. The classification
bug is older than the detection; the detection is what put it in everyone's path.

**Fix:** `ensure_board_is_active`'s "not open" is neither a transport failure nor a
rejection — it is a third answer, and `attempt_ipc_write` needs to tell it apart from
the second. Either give the not-open error its own marker type alongside
`TransportUnreachable` and fall back to the file on it, or split the board check out of
the closure so `attempt_ipc_write` decides on `find_open_board`'s result directly
rather than on a classified error. Marker over message text, the way
`is_transport_unreachable` already works.

**What landed:** the marker. `konnect_ipc::BoardNotOpen` rides the error chain out of
`find_open_board` and `get_board_document`, `IpcFailure::from_error` classifies it as a
third variant `BoardNotOpen`, and every site that decides what to do next tells it
apart from a refusal:

- `attempt_ipc_write` answers `BoardWrite::File`. A KiCad that never opened this board
  holds no unsaved state for it, so the saved file is authoritative.
- `pcb_components`' `get_component_pads` reads the file. Its `Rejected` arm returned an
  error rather than falling through, so a KiCad open on another project made a
  file-answerable read fail.
- `refuse_if_board_open_in_kicad` and the IPC-only tools behind `pcb_components`' `ipc!`
  macro keep the answer they gave — the first already proceeded on `Rejected`, and the
  second has no file path to fall back to. Both now say so in their own arm instead of
  relying on the catch-all, and the first has a test pinning it.

**Gated on the same memory as `Unreachable`,** which the fix above did not say and a
review pass caught. `with_board_ipc_classified` observes the board *before* running the
closure, so "KiCad no longer holds a board this session watched it hold" is reachable:
KiCad crashed with unsaved work and was restarted without it, or the user closed it
mid-operation. A reachable transport says nothing about the work that board carried, so
that case stays [#240]'s refusal and only a board never observed live takes the file
path. The refusal now names which of the two changed — the transport, or the board —
rather than asserting an unreachable IPC on both.

**The file path had to learn why it was taken.** `BoardWrite::File` carries a
`NoLiveBoard` — `Unreachable`, or `NotOpen` with KiCad's own answer naming the boards it
does hold. Three tools (`update_pcb_from_schematic`, `update_footprints_from_library`,
`repair_corrupted_footprints`) are live-IPC-only and led their refusal with "KiCad IPC is
unreachable", which the new path makes false; they now lead with the premise that
applies, and the not-open one keeps the board list that the old `Rejected` route
happened to surface. `FILE_FALLBACK_WARNING`, attached by seven file-editing tools, said
the same thing and now covers both causes, as does `get_component_pads`' description.

Tested at both levels: `konnect-ipc` pins the classification for a KiCad holding another
board and for one holding none, and `pcb_board` drives `add_mounting_hole` against a mock
KiCad open on a different project — asserting both the file write and the warning that
describes it — alongside the refusal for a board observed live and since closed.

[#240]: https://github.com/mixelpixx/Konnect/issues/240
[#241]: https://github.com/mixelpixx/Konnect/issues/241

---

## 4. A net with two names is audited as two rails

**Status:** not fixed, not filed; the tracker has nothing on `net_at` or on net naming.

**Severity:** medium — a false `error`-severity finding on a correctly decoupled rail,
and which rail it lands on changes between runs.

`NetGraph::net_at` (`sch_connectivity.rs:234`) answers "what net is at this point" by
scanning `point_nets` for the first entry whose union-find root matches:

```rust
let labels: Vec<_> = self.point_nets.clone().into_iter().collect();
for (lk, net) in labels { if self.find(lk) == root { return Some(net); } }
```

That is `HashMap` iteration order. When one electrical net carries two names — a rail
named by a `+3V3` power symbol that also has a label on it, a local label on a net that
also has a global one — the name that comes back is arbitrary. `collect_power_nets`
meanwhile lists **both** names as rails, because both are labels on the sheet. So every
capacitor on that net resolves to one name, and the other rail is reported as having no
decoupling at all.

Reproduced on [#336]'s own test sheet with one label added to the `+3V3` rail, three
consecutive runs of the same binary:

```
run 1  error: Power rail '+3V3' has no decoupling capacitors
run 2  error: Power rail '+3V3' has no decoupling capacitors
run 3  error: Power rail 'VCC'  has no decoupling capacitors
```

C2 (10 µF) is on that net in all three. Within a run the choice is stable —
`point_nets` is never mutated after seeding, so one graph gives one answer — which is
why this survived [#336]'s tests: it takes a second name on a net, and it moves
between processes rather than within one.

The audits are the visible victim, but the naming is `net_at`'s, and ten read-only
tools in `sch_analysis` and `sch_export` call it. `get_pin_net` and `get_component_nets`
report the same arbitrary pick.

**Fix:** the audits compare nets by *name* where they mean *identity*. `power_nets`
should be the graph's root for each rail, not its label text, and `cap_nets` the roots
the capacitor pins reach — `points_on_net` already goes the other way and could
anchor it. Failing that, `net_at` should at least choose deterministically (KiCad's own
precedence is global over local over power symbol) so two names on a net stop being a
coin toss. The clone in that loop is worth taking with it: it copies every key *and*
every net-name `String` on every query, and [#336] made `design_review` the heaviest
caller in the tree.

[#336]: https://github.com/mixelpixx/Konnect/pull/336

---

## Notes

- `rotate_component` takes `rotation`, not `angle`. Not a bug — the structured
  `invalid_argument` error named the right field and recovery took one retry. Worth
  keeping as an example of the error taxonomy paying off.
- `batch_delete` given a reference that appears twice deletes one and reports success.
  Defensible, but it makes the duplicate `#PWR` designator harder to climb out of.
- Power-symbol rotation is consistent and worth writing down: rotation adds to the
  pin's `orientation_degrees`, so `GND` at 90 and `+3V3` at 270 both put the body to
  the right of a wire arriving from the left. `get_schematic_pin_locations` on the
  `#PWR` reference confirms it without a render.
- Parallel MCP calls against one schematic were safe in practice — 10 concurrent
  `add_power_symbol` calls serialised cleanly with no revision conflict and no lost
  writes.
- Creation leaves files at mode `0600` rather than the umask default, presumably the
  scratch file's mode surviving the rename: `create_schematic`'s `.kicad_sch`, and also
  `create_project`'s `.kicad_pro` and `.kicad_pcb` and `register_symbol_library`'s
  `sym-lib-table`. `create_symbol`'s `.kicad_sym` and kicad-cli's own outputs come out
  `0644` under the same umask, so it is not the whole writer path. Rewrites preserve
  whatever mode the file already had, so this only bites on creation — and a project
  created through Konnect is unreadable to a second user or a CI job running as another
  account.
- KiCad 10.0.5's own ERC JSON reports positions divided by 100: a violation at
  114.3 mm comes out as `1.143` under `coordinate_units: "mm"`, and `--units in` gives
  `0.045`, which is that wrong value converted. Upstream, not Konnect's — but the
  coordinate Konnect passes through cannot be found on the sheet, so do not chase it.
- `run_erc` writes `<schematic>.erc.json` beside the schematic and removes it afterwards.
  It is the one read-shaped schematic tool that creates a file in the user's project, and
  two runs against the same sheet share that path.
- `trace_from_point` reports `wires_here` and `labels_here` only. On a capacitor pin
  sitting mid-wire under a junction dot it names the net correctly and reports neither the
  pin nor the junction, so "what is at this point" is answered by two of the four things
  that can be there.
- `batch_connect_to_net` writes a label at the pin endpoint and no wire, while
  `connect_to_net`'s description promises "a short wire stub and a net label". Both net up
  identically; worth knowing before comparing wire counts between two sheets.
- `get_footprint_info`'s description promises "pad layout", and the response carries only
  `pad_count` — no pad positions, sizes, shapes or numbers. Checking a placement against
  real pad geometry, which DESIGN-style board work needs, therefore still means parsing
  the `.kicad_mod` directly. Courtyard geometry is not missing: `include_graphics` with
  `graphics_layer: "F.CrtYd"` returns it.

- `add_power_symbol` accepts `PWR_FLAG` and auto-numbers it into the same `#PWR` sequence,
  so a flag and the rail symbol it flags can be stamped at one point — the flag's body
  draws up, the rail's down, and `check_schematic_overlaps` reports the pair as an
  overlap. That report is correct and the placement is intentional; it is the one overlap
  on this sheet.

- The IPC-first board readers [#207] merged cover `get_board_info`, `get_component_pads`
  and `get_pad_position` through it. The rest of `pcb_board`'s readers — `get_layer_list`,
  `list_zones` — and the file scans in `pcb_routing` still parse the file while the
  writers beside them use IPC, so on a board with unsaved changes they answer about a
  board KiCad no longer has. Each needs its own IPC mapping and some have no API
  equivalent, so it is a follow-up rather than a regression. [#153] has since rewritten
  `get_layer_list` to read the stackup by shape, which makes that one a clean starting
  point.

[#153]: https://github.com/mixelpixx/Konnect/pull/153
[#207]: https://github.com/mixelpixx/Konnect/pull/207