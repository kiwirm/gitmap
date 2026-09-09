# Changelog

All notable changes to the Gitmap specification are recorded here. The `version`
field in `manifest.json` tracks the format version. Bump it on any change to a
canonical rule (a change that alters the bytes a conforming writer produces).

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
