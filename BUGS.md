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

On 2026-09-19 this branch was rebased onto `main` @ c5b674e and the three remaining
entries were re-read against that tree rather than re-run. Entry 1's create path is
byte-identical — `write_new_atomic_unlocked` still builds its scratch file with
`tempfile::Builder`, and `persist_journal` still shares it — and so is entry 2's
`sch erc --format json` invocation and `parse_item_pos`, so those measurements still
describe this tree. The board-source entry was re-counted rather than re-read, because `main` has
grown IPC handlers since it was written: both readers it names are still file-only and
the imbalance around them is larger, not smaller. The tracker was re-checked the same
day — [#538], [#541] and [#542] are all still open. So is [#613], the same defect in
`set_mirror`, but it waits on the mirror tool in [#450] rather than on anything here;
[#450] is claimed and in progress.

On 2026-09-21 this branch was rebased onto `main` @ cdcafa8 and the tracker was
re-checked. Nothing here is fixed yet, but the board-source entry has a fix in review: it is the commit
in this branch, open upstream as [#656], where the maintainer's
CHANGES_REQUESTED of 2026-09-20 is answered on that head with all ten required checks
green. That review also produced the first live measurement of that entry's divergence, and
a fourth entry — [#673], filed the same day out of a Windows observation in it. Entry 1
turns out never to have gone unclaimed: [#538] has carried [#555]'s author as assignee
since 2026-09-13, from before that PR was opened, and kept it through the closure;
that author asked in [#555] to be unassigned, and the label and assignee are both still
in place, so [#674] was opened against a claim only a maintainer can clear. Entry 2 and [kicad#25582] beneath it are unchanged, and
[#450], which [#613] waits on, has had no activity since 2026-09-08.

On 2026-09-22 entry 2 moved: [kicad#25582] was closed upstream the previous day with the
fix committed for KiCad 10.0.7, which supplied the version boundary the compensation
needed, and entry 2's fix is now the ERC commit in this branch. It was measured on this
machine's KiCad 10.0.6 rather than re-argued — `kicad-cli sch erc` run twice over the
committed `single_pin_nets` hierarchy, `--format json` against `--format report`, all
twelve items exactly ÷100 apart — and that pair is the fixture it ships with. The
tracker was re-checked the same day: [#541] is still open, unassigned, `P1`,
`status:design-needed`, with the maintainer's implementation direction of 2026-09-13
still the latest word on it.

On 2026-09-25 this branch was rebased onto `main` @ 9de9d0d and the tracker was
re-checked. The board-source entry is gone: [#656] merged on 2026-09-23 and closed
[#542], so the entries after it moved up by one. [#675] came back twice as
CHANGES_REQUESTED. The first review is answered with a real 10.0-branch capture. The
second found a gap in the version gate. The maintainer decided the scope on [#541] the
same day, and the branch build is now pinned as a known limitation, recorded under
entry 2. [#673] was confirmed on Windows by the maintainer, with
a second site. One new entry was added: [#690], filed the same day out of the [#677]
review.

On 2026-09-26 this branch was rebuilt on `main` @ 86d102a, after [#683] merged.
[#674] was approved and moved to `status:ready-to-merge`. [#675] was approved, and then
sent back for a stale base. It was rebuilt as one commit on 86d102a.

An entry is deleted once its fix is merged, and the rest are renumbered, so nothing
outside this file cites one by number.

The ten entries this file used to carry are all merged upstream — IPC socket
auto-detection as [#382], `find_single_pin_nets` counting pins as [#402], the
board-not-open classification as [#407], the two-named-net audit as [#513], `run_erc`
writing into the project directory as [#570], which moved the report to a private
temporary directory removed even when the run fails, `trace_from_point` omitting pins
and junctions as [#588], the unbounded stale-target refusal as [#605], rotation
never reconciling its junctions as [#642] — that one reconstructed by the maintainer
onto the pin-owned no-connect contract from [#641], with [#627] and [#632] closed as
superseded copies — a turn leaving its field text behind as [#614], merged on
2026-09-19 once its regressions moved onto a KiCad-authored fixture, and
`get_layer_list` and `get_netclasses` answering from the saved file as [#656], merged
on 2026-09-23 with a `board_source` argument and per-field `sources`. Socket discovery
has since been narrowed further by [#505], which replaced the connect probe with a
metadata read.

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
unanswered. The maintainer moved [#538]'s assignment from [#555]'s author to me on
2026-09-21. That review settled the shape the fix has to take, recorded under
**Fix** below. Reproduced from scratch on 2026-09-15: a `create_project` into an empty
directory under `umask 022` again produced
`.kicad_pro`/`.kicad_sch`/`.kicad_pcb` and `sym-lib-table` at `0600`, with
`create_symbol`'s `.kicad_sym` at `0644` beside them as the control. A fix is now in
review: the file-mode commit in this branch, open upstream as [#674] on 2026-09-21,
which answers that checklist in full. It was moved to
`status:waiting-on-dependency` on 2026-09-22, when the maintainer put the merge train in
order. On 2026-09-25 the maintainer asked for it to be rebuilt on current `main`. It was
rebuilt unchanged as `8079ae2`, all ten required checks are green, and it was approved
on 2026-09-26, `status:ready-to-merge`.

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
2026-09-13, retested on the 10.0.6 upgrade. Root cause is upstream and filed there as
[kicad#25582] on 2026-09-17, so the fix here is compensation, not correction — and it
now has an end date: upstream **fixed it on 2026-09-21**, one line in
`eeschema/erc/erc_report.cpp` (`pcbIUScale` → `schIUScale`), landed on master as
`407f6293` and cherry-picked to the 10.0 branch as `6d8e1fe`. The issue closed the same
day, `status::fix-committed`, milestone **10.0.7** (due 2026-09-30). That is the version
boundary this entry was missing, so the compensation is now **in review** as [#675] —
the ERC commit in this branch. The scope decision described under **A branch build
between releases** below is made. `74912bf` was approved on 2026-09-26, then sent back
because its base was stale. It was rebuilt as one commit, `751bc2d`, on 86d102a, and is
back for exact-head review.
Reproduced independently on a
second sheet on 2026-09-15: a symbol `add_power_symbol` had just placed at
`(120.65, 219.71)` came back from `run_erc` at `(1.2065, 2.1971)`, and the raw
`kicad-cli` report behind it says
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

**Fix, as taken:** scale on the way out of `run_erc` alone, gated on the version, with
the third state [#541]'s review asked for. The gate reads the `kicad_version` the report
itself carries — the binary that wrote those numbers, rather than whatever `kicad-cli
--version` answers afterwards. Below 10.0.7 the coordinates are multiplied back; from
10.0.7 they are passed through; a version that cannot be placed on either side of the
fix withholds them, keeping KiCad's raw numbers under
`kicad_reported_x`/`kicad_reported_y` so nothing is lost and nothing is offered as a
location. The response states which of the three happened and why, because the same
tool now answers differently on two machines. `run_drc` is untouched — it never carried
the defect — and a test in the DRC module says so.

Two things surfaced while building it, neither in the upstream report:

- **The measurement inside KiCad's own violation text is scaled too.** The same units
  provider formats the prose, so an affected version says `Horizontal Wire, length
  0.1270 mm` for a 12.70 mm wire. Konnect corrects coordinates and does not rewrite
  KiCad's prose, so that divergence survives; the response's `coordinates.reason` says
  so.
- **A development build cannot be classified from its version.** KiCad's dev branches
  carry minor 99, and the fix reached master and 10.0 on the same day, so `10.99.0`
  alone cannot say whether a nightly predates it. Those withhold rather than guess.

**A branch build between releases.** The second review of [#675], on 2026-09-23, found
a case the version gate gets wrong. A KiCad built from the 10.0 branch after `6d8e1fe`
writes correct coordinates but still stamps its reports `10.0.6`.
`KICAD_SEMANTIC_VERSION` stays at the last release until the release commit bumps it. So
the gate multiplies correct coordinates by 100. The report carries nothing else that
could separate the two: an unmodified branch capture is byte-identical to a
release-stamped one apart from `date` and `kicad_version`. Every released binary is
classified correctly.

On 2026-09-23 I asked on [#541] for these builds to be excluded from the supported
contract. The reasons given there: any version-gated behaviour has the same limit, the
window closes when 10.0.7 ships (due 2026-09-30), and both alternatives cost every
10.0.6 user. The alternatives are a second `--format report` run cross-checked against
the JSON, or no coordinates at all on 10.0.6. If the exclusion is accepted, the
unmodified capture goes in as a pinned regression fixture. If not, the 10.0.6 gate
becomes a content check.

The maintainer accepted the exclusion on 2026-09-25. Released builds are the supported
contract, and a branch build between releases is outside it. The unmodified capture is
now committed as `erc_coordinate_scale_kicad10_branch.json`. A test pins what happens to
it: the coordinates are corrected, which leaves every item 100× the text-report oracle.
Unknown and development versions still withhold.

The two options not taken: omitting the coordinate outright, which the version boundary
now makes unnecessary for a released KiCad; and reading `--format report` instead, which
is correct on every version but trades a schema for text parsing, and would still need
the JSON report for everything else in the violation.

## 3. Windows: open-board lists print verbatim `\\?\` paths

**Status:** filed as [#673] on 2026-09-20, out of the [#656] review; unassigned,
`status:ready-for-work`. **Not reproduced by me.** On Fedora `canonicalize` returns a
plain absolute path and the defect cannot appear. The maintainer confirmed it on Windows
11 / KiCad 10.0.5 on 2026-09-23: the `wrong_document` rendering is exactly as quoted
below.

**Severity:** low — display only. Identity is unaffected: the comparison runs on the
canonical forms and is correct.

A board-identity error prints the boards KiCad holds in canonical form and the requested
board as the caller typed it, so one message can show one directory in two spellings:

```
requested board 'C:\Users\me\proj\board.kicad_pcb' is not open in KiCad
  (open boards: \\?\C:\Users\me\proj\other.kicad_pcb)
```

These errors exist so a caller can see which board KiCad actually holds and retry
against it, which is the one use a `\\?\` path is bad for.

**Wider than the one message.** `board_document_label` in `konnect-ipc/src/client.rs` is
the shared source, so `wrong_document`'s `open_documents`, `ambiguous_target`'s
`candidates`, `stale_target`'s `previously_bound` and `ambiguous_open_board`'s reason all
carry it, and `requested` is raw in every one of them. The Windows confirmation found a
second site with the same cause: the sibling-lock path in `source_evidence.detail`, and
the refusal text built from it, prints as `\\?\C:\…\~<name>.kicad_pcb.lck`.

**Cause:** `std::fs::canonicalize` returns a verbatim path on Windows and a plain
absolute path on Unix, and `comparable_identity` feeds the label. Older than [#656]:
`comparable_identity` landed on 2026-08-31 and nothing in the workspace handles the
prefix.

**Fix:** one display helper covering both sites. Strip at the display boundary only — canonical paths must keep reaching
`select_requested_board`'s matching, or the identity defect [#407] fixed comes back. And
`\\?\UNC\` is not strippable as a prefix: `\\?\UNC\server\share\x` has to render as
`\\server\share\x`, so a naive `strip_prefix` corrupts every network path. Worth a matrix
over a drive path, a UNC path, a path that does not exist — that one takes
`comparable_identity`'s lexical branch and has no prefix to begin with — and a Unix path
that must pass through untouched.

## 4. `score_placement` resolves its live board twice

**Status:** filed as [#690] on 2026-09-25, out of the [#677] review;
`status:waiting-on-dependency` on [#677]. Code trace only; not measured.

**Severity:** low — one extra round trip per live call. It never reads the wrong board.

`score_placement` reads the live board through `with_board_ipc_classified`, which binds
the requested board first. The closure then calls `save_document_to_string()`, which
takes no document and resolves it again through `get_board_document()`. That is a second
`GetOpenDocuments`. `get_board_document()` re-selects the bound board and never falls
back to KiCad's first one. If the open documents change between the two lookups, the call
fails with `StaleDocument`.

It is the pattern [#676] removes from six other reads. [#676] missed it because the
extra lookup comes from the argument-less save, not from `find_open_board`.

**Fix:** after [#677], the closure is handed the resolved document and this site ignores
it. Use `save_document_to_string_in(document)`, pinned by [#677]'s counting mock at one
`GetOpenDocuments` per call. Depends on [#677].

The same change should narrow the docstring on `with_bound_board_ipc_classified`
(`tools/mod.rs`). It says a method with no document argument "addresses whichever board
KiCad opened first", which is not true once the helper has bound the board. The
maintainer asked for that on [#676] on 2026-09-25.

[#207]: https://github.com/mixelpixx/Konnect/pull/207
[#538]: https://github.com/mixelpixx/Konnect/issues/538
[#541]: https://github.com/mixelpixx/Konnect/issues/541
[#542]: https://github.com/mixelpixx/Konnect/issues/542
[#676]: https://github.com/mixelpixx/Konnect/issues/676
[#677]: https://github.com/mixelpixx/Konnect/pull/677
[#690]: https://github.com/mixelpixx/Konnect/issues/690
[kicad#25582]: https://gitlab.com/kicad/code/kicad/-/issues/25582
[#555]: https://github.com/mixelpixx/Konnect/pull/555
[#450]: https://github.com/mixelpixx/Konnect/issues/450
[#613]: https://github.com/mixelpixx/Konnect/issues/613
[#614]: https://github.com/mixelpixx/Konnect/pull/614
[#656]: https://github.com/mixelpixx/Konnect/pull/656
[#673]: https://github.com/mixelpixx/Konnect/issues/673
[#674]: https://github.com/mixelpixx/Konnect/pull/674
[#675]: https://github.com/mixelpixx/Konnect/pull/675
[#683]: https://github.com/mixelpixx/Konnect/pull/683

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
  `get_layer_list` to read the stackup by shape, and [#656] has since given it and
  `get_netclasses` an IPC path. (The note used to name `list_zones` alongside it; no zone-reading
  tool exists any more — `add_zone` and `refill_zones` are both writers.)

[#153]: https://github.com/mixelpixx/Konnect/pull/153
