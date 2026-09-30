# KiCad `video` demo sheet

`modul.kicad_sch` is copied byte for byte from the `video` demo distributed with
KiCad 10.0.6 for Fedora (`/usr/share/kicad/demos/video/modul.kicad_sch`, package
`kicad-10.0.6-1.fc44`). The file identifies `eeschema` 9.0 as its generator. It
is a KiCad demo file, not a user design. KiCad licenses its demos under CC BY-SA
4.0.

| File | SHA-256 |
| --- | --- |
| `modul.kicad_sch` | `59f6d8152af769371bd35a08dc11c1bdf9c3b4e56ab079f389a0584d1a4574c7` |

## Why it is here

KiCad drew it, not Konnect, so it can check the order in which a placement
applies rotation and mirror (#613). `L2` is a `video_schlib:INDUCTOR` at
(109.22, 38.1), rotated 270° and `(mirror x)`. Its library anchors the Value at
(2.54, 0), and eeschema wrote the Value at (109.22, 35.56), 2.54mm above the
origin:

| Order | Value offset | Matches the file |
|---|---|---|
| rotate, then mirror | (0, −2.54) | yes |
| mirror, then rotate | (0, +2.54) | no |

Clearing the mirror therefore has to move the Value to (109.22, 40.64). A
reflection taken in the symbol's own frame would move X instead and leave it
where it is.

Across the 25 demo sheets in KiCad 10.0.6 that place a symbol mirrored at 90° or
270°, 206 Reference and Value fields sit on a library anchor that the two orders
place differently. All 206 match rotate-then-mirror, and none match
mirror-then-rotate. This sheet is the smallest of them.

Tests only read it. The checked-in bytes stay unchanged.
