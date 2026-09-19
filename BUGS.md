# Known bugs

Found while dogfooding Konnect on a real board (ch347t-swd-debugger: 25 footprints,
99 nets, KiCad 10.0, Fedora), plus what came out of reading the code while splitting
this branch up.

Every entry below was re-tested on 2026-09-13 against Konnect 0.11.1 built from `main`
@ 39e0061, with KiCad 10.0.5 on Fedora, driving the freshly built server over stdio —
not through the installed plugin binary, which was still 0.6.1 and would have answered
for month-old code. Entry 2 was then retested on the 10.0.6 upgrade and is unchanged;
the others were not re-run against 10.0.6, and only entry 2 depends on the KiCad version
at all. The tracker was searched the same day and each **Status** line says what it
found.

On 2026-09-15 the whole schematic side was exercised end to end by rebuilding this
board's schematic from `DESIGN.md` alone into an empty project — symbol library, 27
parts, 26 nets, ERC — with KiCad 10.0.6. That run reproduced entries 1 and 2
independently, and ended with a netlist matching `DESIGN.md` node for node and a clean
ERC at all severities, so the tools it did exercise are not hiding violations.

Also on 2026-09-17 entry 2 was cut down to a 93-line minimal sheet for an upstream
report, which showed the scaling to be confined to KiCad's ERC *JSON* writer — the text
report on the same sheet is correct. It was filed upstream the same day as
[kicad#25582]; the KiCad tracker was searched first and carried nothing for it.

On 2026-09-17 the same rebuilt board turned up entry 4, in the one schematic operation
that run had not exercised: turning a symbol that was already placed. Measured against
`paulius-next` @ 6ab7ec2 with KiCad 10.0.6, with `kicad-cli sch export svg` as the
oracle for where the text actually lands.

On 2026-09-19 this branch was rebased onto `main` @ c03a13d (v0.12.1) and the four
remaining entries were re-read against that tree rather than re-run. Entry 1's create
path is byte-identical — `write_new_atomic_unlocked` still builds its scratch file with
`tempfile::Builder`, and `persist_journal` still shares it — and so is entry 2's
`sch erc --format json` invocation and `parse_item_pos`, so those measurements still
describe this tree. Entry 3 was re-counted rather than re-read, because `main` has
grown IPC handlers since it was written: both readers it names are still file-only and
the imbalance around them is larger, not smaller. Entry 4 is the defect this branch
fixes. The tracker was re-checked the same day — [#538], [#541], [#542] and [#612] are
all still open.

An entry is deleted once its fix is merged, and the rest are renumbered, so nothing
outside this file cites one by number.

The eight entries this file used to carry are all merged upstream — IPC socket
auto-detection as [#382], `find_single_pin_nets` counting pins as [#402], the
board-not-open classification as [#407], the two-named-net audit as [#513], `run_erc`
writing into the project directory as [#570], which moved the report to a private
temporary directory removed even when the run fails, `trace_from_point` omitting pins
and junctions as [#588], the unbounded stale-target refusal as [#605], and rotation
never reconciling its junctions as [#642] — the last reconstructed by the maintainer
onto the pin-owned no-connect contract from [#641], with [#627] and [#632] closed as
superseded copies. Socket discovery has since been narrowed further by [#505], which
replaced the connect probe with a metadata read.

[#382]: https://github.com/mixelpixx/Konnect/pull/382
[#402]: https://github.com/mixelpixx/Konnect/pull/402
[#407]: https://github.com/mixelpixx/Konnect/pull/407
[#505]: https://github.com/mixelpixx/Konnect/pull/505
[#513]: https://github.com/mixelpixx/Konnect/pull/513
[#570]: https://github.com/mixelpixx/Konnect/pull/570
[#588]: https://github.com/mixelpixx/Konnect/pull/588
[#605]: https://github.com/mixelpixx/Konnect/pull/605
[#627]: https://github.com/mixelpixx/Konnect/pull/627
[#632]: https://github.com/mixelpixx/Konnect/pull/632
[#641]: https://github.com/mixelpixx/Konnect/pull/641
[#642]: https://github.com/mixelpixx/Konnect/pull/642

---

## 1. Created files land at mode 0600

**Status:** CONFIRMED on 0.11.1; filed as [#538] on 2026-09-13. [#555] proposed a fix
on the create path and was closed unmerged on 2026-09-18 with its review checklist
unanswered, so the issue is unclaimed again — but that review settled the shape the fix
has to take, recorded under **Fix** below. Reproduced from scratch on 2026-09-15: a `create_project` into an empty directory under `umask 022` again
produced `.kicad_pro`/`.kicad_sch`/`.kicad_pcb` and `sym-lib-table` at `0600`, with
`create_symbol`'s `.kicad_sym` at `0644` beside them as the control.

**Severity:** medium — a project created through Konnect is unreadable to a second user,
a CI job under another account, or a container user with a different uid.

Under `umask 022`, `create_project`'s `.kicad_pro`/`.kicad_pcb`/`.kicad_sch`,
`create_schematic`'s `.kicad_sch`, and `register_symbol_library`'s `sym-lib-table` all
come out `0600`. Two controls in the same directory and process rule out the umask and
the filesystem: `create_symbol`'s `.kicad_sym` is `0644`, and so is a `kicad-cli sch
export svg` output.

**Cause:** the replace path (`create_scratch_file`) opens with `OpenOptions`, so it gets
`0666 & ~umask`, and `write_atomic_unlocked_with` then copies the destination's mode over
it. The create path, `write_new_atomic_unlocked`, builds its scratch file with
`tempfile::Builder` — `0600` by design — and never adjusts it, because on creation there
is no destination to copy from. `0600` survives `persist_noclobber`.

**Wider than the dogfooding note said:** every `write_new_atomic` caller is affected,
including `export_specctra_dsn` and its manifest and the Freerouting SES output. Those
get handed to other tools and other people, so it is not only a readability annoyance.

`persist_journal`'s `.konnect-transaction-*.json` uses the same path and should *stay*
`0600` — those hold complete before/after file images. The fix has to keep that
deliberate rather than accidental.

**Fix:** give a newly created design file the mode `File::create` would have produced,
without carrying the journal along. [#555]'s review worked out what that costs: the
create path needs an explicit ordinary-versus-private policy rather than one blanket
mode, and the private side cannot stop at `tempfile::Builder::permissions(0o600)` —
that supplies the *creation* mode, which the umask still masks, so a restrictive umask
leaves a journal at `0400`, `0200` or `0000` when [#538] asks for exactly `0600`. The
private path has to chmod `0600` after creation and before persistence, tested under
explicit representative umasks in isolated subprocesses so the assertion does not depend
on the test runner's umask or mutate it for parallel tests, asserting owner read/write
and no group or other access. The negative control is neutralizing that enforcement: the
permissive-umask case must then fail.

## 2. `run_erc` coordinates are 100× too small

**Status:** CONFIRMED on 0.11.1 against KiCad **10.0.5 and 10.0.6**; filed as [#541] on
2026-09-13, retested on the 10.0.6 upgrade. Root cause is upstream and now filed there
as [kicad#25582] on 2026-09-17, so the fix here is compensation, not correction.
Reproduced independently on a second sheet on 2026-09-15: a symbol
`add_power_symbol` had just placed at `(120.65, 219.71)` came back from
`run_erc` at `(1.2065, 2.1971)`, and the raw `kicad-cli` report behind it says
`coordinate_units: mm`, so the ÷100 is in the report Konnect is handed, not in the
parse — `parse_item_pos` reads `pos.x`/`pos.y` verbatim.

**Severity:** medium — the coordinate cannot be found on the sheet, so a violation
cannot be located from Konnect's own output.

Konnect contradicts itself on one pin of one sheet: `get_schematic_pin_locations` puts
`R1` pin 2 at `(100.33, 104.14)`, and `run_erc` reports that same violation at
`(1.0033, 1.0414)` — exactly ÷100.

`kicad-cli` alone shows the same, and it is not a unit mix-up: `--units mm` gives
`1.0033` against a true `100.33`, and `--units in` gives `0.0395` against a true `3.95`,
which is the already-wrong millimetre value converted. KiCad scales before the unit
conversion. The 10.0.6 output is byte-identical to 10.0.5.

**Only the JSON writer is affected.** Measured on 2026-09-17 while building a 93-line
reproducer for the upstream report — one unconnected single-pin symbol at `(127, 88.9)`
mm, KiCad 10.0.6. `--format report` prints `@(127.00 mm, 88.90 mm)`; `--format json` on
the same sheet says `coordinate_units: mm` and `{"x": 1.27, "y": 0.889}`. ERC therefore
holds the right number and the JSON serializer scales it. All three unit modes on that
one file — `1.27, 0.889` mm against a true `127, 88.9`; `0.05, 0.035` in against `5.0,
3.5`; `50.0, 35.0` mils against `5000, 3500` — are the wrong millimetre value converted.

The factor has a plausible mechanism, untested and recorded upstream as such: `127 mm ×
10000 IU/mm = 1270000 IU` divided by the board scale of `1000000 IU/mm` is the reported
`1.27`, which fits the JSON writer formatting a schematic-IU value through `pcbIUScale`
where the text writer uses `schIUScale`.

**The fault is ERC-specific.** `pcb drc` on the same KiCad build reports correct
millimetres — `(136.19, 93.375)` against a footprint at `(140, 93.375)`, matching `y`
exactly — so the two report writers do not share the defect.

**Fix:** three options, now that the fault is localized to the JSON writer.
`run_erc` (`cli.rs:657`) invokes `sch erc --format json`, which is the broken path.
Scale on the way out, scoped to `run_erc` alone — applying it to `run_drc` would corrupt
coordinates that are currently right. Or omit the field, so a caller gets no coordinate
rather than a wrong one. Or read `--format report`, which is correct on both affected
releases and needs no version gate, trading JSON parsing for text parsing and a fixture
of its own. A version pin is the right shape but still has nothing to pin against: both
shipped 10.0.x releases are affected and [kicad#25582] carries no fix yet.

This is the one entry where "do not chase it" was the old advice; it is filed both
here and upstream now, because a caller cannot tell the number is unusable.

## 3. `get_layer_list` and `get_netclasses` answer from the saved file

**Status:** CONFIRMED structurally on 0.11.1; filed as [#542] on 2026-09-13. Not
reproduced against a live KiCad — see below.

**Severity:** low — same class as [#207], which is merged for three other readers.

Both reach only `read_to_string` + `parse_sexp`, while 8 handlers in `pcb_board.rs` and
10 in `pcb_routing.rs` go through IPC — counted again on 2026-09-19 against `main`
@ c03a13d, where the split is 8 against 2 and 10 against 2. The gap has widened since
this was filed (3 and 4 then), because every handler added in between took the IPC
path. Within one file a caller gets live answers from
some tools and saved-file answers from others, with nothing in either response saying
which.

**Scoped deliberately to reads.** The file-only *writers* in these toolsets go through
`write_atomic`, which calls `ensure_kicad_design_document_is_closed`, and
`kicad_editor_lock_path` covers `.kicad_pcb` as well as `.kicad_sch` — so they fail
closed against KiCad's `~<name>.kicad_pcb.lck`. No write is at risk. Reads take no lock,
correctly, which is why reads are the whole exposure.

**Not measured:** no divergent answer was demonstrated against a running KiCad holding
unsaved changes. That consequence follows from [#207]'s accepted premise rather than
from a fresh measurement.

**Fix:** an IPC path with a file fallback, or the `source` marker the IPC-first tools
already carry.

## 4. Rotating a placed symbol leaves its field text behind

**Status:** CONFIRMED on 0.11.1 built from `paulius-next` @ 6ab7ec2, KiCad 10.0.6 on
Fedora. Found 2026-09-17 while re-drawing this board's LED indicators and filed as [#612]
the same day. Fixed on this branch; [#614] carries the fix and is open with changes
requested — the mutation regressions must be rebuilt on a KiCad-authored fixture instead
of the strings the tests write themselves, per `docs/RELIABILITY_CONTRACT.md`. The entry
stays until that merges. The tracker was searched first and carried nothing for it; the nearest,
[#490], is about there being no tool to position field text at all, and was closed by the
tool that now exists.

**Severity:** medium — nothing is corrupted and the netlist is unaffected, but the sheet
is misread. The designator lands on the body it names.

Place a `Device:LED` unrotated and then call `rotate_schematic_component` to 90°. The
body turns; `D1` and `Green` stay 2.54mm above and below the origin, which on a vertical
body is the middle of it and the wires into both pins.

`kicad-cli sch export svg` puts numbers on it. Its invisible `<text>` elements carry each
string's anchor point, and the drawn strokes give the body — on this board's D1, at
(330.2, 74.93) turned to 90°, the symbol axis is `x = 330.2` with stroke vertices at
`y = 71.12, 73.66, 76.2, 78.74`: pin, body, body, pin.

| | `D1` anchor | `Green` anchor | distance from the axis |
|---|---|---|---|
| After `rotate_schematic_component` | (330.20, 73.02) | (330.20, 78.11) | **0.00mm — on it** |
| After `reset_schematic_field_positions` | (327.66, 75.56) | (332.74, 75.56) | 2.54mm |

At `font-size` 1.6933 and `textLength` 2.6827, `D1` spans 71.68..74.36 along that axis:
the upper pin and the top of the body. `Green`, at `textLength` 5.5856, spans
75.32..80.90 — the rest of the body and the whole lower pin.

**Not general to rotation.** Every other rotated symbol on the sheet is placed correctly,
because `add_schematic_component` has carried its anchors through the placement transform
since [#101]. On this board's 8 rotated symbols, only the two LEDs are wrong, and they are
the two that were placed flat and turned afterwards. `Device:R`'s anchor is off to the
side, so the same defect on a resistor reads as a slightly odd offset rather than a
collision; `Device:LED` anchors above and below, where a 90° body goes.

**Cause:** `Symbol::set_rotation`
(`crates/konnect-schematic-editor/src/schematic/symbol.rs`) is two lines — it assigns
`at.rotation` and returns. Property `(at …)` coordinates are absolute in `.kicad_sch`,
which the sibling `translate` accounts for explicitly, moving every field with the body.
`handle_rotate_schematic_component` is the only caller, so the whole exposure is that one
tool. Placement is unaffected: it assigns `at.rotation` directly and builds its fields
from the library anchors afterwards.

**Fix:** rotate each field's position about the symbol origin, reversing the sense when
the body is mirrored — a placement rotates before it mirrors, and a reflection reverses
any rotation conjugated by it. The stored field *angle* must be left alone: KiCad adds
the symbol's rotation when it draws, which `field_at` already documents and pins with a
rendered test. `reset_schematic_field_positions` repairs a sheet already written, and did
before this was found.

[#101]: https://github.com/mixelpixx/Konnect/issues/101
[#490]: https://github.com/mixelpixx/Konnect/issues/490
[#612]: https://github.com/mixelpixx/Konnect/issues/612

[#207]: https://github.com/mixelpixx/Konnect/pull/207
[#538]: https://github.com/mixelpixx/Konnect/issues/538
[#541]: https://github.com/mixelpixx/Konnect/issues/541
[#542]: https://github.com/mixelpixx/Konnect/issues/542
[kicad#25582]: https://gitlab.com/kicad/code/kicad/-/issues/25582
[#555]: https://github.com/mixelpixx/Konnect/pull/555
[#614]: https://github.com/mixelpixx/Konnect/pull/614

---

## Fixed or filed elsewhere since the last pass

- **`batch_delete` on a duplicate designator.** Used to delete one of two components
  sharing a reference and report success. Now refuses: with two `R9` symbols on a sheet,
  `batch_delete(references=["R9"])` returns `ambiguous_target` carrying both UUIDs and
  deletes nothing. Retested 2026-09-13.
- **`get_footprint_info` promises "pad layout" and returns only `pad_count`.** Filed as
  [#409], which covered `get_component_pads` too and recorded the shorts it caused, and
  merged on 2026-09-18 as [#603] — both readers now carry pad size, shape, drill and
  layers. Not re-measured here.
- **`batch_connect_to_net` writes no wire while `connect_to_net` writes one.** Measured
  again: single adds wire +1 / label +1, batch adds wire +0 / label +1. Both tool
  descriptions are now accurate about it — batch says "adding net labels at each pin
  endpoint" and promises no stub — so the documentation mismatch is gone and only the
  API asymmetry remains. Worth knowing before comparing wire counts between two sheets;
  not filed.

[#409]: https://github.com/mixelpixx/Konnect/issues/409
[#603]: https://github.com/mixelpixx/Konnect/pull/603

---

## Notes

Observations, not defects. Recorded against the board above; not re-measured on
2026-09-13 unless a note says so.

- `rotate_component` takes `rotation`, not `angle`. Not a bug — the structured
  `invalid_argument` error named the right field and recovery took one retry. Worth
  keeping as an example of the error taxonomy paying off.
- Power-symbol rotation is consistent and worth writing down: rotation adds to the
  pin's `orientation_degrees`, so `GND` at 90 and `+3V3` at 270 both put the body to
  the right of a wire arriving from the left. `get_schematic_pin_locations` on the
  `#PWR` reference confirms it without a render.
- Parallel MCP calls against one schematic were safe in practice — 10 concurrent
  `add_power_symbol` calls serialised cleanly with no revision conflict and no lost
  writes. Held again on 2026-09-15 across a whole rebuild: batches of up to 7 concurrent
  `add_power_symbol` and `batch_connect_to_net` calls on one sheet lost nothing, and the
  `#PWR` auto-numbering handed out `#PWR001`..`#PWR033` with no collision and no gap
  across concurrent callers. Verified by counting labels in the file, not by trusting
  the responses.
- `add_power_symbol` accepts `PWR_FLAG` and auto-numbers it into the same `#PWR` sequence,
  so a flag and the rail symbol it flags can be stamped at one point — the flag's body
  draws up, the rail's down, and `check_schematic_overlaps` reports the pair as an
  overlap. That report is correct and the placement is intentional; it is the one overlap
  on this sheet.
- The IPC-first board readers [#207] merged cover `get_board_info`, `get_component_pads`
  and `get_pad_position`. The file scans in `pcb_routing` still parse the file while the
  writers beside them use IPC. Each needs its own IPC mapping and some have no API
  equivalent, so it is a follow-up rather than a regression. [#153] has since rewritten
  `get_layer_list` to read the stackup by shape, which makes that one a clean starting
  point — see entry 3. (The note used to name `list_zones` alongside it; no zone-reading
  tool exists any more — `add_zone` and `refill_zones` are both writers.)

[#153]: https://github.com/mixelpixx/Konnect/pull/153
