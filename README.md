# Gitmap

A deterministic, version-control-friendly file format for orienteering maps. This
repository is the specification; it contains no implementation.

Gitmap stores a map as a small directory of text files designed to live in Git:
one manifest plus a few NDJSON streams. It is a canonical representation. The same
map serialises to the same bytes regardless of which authoring format it came from
(OCAD `.ocd`, OpenOrienteering `.xmap`/`.omap`), so `git diff` shows real changes
rather than artefacts of the import path. A cross-format update (re-export from
OCAD over an OMap-sourced base) reads as a clean diff, not a 100%-changed one.

Version 1 is in active use, along with its reference implementation,
[panmap](https://github.com/kiwirm/panmap). Object and colour geometry is
byte-identical across source formats, and a symbol's draw `layers` reduce to one
shape. The residual symbol differences are genuine source/OCAD-format limits,
not the format itself (see [Known gaps](#known-gaps)).

This README is the normative specification. Machine-readable schemas live in
[`schema/`](./schema) (JSON Schema 2020-12); the prose is normative and the schemas
validate structure. A complete example is in
[`examples/minimal.gitmap`](./examples/minimal.gitmap).

## Goals and non-goals

Goals: reviewable diffs with ordinary Git tooling; no editor churn (undo stacks,
view/zoom, local template paths); no source-format ordering noise; enough data to
export losslessly back to a native format; semantic visual diffs, and later merges.

Non-goals: replacing Git; live collaboration; optimising for hand-authored files
over deterministic machine output.

## Package layout

A `.gitmap` package is a directory:

```text
map.gitmap/
  manifest.json      # inline parts + georeferencing
  colors.ndjson      # one color per line
  symbols.ndjson     # one symbol per line
  objects.ndjson     # one object per line
```

Each `*.ndjson` file is one JSON record per line, sorted by a stable key. A gitmap
package is map content only. If an editor or pipeline drops local, non-content
files beside it (view/zoom state, generated `svg`/`geojson` exports), those aren't
part of the format and shouldn't be committed. Templates aren't stored either; they
churn on local paths, and consumers lack the referenced files.

## Determinism and cross-format rules (normative)

A writer MUST:

- write UTF-8 with LF line endings, a trailing newline on each file, and stable
  JSON key ordering;
- sort `colors.ndjson` by `renderOrder` then `id`; `symbols.ndjson` by `code`
  then `id`; `objects.ndjson` by `partId`, then `symbolId`, then geometry (the
  coordinate sequence, then any hole rings), then `text`, then `rotation`;
- omit optional fields that equal their default (`rotation:0`, `text:""`,
  `hAlign:0`, an all-default `pattern`, false booleans);
- store coordinates and scalars at fixed precision (3 dp / `precision`);
- emit no timestamps, derived counts, bounds, or local absolute paths.

Within every record, keys follow a fixed priority order; any key not listed sorts
after all listed keys, alphabetically:

```text
format, version, units, precision, parts, id, order, code, name, type, hidden,
visible, locked, partId, part, coordinates, holes, text, rotation, bounds, rgb,
renderOrder, fontSize, textSymbol, layers, colorId, color, symbolId, width,
height, radius, fontFamily, angle, spacing, lineWidth, opacity, elements,
source, metadata
```

The list spans all record types; a record uses only the keys that apply (an object
has no `rgb`, a colour no `coordinates`). `symbolId` sorts late, so an object emits
`…partId, coordinates, …, symbolId`. Byte-identical output, the conformance bar,
depends on this order.

The rules that make two source formats agree:

1. Coordinates are y-down. OCAD is y-up, so its coordinates are negated on read;
   everything ends in visual (y-down) space.
2. `order` is a dense rank, not the raw source id. An object's `order` is its
   z-rank and a colour's is its palette index, both source ordinals a reader uses
   to restore source order. Symbols store no `order`: it was the code-sorted line
   position (derivable), and a stored index diverged whenever the two exports
   carried different symbol sets. The two formats number records differently, so
   the raw ids can't carry over; files sort by a content key, not source order.
3. Objects carry no stored id; their geometry is their identity. `objects.ndjson`
   sorts by `(partId, symbolId, coordinate sequence, text, rotation)`, so the same
   object lands in the same place from either format. Sorting on the coordinates
   rather than a hash of them keeps a small edit local: a moved object stays near
   its neighbours instead of jumping to a random line. Recognising a move across
   revisions is the differ's job (nearest-match on geometry), not a stored id's.
4. Symbol codes are canonicalised. OCAD's variant-zero suffix collapses (`101.0`
   becomes `101`); a genuinely distinct code like `204.1.0` is kept. A symbol's
   `id` derives from its canonical code, so an object's `symbolId` lines up across
   formats.
5. Precision snaps to OCAD's grid. Rotation (degrees) rounds to 0.1°, CMYK to whole
   percent, and symbol geometry (pattern coordinates, stroke widths) to whole
   units. OCAD can't store finer, so without the snap the two formats disagree in
   the last digit.
6. Holes are structure, not a flag. An area's outer ring is `coordinates`; each
   hole is its own ring in `holes`. Both formats instead flag the boundary on one
   vertex (OCAD on the hole's first coord, OMap on the previous ring's last, its
   `isHolePoint`); gitmap drops the flag and stores the rings, so a diff shows a
   whole hole, not one toggled bit.
7. Coordinate flags are semantic, not format bytes. A vertex is `[x, y]` or
   `[x, y, {…}]`, where the flags say what the point is: `control`, `corner`,
   `dash`. The OCAD `xFlags`/`yFlags` bytes and the OMap curve byte aren't stored;
   Bézier control points come in pairs, so a reader recovers OCAD's cp1/cp2
   (0x01/0x02) from their order.
8. Enumerations are words, not source byte values. `capStyle`/`joinStyle` on
   strokes and `hAlign`/`vAlign` on text read as names (`"round"`, `"square"`,
   `"left"`) rather than a source format's integer, so a diff is legible and the
   stored form doesn't carry OCAD's encoding. A colour reference is `colorId`
   and a coordinate list is `coordinates` everywhere, including inside symbol icon
   primitives.

## Manifest: `manifest.json`

Schema: [`schema/gitmap.manifest.schema.json`](./schema/gitmap.manifest.schema.json).

```json
{
  "format": "gitmap",
  "version": 1,
  "units": "map-units",
  "precision": 3,
  "parts": [
    { "id": "part_main", "name": "Main" }
  ],
  "georeferencing": {
    "scale": 15000,
    "grivation": 23.7,
    "declination": 23.91,
    "projected": {
      "id": "EPSG", "parameter": "2193",
      "refPoint": { "x": 1575000, "y": 5191000 },
      "spec": { "language": "PROJ.4", "value": "+init=epsg:2193" }
    },
    "geographic": {
      "id": "Geographic coordinates",
      "spec": { "language": "PROJ.4", "value": "+proj=latlong +datum=WGS84" },
      "refPointDeg": { "lat": -43.43347842, "lon": 172.69110272 }
    }
  }
}
```

`units` is `map-units` (1 unit = 0.01 mm). `parts` is an inline array of
`{ id, name }`; objects reference a part by `partId`, and the default single part is
`part_main` / `"Main"`. The CRS (`georeferencing`) is inline and optional. For an
OCAD source, `geographic` and `declination` are derived from the projection;
`auxiliaryScaleFactor` is the one CRS field OCAD cannot represent and may be absent.
Optional `notes` and `extensions` carry data the format doesn't otherwise model;
see [Extensions and notes](#extensions-and-notes).

## Extensions and notes

The manifest carries two optional fields for data outside the format's model:

- `notes` — free text from the source map's notes/description field, stored
  verbatim.
- `extensions` — an object of namespaced key/value data a tool attaches to the map
  (survey provenance, pipeline bookkeeping, editor state worth versioning). A writer
  sorts the keys and leaves the values untouched; a reader that doesn't recognise a
  key preserves it rather than dropping it. Namespace keys (reverse-DNS, or a tool
  prefix) so two tools don't collide.

```json
{
  "notes": "Surveyed Feb 2026; contours from 2019 LiDAR.",
  "extensions": {
    "com.example.survey": { "gps": "trimble-r12", "datum": "NZGD2000" }
  }
}
```

### Preserving extensions through a native format

Within a `.gitmap` package, `extensions` is a plain manifest field. OCAD and OMap
have no structured metadata slot, only one free-text notes field, so a writer that
exports to either MUST serialise `extensions` into that notes field as a fenced
block, below any user notes text:

```text
Surveyed Feb 2026; contours from 2019 LiDAR.

--- gitmap-extensions v1 ---
{
  "com.example.survey": { "datum": "NZGD2000", "gps": "trimble-r12" }
}
--- end gitmap-extensions ---
```

- The open fence is `--- gitmap-extensions v<N> ---`, where `N` is the block version
  (currently `1`); the close fence is `--- end gitmap-extensions ---`. Each is on
  its own line, and the body between them is the same sorted JSON as the manifest
  field.
- The marker is plain ASCII, so it survives OMap's XML escaping and OCAD's text
  field unchanged.
- A writer emits the block only when `extensions` has at least one key, so a notes
  field with no extensions round-trips byte-for-byte.
- A reader MUST parse a known-version block into `extensions` and restore the
  surrounding text as `notes`. An unknown fence version is not an error: the reader
  leaves the notes field verbatim, so the block rides through as text, the same
  preserve-don't-discard rule as an unrecognised key. A malformed or nested fence
  is corruption and MUST be rejected.

## Colors: `colors.ndjson`

Sorted by `renderOrder` then `id`. The `id` is `color_<slug(name)>_<order>`, where
`order` (the palette index) disambiguates same-named slots. Symbols reference
colours by this `id`, never a palette index. Both RGB (screen, an `[r,g,b]` integer
triple 0..255) and CMYK (print/OCAD ink, `[c,m,y,k]` in 0..1) are stored. `order` is
the palette index; `renderOrder` is the paint priority (higher = on top). The two
diverge where a source lists colours in a different order than it draws them.
Schema: [`schema/gitmap.color.schema.json`](./schema/gitmap.color.schema.json).

`slug(s)`, used by every id, lowercases `s`, replaces each run of characters outside
`[a-z0-9]` with a single `_`, and trims leading and trailing `_`, so `"Brun 100%"`
becomes `brun_100`. All cross-references are id-keyed, so a conforming writer MUST
slug identically or the reference chain diverges.

```json
{"id":"color_brun_100_6","order":6,"name":"Brun 100%","rgb":[148,96,40],"renderOrder":6,"opacity":1,"cmyk":[0,0.35,0.75,0.42]}
```

## Symbols: `symbols.ndjson`

Sorted by `code` then `id`. The `id` is `sym_<slug(canonical code)>` (`sym_101`);
if two symbols share a canonical code, each after the first takes a `_vN` suffix
(`_v2`, `_v3`). A symbol carries source-independent draw `layers` plus metadata.
Colour references are stable `color_…` ids, lengths are in map units, and angles are
in degrees. A top-level `rotatable:true` marks a symbol that rotates with its object.
Text typography is authoritative in the `text` layer's nested `text` object; there is
no top-level symbol `fontSize`. Schema:
[`schema/gitmap.symbol.schema.json`](./schema/gitmap.symbol.schema.json).

Layer types:

| type | purpose |
| --- | --- |
| `stroke` | line's main stroke (`colorId`/`width`/`dash`/`capStyle`/`joinStyle`, optional `borders`, offsets, mid-symbol placement) |
| `line-elements` | line decoration element arrays (`primSym`/`cornerSym`/`startSym`/`endSymElements`) |
| `fill` | area inner fill |
| `hatch-fill` | area hatch (`colorId`, `spacing`, `lineWidth`, `angle`°, `rotatable`) |
| `point-pattern-fill` | area point-pattern (xmap-native `pattern` record) |
| `point-fill` | point inner disc (`colorId`, `radius`) |
| `point-stroke` | point outer ring (`colorId`, `radius`, `width`) |
| `point-elements` | array of icon primitives (`elements`) |
| `text` | text typography (`fontFamily`, `fontSize` mm, weight, italic, spacing) |

Each source carries some concepts in its own shape: OCAD's `double-line` casing,
`structure-fill` pattern, and `border-symbol` reference; xmap's nested line symbols.
A writer reduces each to one type above (`stroke`, `point-pattern-fill`, an
inlined border, and `line-elements`), so output has one shape per concept and those
input dialects never appear. Field names are frozen for v1; the named fields use
semantic strings (`capStyle`, `joinStyle`), `colorId`, `coordinates`, and
`radius`. The schema stays permissive only for opaque, format-native substructures
(the `pattern` record and nested `element.symbol` trees), which keep their source
field names (`color`, `coords`). The residual cross-format difference is
element-level content, not a type dialect, and affects only `symbols.ndjson` (see
[Known gaps](#known-gaps)).

```json
{"id":"sym_101","code":"101","name":"Contour","type":"line","layers":[{"type":"stroke","colorId":"color_brun_100_6","width":14,"capStyle":"flat","joinStyle":"bevel"}]}
```

## Objects: `objects.ndjson`

One object per line, sorted by `partId`, then `symbolId`, then geometry (the
coordinate sequence, then hole rings), then `text`, then `rotation`, so a symbol's
objects stay contiguous (a diff groups by symbol) and spatially near objects stay
near in the file. There is no stored object `id`. Schema:
[`schema/gitmap.object.schema.json`](./schema/gitmap.object.schema.json).

```json
{"order":1,"type":"line","partId":"part_main","coordinates":[[-18445,-17687,{"corner":true}],[22873,-16816,{"corner":true}]],"symbolId":"sym_101"}
```

Coordinates are compact tuples (map units, y-down): a plain vertex is `[x, y]`, a
flagged vertex `[x, y, {…}]`. The flags object carries only semantic meaning, and
each field is present only when true:

- `control` — a Bézier control point. Control points come in pairs (cp1 then cp2),
  from which a reader recovers OCAD's `xFlags` 0x01/0x02 by position.
- `corner` — a corner (dog-leg) point (OCAD `yFlags` 0x01).
- `dash` — a dash point (OCAD `yFlags` 0x08).

An area with holes stores its outer boundary in `coordinates` and each interior
ring in `holes` (an array of coordinate rings); objects without holes omit `holes`.

Text and area objects may also carry `text`; `rotation` (degrees, CCW); `hAlign`
(`left`/`center`/`right`) and `vAlign` (`baseline`/`top`/`middle`/`bottom`);
`textBox` `{width,height}`; a per-object `pattern` `{rotation?, origin?}` override;
and `tag`/`tagType`, a free-form per-object string with a type code (OCAD's object
string, e.g. course/control codes). The object references its symbol by `symbolId`
only; a diff shows the `symbolId`, and its code is in `symbols.ndjson`.

## Known gaps

The cross-format differences that remain are genuine source/OCAD-format limits,
not dialects; a writer can't bridge them without fabricating data:

- `joinStyle`: OCAD's `lineStyle` byte can't encode some cap+join combos (flat cap
  with round join), so Mapper's OCD export drops the join.
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

`version` starts at `1`. Bump it and add a [`CHANGELOG.md`](./CHANGELOG.md) entry on
any change to the rules above (a change that alters the bytes a conforming
writer produces). Conformance means producing byte-identical output to a conforming
writer for the same map.

## Validating a package

The JSON Schemas are the machine-readable spec: validate a package's files against
them (e.g. with [ajv](https://ajv.js.org), draft 2020-12):

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
