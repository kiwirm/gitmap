# Gitmap

A deterministic, version-control-friendly file format for orienteering maps. This
repository is the specification; it contains no implementation.

Gitmap stores a map as a small directory of text files designed to live in Git:
one file per concern, most of them NDJSON streams. It is a canonical
representation. The same map serialises to the same bytes regardless of which
authoring format it came from (OCAD `.ocd`, OpenOrienteering `.xmap`/`.omap`), so
`git diff` shows real changes rather than artefacts of the import path. A
cross-format update (re-export from OCAD over an OMap-sourced base) reads as a
clean diff, not a 100%-changed one.

This document specifies **version 4**. Its reference implementation is
[panmap](https://github.com/kiwirm/panmap) (0.7.0 and later). Object and colour
geometry is byte-identical across source formats, and a symbol's draw `layers`
reduce to one shape. The residual symbol differences are genuine source/OCAD-format
limits, not the format itself (see [Known gaps](#known-gaps)).

This README is the normative specification. Machine-readable schemas live in
[`schema/`](./schema) (JSON Schema 2020-12); the prose is normative and the schemas
validate structure. A complete example is in
[`examples/minimal.gitmap`](./examples/minimal.gitmap). Where this document and
panmap disagree, panmap's behaviour is the intended one and this document is the
bug.

## Goals and non-goals

Goals: reviewable diffs with ordinary Git tooling; no editor churn (undo stacks,
view/zoom, local template paths); no source-format ordering noise; enough data to
export losslessly back to a native format; semantic visual diffs, and later merges.

Non-goals: replacing Git; live collaboration; optimising for hand-authored files
over deterministic machine output.

## Package layout

A `.gitmap` package is a directory of exactly nine files. Every file is
REQUIRED. "None" is written as an empty file (NDJSON and `notes.txt`) or as `{}`
followed by a newline (JSON). A reader MUST reject a package that is missing any
of them.

```text
map.gitmap/
  manifest.json        # {"format":"gitmap","version":4}, nothing else
  parts.ndjson         # map parts, one {id, name} per line
  georeferencing.json  # the CRS object, or {}
  extensions.json      # namespaced tool data, or {}
  notes.txt            # map notes, plain text
  colors.ndjson        # drawing palette, one colour per line, topmost first
  spot-colors.ndjson   # print separations, one per line
  symbols.ndjson       # one symbol per line
  objects.ndjson       # one object per line
```

Filenames are fixed; there is no index of files. Each concern has its own file so
that a change to one (a colour inserted, a notes line edited) diffs as a change to
that file only.

A gitmap package is map content only. Editor view/zoom state, print settings and
templates (background rasters) are not part of the format: they churn on every
open or on local paths, and consumers lack the referenced files. If an editor or
pipeline drops local, non-content files beside a package (for example a
`private/` directory or generated `svg`/`geojson` exports), they aren't part of
the format and shouldn't be committed. A reader ignores files outside the nine.

## Encoding and determinism (normative)

All files are UTF-8 with LF line endings.

**NDJSON** (`*.ndjson`) is canonical: one compact JSON record per line, every line
(including the last) ends in `\n`, and there are no blank lines. Zero records is
an empty file, not a lone newline. A reader MUST reject a non-empty NDJSON file
that has a blank line or lacks a final newline.

**JSON** (`manifest.json`, `georeferencing.json`, `extensions.json`) is
pretty-printed with two-space indentation and one trailing `\n`. An empty object
is `{}\n`. `georeferencing.json` and `extensions.json` MUST be JSON objects (not
arrays or scalars).

**Numbers.** Coordinates are in map units (1 unit = 0.01 mm on paper), y-down.
Coordinates and non-integer symbol scalars are rounded to 3 decimal places;
integers are written as integers. Units and precision are fixed by this spec; the
manifest does not declare them.

A writer MUST:

- sort `colors.ndjson` in stacking order (topmost first, ties by `id`);
  `symbols.ndjson` by `code` then `id`; `objects.ndjson` by `partId`, then
  `symbolId`, then geometry (the coordinate sequence, then any hole rings), then
  `text`, then `rotation`. `parts.ndjson` and `spot-colors.ndjson` keep source
  order;
- omit optional fields that equal their default (`rotation:0`, `text:""`,
  `hAlign:"left"`, `vAlign:"baseline"`, an all-default `pattern`, false booleans)
  and fields whose value is undefined;
- emit no timestamps, derived counts, bounds, or local absolute paths.

String comparisons in these sort keys, and the alphabetical fallback in key
ordering below, follow ECMAScript `String.prototype.localeCompare` with no locale
argument, as panmap runs it (Unicode collation, root/`en` order). This is not
code-point order: case is a late tie-break, and punctuation such as `_` sorts
before digits and letters, so `sym_101_v2` sorts before `sym_1010`. Numbers
compare numerically.

### Key order

Within every JSON value, at every depth (records, nested layer objects,
`georeferencing.json` and `extensions.json` included), keys follow a fixed
priority order; any key not listed sorts after all listed keys, alphabetically.
Keys whose value is undefined are dropped.

```text
format, version, units, precision, parts, id, order, code, name, type, hidden,
visible, locked, partId, part, coordinates, holes, text, rotation, bounds, rgb,
fontSize, textSymbol, layers, colorId, color, symbolId, width, height, radius,
fontFamily, angle, spacing, lineWidth, opacity, elements, source, metadata
```

The list spans all record types; a record uses only the keys that apply (an object
has no `rgb`, a colour no `coordinates`). `units`, `precision` and `parts` are no
longer manifest fields in version 4 but keep their rank, which still applies to
any nested key of that name (for example inside `extensions.json`). `symbolId`
ranks after `color`, so an object emits `…partId, coordinates, …, symbolId`, and
a colour emits `id, order, name, rgb, opacity, cmyk`. Byte-identical output, the
conformance bar, depends on this order.

### Cross-format rules

The rules that make two source formats agree:

1. Coordinates are y-down. OCAD is y-up, so its coordinates are negated on read;
   everything ends in visual (y-down) space. Symbol icon-primitive (`element`)
   coordinates are symbol-internal and stay y-up.
2. Stored ordinals are canonical, not raw source ids. An object's `order` is its
   dense z-rank. A colour's `order` is its source palette number (OCAD colour
   number, Mapper priority), kept for round-trips; colour stacking is the line
   order of `colors.ndjson`, not a stored field. Symbols store no `order`: it was
   the code-sorted line position (derivable), and a stored index diverged whenever
   the two exports carried different symbol sets.
3. Objects carry no stored id; their geometry is their identity. `objects.ndjson`
   sorts by `(partId, symbolId, coordinate sequence, holes, text, rotation)`, so
   the same object lands in the same place from either format. Sorting on the
   coordinates rather than a hash of them keeps a small edit local: a moved object
   stays near its neighbours instead of jumping to a random line. Recognising a
   move across revisions is the differ's job (nearest-match on geometry), not a
   stored id's.
4. Symbol codes are canonicalised. A dotted numeric code is parsed as
   `main * 1000 + sub` (each further group a two-digit slot of `sub`) and
   re-emitted, so OCAD's variant-zero suffix collapses (`101.0` becomes `101`),
   `204.01` becomes `204.1`, and a genuinely distinct code like `204.1.0` is kept.
   Non-numeric codes pass through. A symbol's `id` derives from its canonical code,
   so an object's `symbolId` lines up across formats.
5. Values snap to OCAD's grid. Rotation (degrees) rounds to 0.1°; CMYK to whole
   percent (2 dp of a 0..1 fraction); text `fontSize` to OCAD's tenths-of-a-point
   grid; stroke `width`, `startOffset`, `endOffset`, `endLength`, `segmentLength`,
   `pointedCapLength` and pattern `height` to whole units; point radii to half
   units; and point-symbol element coordinates to a 5-unit (0.05 mm) grid. OCAD
   can't store finer (or the two exporters round half-units in different
   directions), so without the snap the formats disagree in the last digit. Other
   symbol scalars round to 3 dp.
6. Holes are structure, not a flag. An area's outer ring is `coordinates`; each
   hole is its own ring in `holes`. Both formats instead flag the boundary on one
   vertex (OCAD on the hole's first coord, OMap on the previous ring's last, its
   `isHolePoint`); gitmap drops the flag and stores the rings, so a diff shows a
   whole hole, not one toggled bit.
7. Coordinate flags are semantic, not format bytes. A vertex is `[x, y]` or
   `[x, y, {…}]`, where the flags say what the point is: `control` or `dash`. The
   OCAD `xFlags`/`yFlags` bytes and the OMap curve byte aren't stored. Bézier
   control points come in pairs, so a reader recovers OCAD's cp1/cp2 (0x01/0x02)
   from their order. OCAD's corner (`yFlags` 0x01) and dash (0x08) points both
   become `dash`: Mapper has a single marked-vertex flag (DashPoint) for both, so
   the distinction can't live in the canonical form. An OCD writer re-derives
   which byte to write from the symbol, as Mapper's own OCD export does.
8. Enumerations are words, not source byte values. `capStyle`
   (`flat`/`round`/`square`/`pointed`) and `joinStyle` (`bevel`/`miter`/`round`)
   on strokes and `hAlign`/`vAlign` on text read as names rather than a source
   format's integer, so a diff is legible and the stored form doesn't carry OCAD's
   encoding. A colour reference is `colorId` and a coordinate list is
   `coordinates` everywhere, including inside symbol icon primitives.
9. Single-format fields are dropped. A field that only one source format can
   carry (for example Mapper's `minimumMidSymbolCount`, hatch `lineOffset`, fill
   `minArea`, OCAD's stroke `lineStyle` byte and text framing `fr*` fields) is not
   written, because keeping it would make the two exports of one map differ.

## Manifest: `manifest.json`

Schema: [`schema/gitmap.manifest.schema.json`](./schema/gitmap.manifest.schema.json).

```json
{
  "format": "gitmap",
  "version": 4
}
```

The manifest has exactly these two keys. A reader MUST reject a manifest whose
`format` is not `"gitmap"`, whose `version` is not `4`, or that has any other key.
The manifest fields of earlier versions (`units`, `precision`, `parts`,
`georeferencing`, `notes`, `extensions`, `spotColors`) are either fixed by this
spec or moved to their own files.

## Parts: `parts.ndjson`

Schema: [`schema/gitmap.part.schema.json`](./schema/gitmap.part.schema.json).

One `{ id, name }` record per line, in source order. Objects reference a part by
`partId`. A single-part map writes the default part:

```json
{"id":"part_main","name":"Main"}
```

Every object in a single-part map has `partId: "part_main"`. A map with more than
one part lists every part. A reader treats a one-line `parts.ndjson` as the
implicit default part.

## Georeferencing: `georeferencing.json`

Schema: [`schema/gitmap.georeferencing.schema.json`](./schema/gitmap.georeferencing.schema.json).

The map's coordinate reference system as one JSON object, pretty-printed with the
stable key order (so its keys come out alphabetically, with any nested `id`
first). A map with no georeferencing writes `{}`.

```json
{
  "declination": 23.91,
  "geographic": {
    "id": "Geographic coordinates",
    "refPointDeg": {
      "lat": -43.43347842,
      "lon": 172.69110272
    },
    "spec": {
      "language": "PROJ.4",
      "value": "+proj=latlong +datum=WGS84"
    }
  },
  "grivation": 23.7,
  "projected": {
    "id": "EPSG",
    "parameter": "2193",
    "refPoint": {
      "x": 1575000,
      "y": 5191000
    },
    "spec": {
      "language": "PROJ.4",
      "value": "+init=epsg:2193"
    }
  },
  "scale": 15000
}
```

`scale` is the map scale denominator; `grivation` and `declination` are degrees.
For an OCAD source, `geographic` and `declination` are derived from the projection.
`auxiliaryScaleFactor` is the one CRS field OCAD cannot represent and may be
absent (see [Known gaps](#known-gaps)).

## Extensions: `extensions.json`

An object of namespaced key/value data a tool attaches to the map (survey
provenance, pipeline bookkeeping, editor state worth versioning), or `{}` when
there is none. A writer leaves the values untouched apart from the stable key
order; a reader that doesn't recognise a key preserves it rather than dropping it.
Namespace keys (reverse-DNS, or a tool prefix) so two tools don't collide.

```json
{
  "com.example.survey": {
    "datum": "NZGD2000",
    "gps": "trimble-r12"
  }
}
```

## Notes: `notes.txt`

The source map's free-text notes field as plain UTF-8 text, so editing one line
of the notes diffs as one line. A writer canonicalises the text so that both
source formats agree:

- CRLF and lone CR become LF (OCAD stores CRLF);
- trailing spaces and tabs are removed from every line (OCAD trims them on save
  where Mapper keeps them);
- trailing newlines at the end are removed;
- the result is followed by exactly one `\n`.

A map with no notes (or notes that are empty after canonicalisation) writes an
empty file. A reader MUST reject a non-empty `notes.txt` that doesn't end in `\n`;
the notes text is the file without that final newline.

### Preserving extensions through a native format

OCAD and OMap have no structured metadata slot, only one free-text notes field.
A writer exporting to either format serialises the map's extensions into that
notes field as a fenced block below the notes text, and a reader of either format
parses the block back out into `extensions`. Inside a gitmap package the block
therefore doesn't appear in `notes.txt`; the data is in `extensions.json`.

```text
Surveyed Feb 2026; contours from 2019 LiDAR.

--- gitmap-extensions v1 ---
{
  "com.example.survey": {
    "datum": "NZGD2000",
    "gps": "trimble-r12"
  }
}
--- end gitmap-extensions ---
```

- The open fence is `--- gitmap-extensions v<N> ---`, where `N` is the block
  version (currently `1`); the close fence is `--- end gitmap-extensions ---`. Each
  is a whole line.
- The body is the extensions object as JSON with two-space indentation and keys
  sorted at every depth in plain code-unit order (not the gitmap key-priority order
  used in `extensions.json`).
- Writing: the block is emitted only when `extensions` has at least one key, so a
  notes field with no extensions round-trips byte-for-byte. When it is emitted, any
  existing block is removed from the notes text and trailing whitespace trimmed;
  the block then follows after one blank line, or stands alone if there is no
  other text. OCAD's notes field is written with CRLF line endings.
- Reading: the first open-fence line is located. An unknown fence version is not
  an error: the notes field is kept verbatim (the block rides through as text),
  the same preserve-don't-discard rule as an unrecognised key. For a known
  version, a second open fence before the close fence (nesting) or a missing close
  fence is corruption and MUST be rejected, and so is a body that isn't a JSON
  object (an empty body is `{}`). The notes text is what precedes the block
  (trailing newlines removed) and what follows it (leading newlines removed),
  joined by one blank line when both are non-empty.
- The marker is plain ASCII, so it survives OMap's XML escaping and OCAD's text
  field unchanged.

## Colors: `colors.ndjson`

Schema: [`schema/gitmap.color.schema.json`](./schema/gitmap.color.schema.json).

The drawing palette, one colour per line. **Line order is the stacking order:**
the first line is the topmost colour, as in OCAD's colour table and Mapper's
priority list (ties, which a valid source doesn't produce, break by `id`). There
is no stored stacking index, so inserting a colour adds one line instead of
renumbering every colour below it. A reader assigns stacking from line position.

```json
{"id":"color_black","order":1,"name":"Black","rgb":[0,0,0],"opacity":1,"cmyk":[0,0,0,1]}
{"id":"color_brown_100","order":6,"name":"Brown 100%","rgb":[148,96,40],"opacity":1,"cmyk":[0,0.35,0.75,0.42]}
```

- `id` is `color_<slug(name)>` (`color_unnamed` when the slug is empty). It derives
  from the name only, so an OCAD colour renumbering or a Mapper priority shift
  doesn't change any id or cascade into every symbol's `colorId`. Names aren't
  unique (ISOM sets repeat "Black" or "Purple" at different levels, and "Purple"
  and "PURPLE" slug alike), so colours sharing a slug are ranked by stacking
  position (topmost first), then by `order`; the first keeps the bare id and the
  rest take `_2`, `_3`, and so on. Reordering only reshuffles suffixes within such
  a same-name group. Symbols reference colours by this `id`, never a number.
- `order` is the colour's source palette number (OCAD colour number; Mapper
  priority), kept so an export restores the source numbering. It is not part of
  the id and does not define stacking.
- `rgb` is the screen colour as an integer `[r, g, b]` triple (0..255).
- `cmyk` is the print/OCAD ink `[c, m, y, k]`, each 0..1, snapped to whole percent
  (2 dp). Both are stored so a gitmap-to-OCD export keeps ink density.
- `opacity` is 0..1.

`slug(s)`, used by every id, decomposes `s` to Unicode NFKD and removes combining
marks (U+0300 to U+036F) so accents don't become `_` (`"Rosé"` gives `rose`), trims
it, lowercases it, replaces each run of characters outside `[a-z0-9]` with a
single `_`, and trims leading and trailing `_`. So `"Brown 100%"` becomes
`brown_100`. All cross-references are id-keyed, so a conforming writer MUST slug
identically or the reference chain diverges.

## Spot colours: `spot-colors.ndjson`

Schema: [`schema/gitmap.spot-color.schema.json`](./schema/gitmap.spot-color.schema.json).

The map's print separations ("Process Black", "PMS 299 Blue", …), the named inks
it is separated onto when printed, one `{ name, cmyk, rgb }` per line in source
order (first seen). A separation is identified by its `name`; a repeated name is
listed once.

```json
{"name":"Process Black","rgb":[44,46,53],"cmyk":[0,0,0,1]}
```

`name` is required; `cmyk` (`[c, m, y, k]` in 0..1) and `rgb` (an integer
`[r, g, b]` triple, 0..255, the same shape as a palette colour's) are written when
the source has them.

Separations are deliberately not palette colours. No symbol draws in one, they
have no stacking position, and the two source formats disagree about whether a
separation also appears in the palette. OCAD keeps a dedicated table (parameter
string record 10). Mapper's xmap either appends a dedicated `<color>` per
separation (whose name equals its own `<namedcolor>`), or tags an existing palette
colour with a `<namedcolor>` and adds no record at all, depending on how the map
was authored. Folding separations into `colors.ndjson` would make the palette
differ between two exports of the same map, so Mapper's dedicated separation
colours are kept out of the palette and every separation is listed here instead.

## Symbols: `symbols.ndjson`

Schema: [`schema/gitmap.symbol.schema.json`](./schema/gitmap.symbol.schema.json).

Sorted by `code` then `id` (a symbol without a code sorts first). The `id` is
`sym_<slug(canonical code)>` (`sym_101`); a symbol with no code uses its source id
in place of the code. If two symbols share a canonical code, the first in source
order keeps the bare id and each later one takes a `_vN` suffix (`sym_522_v2`,
`sym_522_v3`).

```json
{"id":"sym_101","code":"101","name":"Contour","type":"line","layers":[{"type":"stroke","colorId":"color_brown_100","width":14,"capStyle":"flat","joinStyle":"bevel"}]}
```

A symbol carries `code` (canonical; omitted when empty), `name`, `type` (`point`,
`line`, `area`, `combined`, `text`, `line-text`, `unknown`), the
present-only-when-true flags `hidden`, `protected` and `rotatable` (the symbol
rotates with its object), and source-independent draw `layers`. Colour references
are stable `color_…` ids, lengths are in map units, and angles are in degrees.
Text typography is authoritative in the `text` layer's nested `text` object; there
is no top-level symbol `fontSize` and no per-layer `fontSize`/`fontFamily`. An area
symbol with a fill and a border (a border-symbol reference or a stroke) is
`combined`; a symbol labelled `combined` whose layers reduce to a plain line is
`line`.

Layer types:

| type | purpose |
| --- | --- |
| `fill` | area inner fill |
| `hatch-fill` | area hatch (`colorId`, `spacing`, `lineWidth`, `angle`°, `rotatable`) |
| `point-pattern-fill` | area point-pattern (xmap-native `pattern` record) |
| `stroke` | line's main stroke (`colorId`/`width`/`dash`/`capStyle`/`joinStyle`, optional `borders`, offsets, mid-symbol placement) |
| `point-fill` | point inner disc (`colorId`, `radius`) |
| `point-stroke` | point outer ring (`colorId`, `radius`, `width`) |
| `point-elements` | array of icon primitives (`elements`) |
| `text` | text typography (`fontFamily`, `fontSize` mm, weight, italic, spacing) |
| `line-elements` | line decoration element arrays (`primSym`/`cornerSym`/`startSym`/`endSymElements`) |

`layers` is in this canonical group order (the table's order), stable within a
group so the order of several strokes, which is the order they paint, is kept.
The two source formats emit the same layer types in different orders, so sorting
by role is what makes them agree.

Each source carries some concepts in its own shape: OCAD's `double-line` casing,
`structure-fill` pattern, and `border-symbol` reference; xmap's nested line
symbols. A writer reduces each to one type above (`stroke` with `borders`,
`point-pattern-fill`, an inlined border `stroke`, and `line-elements`), so output
has one shape per concept and those input dialects don't appear. A border
`shift` is always half its `width`. Field names use semantic strings (`capStyle`,
`joinStyle`), `colorId`, `coordinates`, and `radius` (an OCAD icon primitive's
diameter is halved). Element coordinates use the object tuple shape with the
flags `control`, `hole` and `dash` (`hole` remains because an icon primitive
isn't ring-split). The schema stays permissive only for opaque, format-native
substructures (the `pattern` record and nested `element.symbol` trees), which keep
their source field names (`color`, `coords`). The residual cross-format difference
is element-level content, not a type dialect, and affects only `symbols.ndjson`
(see [Known gaps](#known-gaps)).

## Objects: `objects.ndjson`

Schema: [`schema/gitmap.object.schema.json`](./schema/gitmap.object.schema.json).

One object per line, sorted by `partId`, then `symbolId`, then geometry (the
coordinate sequence compared vertex by vertex, x then y, with a prefix sorting
first; then the hole rings likewise), then `text`, then `rotation`. A symbol's
objects stay contiguous (a diff groups by symbol) and spatially near objects stay
near in the file. Vertex flags don't take part in the comparison. There is no
stored object `id`.

```json
{"order":0,"type":"line","partId":"part_main","coordinates":[[0,0],[10000,0,{"dash":true}],[10000,10000]],"symbolId":"sym_101"}
```

`order` is the object's 0-based index in render order (its z-rank; 0 is drawn
first); a reader sorts on it to restore render order. `partId` references
`parts.ndjson` (`part_main` in a single-part map). `symbolId` references
`symbols.ndjson`; a diff shows the `symbolId`, and its code is in
`symbols.ndjson`.

Coordinates are compact tuples (map units, y-down, 3 dp): a plain vertex is
`[x, y]`, a flagged vertex `[x, y, {…}]`. The flags object carries only semantic
meaning, and each field is present only when true:

- `control`: a Bézier control point. Control points come in pairs (cp1 then cp2),
  from which a reader recovers OCAD's `xFlags` 0x01/0x02 by position.
- `dash`: a marked vertex, which is OCAD's corner (`yFlags` 0x01) or dash
  (`yFlags` 0x08) point and Mapper's DashPoint.

An area with holes stores its outer boundary in `coordinates` and each interior
ring in `holes` (an array of coordinate rings); objects without holes omit `holes`.

Objects may also carry `text`; `rotation` (degrees, counter-clockwise, 0.1°
grid); `hidden: true`; `hAlign` (`center`/`right`; `left` is the omitted default)
and `vAlign` (`top`/`middle`/`bottom`; `baseline` is the omitted default);
`textBox` `{width, height}`; a per-object `pattern` `{rotation?, origin?}` override
(rotation in degrees, dropped when all-default); and `tag`/`tagType`, a free-form
per-object string with a type code (OCAD's object string, for example
course/control codes).

## Known gaps

The cross-format differences that remain are genuine source/OCAD-format limits,
not dialects; a writer can't bridge them without fabricating data:

- `joinStyle`: OCAD's `lineStyle` byte can't encode every cap+join pair, so
  Mapper's OCD export substitutes or drops the join. A writer normalises the pairs
  it can (a round cap forces a round join, flat cap with round join becomes miter,
  pointed cap with bevel join becomes round, miter collapses to bevel, and a dashed
  stroke drops a non-round join), but not every case converges.
- symbol `name`: a map's two Mapper exports can name the same symbol differently;
  converging would mean an arbitrary pick.
- double-line width: `line_width = dblWidth + borderWidth` holds for some casings
  but not all, a per-symbol ambiguity rather than a rule.
- decoration geometry: a few symbols draw a tick or edge a few units longer in one
  export than the other, non-systematic and genuinely different.
- xmap-only companion symbols: Mapper auto-generates variants (a "minimum size"
  point form, a "flat ends" line form) that share a parent's code and that its OCD
  export omits, a real symbol-set difference.
- `auxiliaryScaleFactor`: the one CRS field OCAD can't store, a "ground-true scale"
  choice it never records, absent for OCAD sources. A future opt-in could set it to
  `1 / gridScaleFactor` by the ground-true convention, a domain assumption rather
  than recovered data.

These touch only `symbols.ndjson` and that one CRS field, never object or colour
geometry.

## Versioning

The current version is `4`. Bump it and add a [`CHANGELOG.md`](./CHANGELOG.md)
entry on any change to the rules above (a change that alters the bytes a
conforming writer produces). Conformance means producing byte-identical output to
a conforming writer for the same map.

There are no fallbacks. A reader accepts exactly one version and rejects every
other, older ones included; it never reads an old package through a compatibility
path. A package written in an older version is migrated by re-converting the map
from its original source (the `.ocd`/`.omap` it came from) with a current writer,
not by upgrading the old gitmap in place. Readers likewise reject malformed input
(a missing file, extra manifest keys, a blank NDJSON line, a missing final
newline) rather than repairing it.

## Validating a package

The JSON Schemas are the machine-readable spec. Validate each JSON file, and each
line of each NDJSON file, against its schema (for example with
[ajv](https://ajv.js.org), draft 2020-12):

| file | schema |
| --- | --- |
| `manifest.json` | `gitmap.manifest.schema.json` |
| `parts.ndjson` (each line) | `gitmap.part.schema.json` |
| `georeferencing.json` | `gitmap.georeferencing.schema.json` |
| `colors.ndjson` (each line) | `gitmap.color.schema.json` |
| `spot-colors.ndjson` (each line) | `gitmap.spot-color.schema.json` |
| `symbols.ndjson` (each line) | `gitmap.symbol.schema.json` |
| `objects.ndjson` (each line) | `gitmap.object.schema.json` |

`extensions.json` is any JSON object and `notes.txt` is text. Their rules, the
file set, NDJSON framing, sort order and key order are checked against the prose,
not the schemas.

```js
import Ajv2020 from "ajv/dist/2020.js";
const ajv = new Ajv2020({ strict: false });
const validateObject = ajv.compile(require("./schema/gitmap.object.schema.json"));
// validate each line of objects.ndjson …
```

They can also generate types (`json-schema-to-typescript`) or drive editor
validation.

## Reference implementation

[panmap](https://github.com/kiwirm/panmap) reads and writes gitmap alongside OCAD
and OMap; its round-trip and cross-format identity tests are the executable spec.

## Contributing

A change that alters the bytes a conforming writer produces must update [`schema/`](./schema)
and the example in [`examples/minimal.gitmap`](./examples/minimal.gitmap). Bump `version` and add a
[`CHANGELOG.md`](./CHANGELOG.md) entry.

Validate any such change against panmap's convergence gate before proposing it: the
same map read from OCAD and from OMap must serialise to identical bytes. Open an
issue to discuss a format change first.

## License

MIT. See [`LICENSE`](./LICENSE).
