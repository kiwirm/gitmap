# Changelog

All notable changes to the Gitmap specification are recorded here. The `version`
field in `manifest.json` tracks the format version. Bump it on any change to a
canonical rule (a change that alters the bytes a conforming writer produces).

## Version 4 — 2026-10

One concern per file, colour stacking by line order, and a strict reader.
Reference implementation: panmap 0.7.0.

- Package: exactly nine REQUIRED files, `manifest.json`, `parts.ndjson`,
  `georeferencing.json`, `extensions.json`, `notes.txt`, `colors.ndjson`,
  `spot-colors.ndjson`, `symbols.ndjson`, `objects.ndjson`. "None" is an empty
  file (NDJSON, `notes.txt`) or `{}\n` (JSON). A reader rejects a package missing
  any file. There is no `private/view.json` and no templates.
- Manifest: exactly `{"format":"gitmap","version":4}`. A reader rejects any other
  version and any extra key. `units` and `precision` are removed; the spec fixes
  coordinates as map units (0.01 mm) at 3 decimal places.
- Parts move from the manifest to `parts.ndjson`, one `{id, name}` per line; a
  single-part map writes `{"id":"part_main","name":"Main"}`.
- Georeferencing moves from the manifest to `georeferencing.json` (pretty JSON,
  stable key order), `{}` when the map has none.
- Extensions move from the manifest to `extensions.json`, `{}` when none. The
  native-format notes fence (`--- gitmap-extensions v1 ---`) is unchanged.
- Notes move from the manifest to `notes.txt`: plain UTF-8 text with LF line
  endings, trailing whitespace trimmed per line and at the end, followed by
  exactly one `\n`; an empty file when there are no notes. A non-empty
  `notes.txt` without a final newline is invalid.
- Colours: `renderOrder` is removed. Line order is the stacking order, the first
  line being the topmost colour (OCAD colour table / Mapper priority order), so
  inserting a colour adds one line instead of renumbering every colour below it.
  This corrects the version 1 text, which described `renderOrder` as "higher = on
  top"; the sort was always the reverse (ascending, topmost first).
- Colour ids: documented as `color_<slug(name)>` (not
  `color_<slug(name)>_<order>`), with same-slug colours suffixed `_2`, `_3`, … in
  stacking order then `order`. `slug` normalises accents (NFKD, combining marks
  removed). `order` is the source palette number. (This id scheme shipped in
  panmap 0.6.0 under version 3; see below.)
- Spot colours move from the manifest's `spotColors` to `spot-colors.ndjson`,
  `{name, cmyk, rgb}` one per line, `rgb` now an `[r, g, b]` integer array like
  a palette colour's (version 3 wrote the CSS string `"rgb(r, g, b)"`). They are
  not palette colours: no symbol draws
  in one, they have no stacking position, and OCAD and Mapper disagree on whether a
  separation also appears in the palette.
- NDJSON is canonical: one record per line, every line ends in `\n`, no blank
  lines, and zero records is an empty file (previously a lone newline). Readers
  reject blank lines and a missing final newline.
- No fallbacks: a reader accepts exactly version 4. Older packages are
  re-converted from their source, never read through a compatibility path.
- Specification catch-up (behaviour panmap already had, now written down): the
  vertex flag `corner` is gone, with OCAD corner and dash points both written as
  `dash`; symbols carry `protected`; layers are in a fixed canonical group order;
  the additional OCAD-grid snaps (stroke width and offsets, point radii, font
  size, point-element coordinates); the dropped single-format fields; string sorts
  use `localeCompare` collation; key order applies at every depth; the
  extensions fence body is sorted in code-unit order. New schemas for parts, spot
  colours and georeferencing.

## Versions 2 and 3

Implementation-only bumps made in panmap and not separately specified here. A
version 2 or 3 package is not readable by a version 4 reader; re-convert it from
its source. What panmap's history shows:

- **Version 2** (panmap commit `8f70458`, 2026-09-19, "converge OCD/xmap on
  coordinates, flags and symbol defaults"): OCAD's corner and dash vertex flags
  collapsed into one `dash` flag (Mapper's DashPoint), with the OCD writer
  re-deriving the byte from the symbol; fractional xmap ordinates rounded rather
  than truncated onto the OCAD grid; spot colours modelled apart from the palette
  and carried as a manifest `spotColors` array; Mapper's default
  `midSymbolsPerSpot`/`minimumMidSymbolCount` dropped.
- **Version 3** (panmap 0.5.0, commit `aa1b511`, 2026-10-18): symbols gained
  `protected`, and text typography gained a canonical `lineBelow` sub-object; the
  serialiser dropped remaining single-format fields (`frColor`/`frStyle`,
  top-level `dashLength`/`breakLength`, `minArea`, `iconText`, wrapper metadata).
  panmap 0.6.0 (`059d824`) and 0.6.1 (`f4460cc`) then changed output bytes without
  a further bump: name-based colour ids with accent-safe slugs (replacing
  `color_<name>_<number>`), canonical LF notes with trailing whitespace trimmed,
  dashed mid-symbol spacing, and border/width rounding that makes a gitmap
  re-write a fixed point.

## Version 1 — 2026-09

Initial specification.

- Package: `manifest.json` (inline `parts` and georeferencing) plus `colors.ndjson`
  / `symbols.ndjson` / `objects.ndjson`. Filenames are fixed by convention; there is
  no `files` map and no separate `parts.json`.
- Naming: the dense-rank field is `order` (objects: render-order z-rank; colours:
  palette index, both source ordinals). Symbols store no `order`; their rank is the
  code-sorted line position (derivable), so a re-export with a slightly different
  symbol set doesn't churn every later symbol. A symbol's draw layers are `layers`.
- Coordinates: compact tuples, `[x, y]` or `[x, y, {…}]`, with semantic flags
  (`control`, `corner`, `dash`). Format flag bytes (OCAD `xFlags`/`yFlags`, the OMap
  curve byte) are not stored; Bézier control points come in pairs, so a reader
  recovers cp1/cp2 by order. Symbol icon-primitive (`element`) geometry uses the same
  `coordinates` field and tuple shape, and keeps `hole` since a primitive isn't
  ring-split.
- Hole rings: explicit structure. An area's outer boundary is `coordinates` and its
  holes a separate `holes` array, rather than a positional hole flag on a vertex.
- Colours: `rgb` is an `[r,g,b]` integer triple (0..255), matching `cmyk`'s array
  shape; both are stored so a gitmap-to-OCD export keeps ink.
- Typography: authoritative in the `text` layer's nested `text` object, with no
  duplicated top-level symbol/layer `fontSize`/`fontFamily`.
- Enumerations and references: `capStyle`/`joinStyle` on strokes and `hAlign`/`vAlign`
  on text are semantic strings, not source integers; a colour reference is `colorId`,
  a coordinate list is `coordinates`, and a circular mark's size is `radius`,
  everywhere (elements and borders included); object and pattern rotation are in
  degrees. No derived counts are stored; an element's coordinate count is recomputed
  on read. A free-form per-object payload is `tag`/`tagType`.
- Canonical rules: y-down coordinates; objects carry no stored id and
  `objects.ndjson` sorts by `(partId, symbolId, geometry, text, rotation)`; a fixed
  JSON key order (see the README); canonicalised symbol codes (`101.0` to `101`);
  rotation (degrees) snapped to 0.1° and CMYK to whole percent.
- Georeferencing: `geographic` ref point and `declination` are derivable for OCAD
  sources; `auxiliaryScaleFactor` is not, since OCAD can't store it.
- Cross-format: object and colour geometry is byte-identical; each symbol-layer
  concept has one canonical shape (OCAD's `double-line`, `structure-fill`, and
  `border-symbol`, and xmap's nested line symbols, are canonicalised away and never
  emitted). The residual difference is element-level content bounded by genuine
  source/OCAD-format limits (see the README's Known gaps), not a layer dialect.
- JSON Schema (2020-12) in `schema/`.
