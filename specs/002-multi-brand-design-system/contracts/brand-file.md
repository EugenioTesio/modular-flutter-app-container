# Contract: Brand File (brand-schema version 1)

Covers FR-008 to FR-017 and FR-068. The token list and validation order are in [data-model.md](../data-model.md#brand-file-schema-version-1), and the parser rationale is in [research R4](../research.md#r4-brand-file-format-and-parser).

## Format

A brand file is a JSON document in the **W3C Design Tokens Community Group Format Module 2025.10**.
- Objects that have `$value` are tokens. Objects that don't are groups.
- `$type` is inherited from the nearest ancestor group.
- The root group carries the brand metadata in `$extensions`.

```json
{
  "$extensions": {
    "com.github.eugeniotesio.design_system": { "id": "forest", "name": "Forest", "schemaVersion": 1 }
  },
  "color": {
    "$type": "color",
    "brand": {
      "50":  { "$value": { "colorSpace": "srgb", "components": [0.93, 0.97, 0.95], "hex": "#edf7f2" } },
      "600": { "$value": { "colorSpace": "srgb", "components": [0.043, 0.431, 0.31], "hex": "#0b6e4f" } }
    },
    "neutral": { "0": { "$value": { "colorSpace": "srgb", "components": [1, 1, 1] } } }
  },
  "font": { "family": { "base": { "$type": "fontFamily", "$value": ["Forest Sans", "system-ui"] } } },
  "typography": {
    "$type": "typography",
    "body": { "$value": {
      "fontFamily": "{font.family.base}",
      "fontSize": { "value": 16, "unit": "px" },
      "fontWeight": "regular",
      "letterSpacing": { "value": 0, "unit": "px" },
      "lineHeight": 1.5
    } }
  },
  "spacing": { "$type": "dimension", "md": { "$value": { "value": 20, "unit": "px" } } },
  "opacity": { "$type": "number", "pressed": { "$value": 0.12 } }
}
```

*This example is abbreviated. A valid file contains all 59 required tokens. The reference template is `widgetbook/brands/white_label.json`.*

## Supported value types

| `$type` | Accepted `$value` | Converted to |
|---------|-------------------|--------------|
| `color` | `{colorSpace, components, alpha?, hex?}`. The `srgb`, `hsl`, `hwb`, and `display-p3` color spaces are converted exactly. Any other color space is accepted **only if** `hex` is present, in which case the hex value is used. `alpha` defaults to 1, and `components` must not contain `"none"`. | `Color`. Display-P3 keeps its color space. |
| `dimension` | `{value: number, unit: "px" \| "rem"}`. The unit is required even when the value is 0. | logical pixels: `px` as is, `rem` × 16 |
| `fontFamily` | a string, or an array of strings | `fontFamily` = the first name that isn't generic, or null when all names are generic (`system-ui`, `sans-serif`), meaning the platform font. The remaining names become `fontFamilyFallback`. |
| `fontWeight` | an integer from 1 to 1000, or a spec keyword (`thin`, `hairline`, `extra-light`, `ultra-light`, `light`, `normal`, `regular`, `book`, `medium`, `semi-bold`, `demi-bold`, `bold`, `extra-bold`, `ultra-bold`, `black`, `heavy`, `extra-black`, `ultra-black`) | the nearest of `FontWeight.w100` to `w900` |
| `number` | a JSON number | `double` |
| `typography` | `{fontFamily, fontSize, fontWeight, letterSpacing, lineHeight}`. All five are required, and each may be a reference. | `DsTypeStyle`, with `lineHeight` as a multiplier |

## References

- **Curly-brace aliases**: `"{group.token}"` resolves to the referenced token's resolved `$value`, and chains are followed. Targeting a group is an `unresolvedReference` error.
- **JSON Pointer `$ref`**: RFC 6901 pointers within the same file, such as `"#/color/brand/600/$value"`, including pointers into sub-values.
- **`$extends`** on groups: a deep merge in which local definitions win. A target that is a token is an `invalidValue`.
- **`$root`**: the base token of a group, addressed as `group.$root`.
- **Cycles**: a cycle in any of these (alias, pointer, or `$extends`) is a `circularReference` problem. The path of every member of the cycle is reported.

## Parsing contract

| Rule | Result |
|------|--------|
| The file is not JSON, or its root is not an object | One `malformedFile` problem, and parsing stops |
| A name starts with `$` and isn't a reserved property, or contains `{`, `}`, or `.` | `invalidValue` at that path |
| A node has both `$value` and children | `invalidValue` |
| The metadata is missing, or `id` or `name` is invalid | `missingToken` or `invalidValue` at `$extensions.com.github.eugeniotesio.design_system.<field>` |
| `schemaVersion` is not in `DsBrandFile.supportedSchemaVersions` | `unsupportedSchemaVersion`. The message names the supported versions. |
| A required path is missing | `missingToken` |
| A required token resolves to the wrong `$type`, or has no type at all (types are never inferred) | `wrongType` |
| A value doesn't match its type, or is out of range (opacity outside 0–1, a negative dimension, a font size of 0 or less) | `invalidValue` |
| Unsupported color space without `hex`, unsupported unit, or `"none"` components | `unsupportedFormat` |
| A reference can't be resolved | `unresolvedReference` |
| A reference forms a cycle | `circularReference` |
| A non-required token has any problem, including an unknown `$type` | Ignored (FR-012) |
| `$description`, `$deprecated`, and unknown `$extensions` keys | Ignored |

All problems are collected before the parser returns. A file with N problems yields exactly N `DsBrandFileProblem`s, ordered by path (SC-005). The parser never throws for bad input (FR-014), and never reads files, assets, or the network (FR-013).

Usage from the App Shell (how the JSON is obtained is out of scope):

```dart
switch (DsBrandFile.parseJson(brandJson)) {
  case DsBrandParseSuccess(:final brand):
    runApp(ShellApp(brand: brand));
  case DsBrandParseFailure(:final exception):
    logger.warning('Brand rejected: ${exception.problems.length} problems'); // never log the raw file
    runApp(const ShellApp(brand: DsBrand.whiteLabel));
}
```

## Schema evolution

- `DsBrandFile.supportedSchemaVersions` is `{1}` in 0.2.0.
- A release that adds, renames, or removes a required token bumps the schema version. Its changelog lists the token paths that changed, and says which versions are still accepted (FR-068).
- `white_label.json` always targets the newest schema version, and its parsed result always equals `DsBrand.whiteLabel` (FR-017).

## Designer guide (`doc/brand-files.md`)

The repository documents the export path for designers:
- Figma Variables or Tokens Studio, in DTCG mode.
- Use the token names in the table in the data model.
- Use sRGB colors, or include `hex`.
- Use `px` units.
- Drop the file into `widgetbook/brands/` and restart the Widgetbook to preview it (SC-007).
