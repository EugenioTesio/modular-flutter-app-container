# Contract: Widgetbook Project

Covers FR-034 to FR-039, FR-063, and User Story 4. The rationale is in [research R9](../research.md#r9-widgetbook-project).

## Project

| Item | Value |
|------|-------|
| Location | `widgetbook/` in the Design System repository |
| Package | `design_system_widgetbook`, `publish_to: none` |
| Dependencies | `widgetbook: 3.25.0` (exact pin) and `design_system: path: ../`. Nothing depends on this project. |
| Platforms | macOS and web, for local use. It is not hosted. |
| Run | `cd widgetbook && flutter run -d macos` (or `-d chrome`) |
| Assets | `brands/`, the repository's only brands folder |
| Analysis | Its own `analysis_options.yaml`. The `design_system_lints` plugin is not enabled, because use cases may use literal test data. |

## Startup

1. Load `AssetManifest`, and select every asset under `brands/` that ends in `.json`.
2. Parse each file with `DsBrandFile.parseJson`.
3. Sort the valid brands by `name`.
   - A file whose `id` is `white_label` must equal `DsBrand.whiteLabel`. It is shown as the single "White label" entry, and the hard-coded brand is used for it.
   - If two files share an `id`, the second is treated as invalid, with the problem "duplicate id".
4. Invalid files are kept with their `DsBrandFileException`. They never stop the Widgetbook (FR-037).
5. Each valid brand is checked with `DsContrastGate.check`. A brand whose report hasn't `passed` stays selectable and keeps its `DsContrastReport` for the brand switcher and its "Brand files" page (FR-037).

## Addons (outermost first)

| # | Addon | Setting | Effect |
|---|-------|---------|--------|
| 1 | `ViewportAddon` | one phone and one tablet viewport, plus `Viewports.none` | sets the device size, pixel ratio, and platform. It is the v3 replacement for the device-frame addon. |
| 2 | `MaterialThemeAddon` | `Light` = `DsTheme.light()`, `Dark` = `DsTheme.dark()` | sets the brightness the brand follows |
| 3 | `DsBrandAddon` (custom `WidgetbookAddon<DsBrand>`) | `ObjectDropdownField<DsBrand>` listing White label plus the valid brands, labelled by `name`, with `⚠ ` before the name of a brand that fails the contrast gate | wraps the use case in `DsBrandScope(brand: setting)` |
| 4 | `DsGuidelinesAddon` (custom `WidgetbookAddon<bool>`) | `BooleanField('guidelines', initialValue: false)` | wraps the use case in `DsGuidelines(enabled: setting)`. It has no effect in a release build (FR-061). |

The brand addon reuses `DsBrandScope`, so the Widgetbook exercises the same nested-brand path as a micro app.

## Catalog layout

```text
Atoms/
  DsButton · stable           → use cases: Primary, Secondary, Tertiary, With icon, Disabled, Loading, Expanded
  DsTextField · stable        → Label only, Hint and helper, Error, Disabled, Obscured
  DsLoadingIndicator · stable → Default, Custom size
Molecules/
  DsCard · stable             → Static, Tappable
  DsListTile · stable         → Title only, All slots, Tappable
  DsInlineMessage · stable    → Info, Success, Warning, Error, With action
  DsEmptyState · stable       → Title only, With action
Organisms/                    → (empty until an organism exists)
Templates/
  DsPageScaffold · stable     → Body only, With primary action, With back
Brand files/
  <brand name>                → token preview: color scales, semantic roles (light and dark), type scale, spacing, radius
  ⚠ <brand name>              → the same preview, headed by its contrast report (mode · foreground on background · ratio < required)
  ⚠ <file name>               → problem report for each invalid file (path · type · message)
```

- The folders and the `· status` suffix come from `DsComponentCatalog.all`, so they always match each component's `info` (FR-035, FR-063).
- Knobs (`context.knobs.string`, `boolean`, `object.dropdown`, …) expose every constructor parameter except callbacks.

## Coverage check (`widgetbook/test/catalog_coverage_test.dart`)

```text
// Feature: Widgetbook covers every public component
Given the Widgetbook directory
  When it is compared with DsComponentCatalog.all, Then every entry has a component node with at least one use case
  When a catalog entry has no node, Then the test fails naming that component (FR-039)
Given a brands folder with one invalid file
  When the Widgetbook starts, Then the valid brands are listed and the invalid file appears only under "Brand files" with its problems
Given a brands folder with one valid brand that fails the contrast gate
  When the Widgetbook starts, Then the brand is selectable as "⚠ <name>", and its "Brand files" page lists every failing pair
```

The package's architecture test makes sure that every exported component is in `DsComponentCatalog.all` ([R14](../research.md#r14-component-metadata-level-status-catalog)). The two tests together make sure that every public component has a use case.
