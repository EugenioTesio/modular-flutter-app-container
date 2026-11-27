# Data Model: Multi-Brand Design System

Phase 1 of [plan.md](plan.md). It covers the entities, their fields, the validation rules, and the state transitions. The public Dart signatures are in [contracts/design-system-api.md](contracts/design-system-api.md), and the file format is in [contracts/brand-file.md](contracts/brand-file.md).

## Brand file schema (version 1)

A brand file is a W3C DTCG 2025.10 document. Brand-schema version 1 **requires** the 59 tokens below (FR-019). Any other token is ignored (FR-012). Each path is the DTCG dot path. The type comes from the token's own `$type`, or else from the nearest group's `$type`.

| Group | Required paths | `$type` | Count |
|-------|----------------|---------|-------|
| Brand color scale | `color.brand.{50,100,200,300,400,500,600,700,800,900}` | `color` | 10 |
| Neutral scale | `color.neutral.{0,50,100,200,300,400,500,600,700,800,900}` | `color` | 11 |
| Status scales | `color.{success,warning,error,info}.{100,500,700}` | `color` | 12 |
| Type scale | `typography.{display,headline,title,body,bodyStrong,label,caption}` | `typography` | 7 |
| Spacing | `spacing.{xxs,xs,sm,md,lg,xl,xxl}` | `dimension` | 7 |
| Corner radius | `radius.{sm,md,lg,pill}` | `dimension` | 4 |
| Border widths | `borderWidth.{thin,thick}` | `dimension` | 2 |
| State-layer opacities | `opacity.{hovered,focused,pressed}` | `number` (0–1) | 3 |
| Icon sizes | `iconSize.{sm,md,lg}` | `dimension` | 3 |

**Metadata** sits in the root group's `$extensions`, under the key `com.github.eugeniotesio.design_system`:

| Field | Type | Rule |
|-------|------|------|
| `id` | string | Required. Matches `^[a-z0-9_]+$`, and is unique among the brands an app loads. Only the Widgetbook checks uniqueness; an app that loads several brands keeps their ids unique. |
| `name` | string | Required and non-empty. It is the display name, for example in the Widgetbook switcher |
| `schemaVersion` | integer | Required. Must be in `DsBrandFile.supportedSchemaVersions`, which is `{1}` in 0.2.0 |

## Entities

### Brand (`DsBrand`)

A complete, immutable set of primitive token values. Two brands with equal fields are equal (FR-015).

| Field | Type | Notes |
|-------|------|-------|
| `id` | `String` | From the metadata |
| `name` | `String` | From the metadata |
| `colors` | `DsBrandColors` | `brand` (`DsBrandScale`: `shade50` … `shade900`), `neutral` (`DsNeutralScale`: `shade0` … `shade900`), and `success`, `warning`, `error`, `info` (`DsStatusScale`: `shade100`, `shade500`, `shade700`) |
| `typography` | `DsBrandTypography` | Seven `DsTypeStyle`s: `fontFamily` (`String?`, where null means the platform font), `fontFamilyFallback`, `fontSize`, `fontWeight`, `lineHeight` (a multiplier), and `letterSpacing` |
| `spacing` | `DsSpacing` | `xxs`, `xs`, `sm`, `md`, `lg`, `xl`, `xxl` |
| `radius` | `DsRadius` | `sm`, `md`, `lg`, `pill` |
| `borderWidths` | `DsBorderWidths` | `thin`, `thick` |
| `opacities` | `DsOpacities` | `hovered`, `focused`, `pressed` |
| `iconSizes` | `DsIconSizes` | `sm`, `md`, `lg` |

**Validation**: every field is required, and there are no defaults inside a brand. All dimensions must be at least 0. Opacities must be in [0, 1]. Font sizes must be greater than 0.

**White-label brand** (`DsBrand.whiteLabel`): a `const` instance defined in code (FR-016). `widgetbook/brands/white_label.json` must parse to an equal brand (FR-017). The white label must pass the contrast gate in both modes (FR-018). The proposed values below are confirmed by the contrast gate and the golden images:

- **Neutral and status scales**: unchanged from the 0.1.0 `DsPalette`:
  - neutrals `0xFFFFFFFF` … `0xFF14171C`;
  - success `0xFFE3F4E8` / `0xFF2E9E57` / `0xFF1B6B39`;
  - warning `0xFFFDF1DC` / `0xFFE09B1A` / `0xFF7A4F00`;
  - error `0xFFFCE6E6` / `0xFFD93838` / `0xFFA31F1F`.
- **Brand scale**, from 50 to 900: `0xFFEEF4FD`, `0xFFD6E4FB`, `0xFFB3CDF9`, `0xFF8AB4F8`, `0xFF5A8FEA`, `0xFF3B74E0`, `0xFF1F5FD1`, `0xFF184BA8`, `0xFF12387F`, `0xFF0C2556`. Step 600 is the 0.1.0 standard primary.
  - Light action on the background is about 5.5:1, and white text on the action is about 5.8:1.
  - Dark action (step 300) on the background is about 8.5:1.
- **Info scale**: `0xFFE3EEFD` / `0xFF3B74E0` / `0xFF184BA8`.
- **Typography**: the 0.1.0 type scale (display 32/1.25/w700, headline 24/1.3/w700, title 20/1.3/w600, body 16/1.5/w400, bodyStrong 16/1.5/w600, label 14/1.4/w600, caption 12/1.4/w400). The family is `system-ui`, which is read as the platform font, and letter spacing is 0.
- **Spacing**: 4, 8, 12, 16, 24, 32, 48.
- **Radius**: 4, 8, 16, 999.
- **Border widths**: thin 1, thick 2.
- **Opacities**: hovered 0.08, focused 0.12, pressed 0.12.
- **Icon sizes**: sm 18, md 24, lg 48.

**Sample brand** (`widgetbook/brands/sample_forest.json`): a green brand scale with step 600 = `0x0B6E4F`, larger radii (6, 12, 20, 999), a different spacing scale (4, 8, 12, 20, 28, 36, 52), and the font family `Forest Sans`. No app bundles that font, so it shows the platform fallback. It exists so the golden images and the Widgetbook show that every token group follows the brand.

### Brand area

The part of the widget tree under one brand.
- The top-level area comes from `MaterialApp`'s `theme` and `darkTheme` (`DsTheme.light/dark(brand:)`).
- Nested areas come from `DsBrandScope`.
- A component resolves to the nearest area. If there is none, it uses the white label (FR-027, FR-029, FR-032).

### Tokens (`DsTokens`, a `ThemeExtension`)

The resolved tokens for one brand in one mode. They are installed in `ThemeData.extensions`.

| Field | Type | Derived from |
|-------|------|--------------|
| `brand` | `DsBrand` | the brand being placed |
| `brightness` | `Brightness` | the enclosing `Theme` (FR-031) |
| `colors` | `DsColors` | the [semantic mapping](#semantic-mapping) for `brightness` |
| `typography` | `DsTypography` | `brand.typography`, converted to `TextStyle`s with no color |
| `spacing`, `radius`, `borderWidths`, `opacities`, `iconSizes` | the brand's value classes | copied from `brand` |

`lerp` doesn't interpolate: it returns `this` for t < 0.5 and `other` from t = 0.5, so no frame mixes two brands (FR-028). The App Shell turns off `MaterialApp`'s theme animation, so brand and mode switches are instant. `lerp` only receives a t strictly between 0 and 1 while a theme animation is running, so in debug builds such a call prints one warning per run saying that switches aren't instant (FR-028). Equality is by value.

### Semantic colors (`DsColors`, `DsColorRole`)

There are 20 roles, the same as in 0.1.0: `background`, `surface`, `surfaceVariant`, `border`, `textPrimary`, `textSecondary`, `textDisabled`, `textOnAction`, `actionPrimary`, `actionPrimaryPressed`, `actionDisabled`, `focus`, and the background and foreground pairs for `info`, `success`, `warning`, and `error`. `DsColorRole` is an enum with one value per role, and the contrast gate reports by role.

#### Semantic mapping

The Design System owns this mapping, and brands can't override it (FR-020). Here `n` is the neutral scale, `b` the brand scale, `s`/`w`/`e`/`i` are the success, warning, error, and info scales, and a number is the step.

| Role | Light | Dark |
|------|-------|------|
| background | n50 | n900 |
| surface | n0 | n800 |
| surfaceVariant | n100 | n700 |
| border | n300 | n600 |
| textPrimary | n900 | n50 |
| textSecondary | n600 | n300 |
| textDisabled | n400 | n500 |
| actionPrimary | b600 | b300 |
| actionPrimaryPressed | b700 | b200 |
| actionDisabled | n200 | n700 |
| focus | b600 | b300 |
| textOnAction | n0 or n900, whichever has the higher contrast against actionPrimary | same rule |
| infoBackground / infoForeground | i100 / i700 | i700 / i100 |
| successBackground / successForeground | s100 / s700 | s700 / s100 |
| warningBackground / warningForeground | w100 / w700 | w700 / w100 |
| errorBackground / errorForeground | e100 / e700 | e700 / e100 |

`textOnAction` is chosen from two existing primitives, and nothing is ever adjusted (FR-021).

### Typography (`DsTypography`)

Seven `TextStyle`s: `display`, `headline`, `title`, `body`, `bodyStrong`, `label`, and `caption`. Like 0.1.0, they carry no color, so components apply a color role. `fontFamily` is null when the brand names a generic family. An unavailable family falls back to the platform font.

### Spacing (`DsSpacing`, `DsSpacingStep`) and gap (`DsGap`)

- `DsSpacing` holds the seven values and `EdgeInsets` helpers (`all(step)`, `symmetric(...)`, `only(...)`).
- `DsSpacingStep` enumerates the seven steps.
- `DsGap.step` (a `DsSpacingStep`): `DsGap(step)` is a square box of that step's size, taken from the nearest brand. When the guidelines are on, it is tinted without changing its size.

### Brand file parsing

- **`DsBrandFile`**: static `parseJson(String)`, static `parseMap(Map<String, Object?>)`, and the `supportedSchemaVersions` set.
- **`DsBrandParseResult`** (sealed): either `DsBrandParseSuccess`, whose `DsBrandParseSuccess.brand` is the parsed `DsBrand`, or `DsBrandParseFailure`, whose `DsBrandParseFailure.exception` is the `DsBrandFileException` that lists every problem.
- **`DsBrandFileException`** (`implements Exception`): `problems` (`List<DsBrandFileProblem>`, never empty), and a `toString` that lists every problem.
- **`DsBrandFileProblem`**:
  - `path`: the DTCG dot path. It is empty for whole-file problems. Metadata problems use `$extensions.com.github.eugeniotesio.design_system.<field>`.
  - `type`: a `DsBrandFileProblemType`.
  - `message`: a human-readable explanation, which never contains the file's raw contents beyond the offending value.
- **`DsBrandFileProblemType`**: `malformedFile`, `missingToken`, `wrongType`, `invalidValue`, `unresolvedReference`, `circularReference`, `unsupportedFormat`, `unsupportedSchemaVersion`.

**Validation order**: the parser runs four phases and collects problems in each before it reports (SC-005):
1. **Read the JSON.** If this fails, the report has a single `malformedFile` problem and stops.
2. **Check the structure.** Names may not start with `$` or contain `{`, `}`, or `.`. A node may not have both `$value` and children. `$extends` must target a group.
3. **Resolve references.** This covers aliases, `$ref`, and `$extends`, and detects cycles.
4. **Check types and required tokens.** This covers the metadata and schema version, then every required path: present, of the right type, and with a valid value.

When parsing fails, no brand is produced.

### Contrast gate

- **`DsContrastGate.check(brand)`** returns a `DsContrastReport`.
- **`DsContrastReport`**: `brandId`, `violations` (a `List<DsContrastViolation>`), and `passed` (whether there are no violations).
- **`DsContrastViolation`**: `brightness`, `foreground` (a `DsColorRole`), `background` (a `DsColorRole`), `ratio` (rounded to 2 decimals), and `requiredRatio`.

#### Contrast gate pairs

Each pair is checked in both light and dark mode (FR-046):

| Foreground | Background(s) | Minimum |
|------------|---------------|---------|
| textPrimary | background, surface, surfaceVariant | 4.5 |
| textSecondary | background, surface | 4.5 |
| textOnAction | actionPrimary, actionPrimaryPressed | 4.5 |
| actionPrimary (used as text by tertiary buttons) | background, surface | 4.5 |
| infoForeground | infoBackground | 4.5 |
| successForeground | successBackground | 4.5 |
| warningForeground | warningBackground | 4.5 |
| errorForeground | errorBackground | 4.5 |
| focus | background, surface | 3.0 |

`textDisabled` and `actionDisabled` are exempt, as WCAG exempts inactive components. The pair list belongs to the Design System version. Adding a pair is listed in the changelog.

### Component metadata

- **`DsComponentInfo`**: `name` (the class name), `level`, `status`, `since` (a version), and an optional `removedIn` (a version, required when the status is deprecated).
- **`DsComponentLevel`**: `atom`, `molecule`, `organism`, `template`.
- **`DsComponentStatus`**: `experimental`, `stable`, `deprecated`.
- **`DsComponentCatalog.all`**: a list with every component's `info`, in level order.

Initial catalog for 0.2.0. Every entry is `stable` and `since: 0.1.0` (FR-064):

| Component | Level |
|-----------|-------|
| `DsButton`, `DsTextField`, `DsLoadingIndicator` | atom |
| `DsCard`, `DsListTile`, `DsInlineMessage`, `DsEmptyState` | molecule |
| `DsPageScaffold` | template |

`DsGap` is a layout helper, not a component. It has no `info`, and is excluded from the catalog and from the coverage checks.

### QA identifier and guidelines

- **QA identifier**: `String? qaIdentifier` on every component. It is exposed only when `DsQaIdentifiers.enabled` is true, a `const` taken from `DS_QA_IDENTIFIERS` (FR-057, FR-058).
- **`DsGuidelines`**: an `InheritedWidget` with `enabled`. `DsGuidelines.isEnabledOf(context)` returns `DsGuidelines.buildEnabled` (a `const` from `DS_GUIDELINES`) OR the nearest `enabled`. In release builds it always returns false, whatever the switch and the widget say (FR-061).
- **Level colors**: pink for atoms, blue for molecules, orange for organisms, and green for templates. These are fixed debug colors that don't follow the brand.

### Golden image

A PNG at `test/components/goldens/<component>/<state>__<brand id>__<light|dark>.png`. It is drawn at DPR 2.0 and holds one component in one state, cropped to the component plus `spacing.md` padding (FR-040, FR-043).

## State transitions

**Brand lifecycle**:

```text
brand file contents ──parse──► DsBrandParseFailure (problems)   [no brand]
                     └──────► DsBrandParseSuccess (DsBrand)
DsBrand ──place──► top-level area (MaterialApp theme) or nested area (DsBrandScope)
top-level area ──App Shell rebuilds with another brand──► new tokens apply on the next frame, with no animation; widget state kept
```

**Component status**:

```text
experimental ──(API settled, checklist complete)──► stable ──(replacement exists)──► deprecated ──(≥ 1 minor release)──► removed
experimental ──(abandoned)──► removed      [allowed in any minor release; still listed in the changelog]
```

**Brand-schema version**: a release that adds, renames, or removes a required token bumps `schemaVersion`. `supportedSchemaVersions` may keep accepting older versions only if the Design System can still fill every primitive token from them. Otherwise those versions are dropped and the change is listed in the changelog (FR-068).
