# Research: Multi-Brand Design System in Its Own Repository

Phase 0 of [plan.md](plan.md). Each entry records a decision, why it was made, and what else was considered. All facts were checked on 2026-11-26, against Flutter 3.47.9 / Dart 3.13.7 (the local SDK) and pub.dev.

## R1. Repository move and history

**Decision**: Move the package with `git subtree split --prefix=packages/design_system` in `foundations`.
1. Fetch the split branch into a local clone of `modular-flutter-app-design-system`.
2. Merge it into `main` with `--allow-unrelated-histories`, keeping the existing README commit.
3. Add the repository scaffolding (README, `.gitignore`, `CLAUDE.md`).
4. Tag `v0.1.1`.

Foundations then depends on:

```yaml
design_system:
  git:
    url: https://github.com/EugenioTesio/modular-flutter-app-design-system.git
    ref: v0.1.1
```

Foundations also:
- deletes `packages/design_system/`;
- updates `tool/git_consumer_smoke.sh` so that it checks the git source and tag instead of the nested path;
- replaces the "Moving the Design System" README section with "Developing against a local Design System checkout". That section describes a git-ignored `pubspec_overrides.yaml` pointing to `../design-system`.

The container initializes its existing `design-system` submodule at the `v0.1.1` commit.

**Rationale**:
- Subtree split keeps every commit of the interim package (SC-002). The split puts the package at the repository root, so the git dependency needs no `path:`.
- Merging the unrelated history keeps the remote's README commit, so nothing on the remote has to be force-pushed.
- `v0.1.1` is a patch release: the library is byte-for-byte the same, and only repository files are added (FR-002). Foundations itself releases `0.1.1`, because its dependency source changed and its API did not.

**Alternatives considered**:
- *Force-push the split branch over `main`*: this rewrites a remote branch. Rejected: it is destructive, and the merge achieves the same result.
- *Copy the files without history*: this loses history and breaks SC-002.
- *Keep the version at `0.1.0`*: rejected, because the tag would then describe two different repository states.

## R2. Two releases: an unchanged move, then breaking multi-brand

**Decision**: Ship two Design System releases.
- **`0.1.1`** is the move. Consumers change nothing.
- **`0.2.0`** is multi-brand. It is breaking, comes with migration notes, and is adopted by Foundations `0.2.0`, which is also marked breaking.

**Rationale**:
- Spacing, radius, and the other dimensions become brand values, so reading them as static constants (`DsSpacing.md`) can no longer work. Pre-1.0 semver signals that with a minor bump.
- Splitting the releases keeps the Foundations spec's promise that the move changes 0 consumer lines (SC-001). It also lets the move be verified on its own.

**Alternatives considered**: One combined release. Rejected: it would mix a no-op relocation with a breaking API change and make SC-001 impossible to check.

## R3. How components find the nearest brand

**Decision**: The brand travels inside `ThemeData` as a single `ThemeExtension`, `DsTokens`. It holds the brand, the brightness, and every resolved token group: colors, typography, spacing, radius, border widths, opacities, and icon sizes.

- **Top-level brand**: the App Shell passes `DsTheme.light(brand: b)` and `DsTheme.dark(brand: b)` to `MaterialApp`, which is how the theme is placed today.
- **Nested brand**: a micro app wraps its screens in `DsBrandScope(brand: b, child: …)`. It builds a `Theme` for `b`, using the brightness of the enclosing `Theme`.
- **Reading tokens**: components read tokens through `BuildContext` getters (`context.dsColors`, `context.dsSpacing`, …). These resolve the nearest `DsTokens`, and fall back to the white label when there is none.

**Rationale**:
- `Theme` is an `InheritedTheme`. `showDialog`, `showModalBottomSheet`, and `showMenu` call `InheritedTheme.capture(from: context, to: navigator.context)` (verified in `material/dialog.dart:1640`, `bottom_sheet.dart:1320`, and `popup_menu.dart:1220`), so overlays keep a nested brand (FR-030) with no extra code.
- Replacing `MaterialApp`'s theme keeps the `Navigator` and the widget state (FR-028). With `themeAnimationStyle: AnimationStyle.noAnimation`, `MaterialApp` builds a plain `Theme` instead of an `AnimatedTheme` (verified in `material/app.dart:1056`), so the switch is instant, as the user chose. `DsTokens.lerp` snaps instead of interpolating, so an `AnimatedTheme` placed elsewhere still never mixes two brands.
- Material widgets inside a brand area follow that brand, for example the cursor and selection handles (FR-033).
- Brightness comes from the enclosing `Theme`, so every brand area follows the app's mode (FR-031).
- No component takes a brand (FR-023).

**Alternatives considered**:
- *A custom `InheritedWidget`*: it would still need its own `InheritedTheme.capture` support and a parallel `Theme` for Material widgets, which means two sources of truth.
- *A global initializer and singletons*, as in `ueno_bank`'s `DesignSystemInitializer`: rejected. It allows no nesting, it is global mutable state, and the Foundations spec forbids global access points.
- *Constructor parameters*: rejected by the user.

## R4. Brand file format and parser

**Decision**: Parse a strict, complete subset of the W3C DTCG Format Module 2025.10.

- **Structure**:
  - Tokens are objects that have `$value`. Groups are objects that don't.
  - `$type` is inherited from the nearest group. The parser never infers a type from a value.
  - Names may not start with `$` or contain `{`, `}`, or `.`.
  - `$root` tokens are supported.
  - `$extends` group inheritance is supported, as a deep merge, with cycle detection.
- **References**:
  - Curly-brace aliases such as `"{color.brand.600}"` resolve to whole tokens, and alias chains are followed.
  - JSON Pointer `$ref` (RFC 6901) is resolved within the file, including into sub-values. The spec says tools must support it.
  - Unresolved and circular references are errors.
- **Types used for primitive tokens**:
  - `color`: `srgb`, `hsl`, `hwb`, and `display-p3` color spaces are accepted. Any other color space is accepted only when the optional `hex` field is present (the file's own sRGB fallback). Otherwise it is rejected as `unsupportedFormat`.
  - `dimension`: the `px` unit maps 1:1 to logical pixels. The `rem` unit is multiplied by a fixed 16 px.
  - `fontFamily`: the generic names `system-ui` and `sans-serif` mean the platform font (a null `fontFamily`). An array of names keeps the first non-generic name and passes the rest as fallbacks.
  - `fontWeight`: numbers in 1–1000 and the spec's keyword list. They map to the nearest of `w100` … `w900`.
  - `number`.
  - `typography`: all five sub-values are required. `lineHeight` is a multiplier.
- **Metadata**: it sits in the root group's `$extensions` under the reverse-domain key `com.github.eugeniotesio.design_system`, as `{ "id", "name", "schemaVersion" }`.
- **Unused tokens**: tokens of any `$type` outside the required paths are ignored (FR-012). An unknown `$type` on a required path is a `wrongType` problem.
- **API**:
  - `DsBrandFile.parseJson(String)` and `DsBrandFile.parseMap(Map<String, Object?>)` return the sealed `DsBrandParseResult`: either `DsBrandParseSuccess`, or `DsBrandParseFailure` carrying a `DsBrandFileException` with every `DsBrandFileProblem` (FR-010, FR-014).
  - The parser collects problems in all phases (reading, references, types, required paths) before it reports, so one run lists them all (SC-005).

**Rationale**:
- Figma Variables exports, Tokens Studio's DTCG mode, and plugins such as TokensBrücke all write this format (user decision, research sources below).
- Supporting `$ref`, `$extends`, and `$root` keeps the parser faithful to the spec's MUSTs, so real exports don't fail on valid syntax.
- The Design System can't use Foundations' `Result` because it never depends on Foundations. A sealed result scoped to the feature follows the house rule "return failures, don't throw" and still allows exhaustive `switch`.

**Alternatives considered**:
- *Throwing `DsBrandFileException`*: rejected by FR-014.
- *A custom flat schema*: rejected by the user.
- *Ignoring unsupported color spaces*: that would silently change colors. Rejected by FR-021.
- *Converting `oklch` and `lab`*: this needs gamut mapping, which alters values. Exports usually include `hex`, which covers the case.

**Facts used** (designtokens.org/tr/2025.10/format and /color):
- **Color**: `colorSpace` and `components` are required. `alpha` defaults to 1. `hex`, when present, is 6-digit CSS hex.
- **Dimension**: `{value, unit}`, where `unit` is `px` or `rem`, and is required even when the value is 0.
- **fontWeight**: a number in [1, 1000] or a keyword.
- **Typography**: `fontFamily`, `fontSize`, `fontWeight`, `letterSpacing`, `lineHeight`.
- **Errors**: tools MUST report circular and unresolved references.

## R5. Primitive schema (brand-schema version 1) and semantic mapping

**Decision**:
- Brand-schema version 1 requires 59 tokens. The paths are listed in [data-model.md](data-model.md#brand-file-schema-version-1).
- The Design System owns the mapping from primitive tokens to semantic roles, for light and dark (also in the data model).
- The `textOnAction` color is *selected* from `neutral.0` or `neutral.900`, whichever has the higher contrast against the action color. It is never computed by adjusting a color, so FR-021 holds.

**Rationale**:
- The schema covers everything the eight components use today: the neutral and status scales, a full brand scale (replacing the single `primary` color), the type scale, spacing, radius, and the values hard-coded today (border widths 1 and 2, state-layer opacities 0.08 and 0.12, icon sizes 18, 24, and 48), as FR-024 requires.
- The info scale replaces today's info roles, which were tinted from `primary`.

**Alternatives considered**:
- *Letting brands override semantic roles*: out of scope (spec Assumptions).
- *Requiring only the scale steps the mapping uses*: rejected. Designers export full scales, and future components will need more steps.

## R6. Contrast gate

**Decision**: `DsContrastGate.check(DsBrand)` returns a `DsContrastReport` listing `DsContrastViolation`s (brightness, foreground role, background role, measured ratio, required ratio).

- It uses the WCAG 2.x relative-luminance formula, the one the internal `DsContrast` uses today.
- The pairs and minimums are fixed in [data-model.md](data-model.md#contrast-gate-pairs).
- Repository tests run the gate on every brand file (FR-047).
- In debug builds, `DsTheme`/`DsBrandScope` runs it inside an `assert` and warns once per brand, telling brands apart by value (FR-048, R7). Release builds never run it.

**Rationale**: The checks are the same as the interim package's token tests, now applied to any brand. Measured ratios turn "fails accessibility" into an actionable fix list.

**Alternatives considered**:
- *APCA (WCAG 3 draft)*: not stable yet, and the Foundations spec's rules are written in WCAG 2 ratios.
- *Rejecting a failing brand at parse time*: rejected. A brand that is still being designed must stay previewable in the Widgetbook.

## R7. Debug-only warnings ("warn once")

**Decision**: There are three warnings:
- no brand above a component (FR-032), once per run;
- a theme animation running, detected when `DsTokens.lerp` is called with 0 < t < 1, so brand and mode switches aren't instant (FR-028), once per run;
- a brand that fails the contrast gate (FR-048), once per brand.

All three are reported with `debugPrint`, inside `assert(() { … }())`. They are de-duplicated by a private static `Set<Object>` that exists only in debug builds. The two run-level warnings use a fixed key each. The contrast warning uses the `DsBrand` itself, compared by value, so two different brands that share an id each get their own warning (ids are meant to be unique, but only the Widgetbook checks that).

**Rationale**: "Once per app run" needs memory, and `assert` bodies are removed in release builds. The set is not an access point, holds no product state, and does nothing in release. It is recorded as an accepted exception to gate G3 in [plan.md](plan.md#constitution-check).

**Alternatives considered**:
- *Warning on every build*: floods the console.
- *`FlutterError.reportError`*: shows as a red error in tests, which is too loud for a fallback that works.

## R8. Spacing helpers

**Decision**:
- `DsSpacing` becomes a value class with fields `xxs` … `xxl` and `EdgeInsets` helpers, read with `context.dsSpacing`.
- `DsSpacingStep` is an enum of the same steps.
- `DsGap(step)` is a square gap that reads the nearest brand's spacing. When the guidelines overlay is on, it tints itself without changing its size (FR-025, FR-061).

**Rationale**:
- Screens need spacing that follows the nearest brand without hard-coding numbers.
- An enum gives the lint a precise suggestion to make.
- `DsGap` gives the overlay something to tint, which matches `SpacingFoundation.box` in `ueno_bank`.

**Alternatives considered**: An axis-aware gap (like the `gap` package). Deferred: a square gap works in both `Row` and `Column`, and adds no dependency.

## R9. Widgetbook project

**Decision**: Use `widgetbook` 3.25.0, the stable line; 4.0.0 is in beta. It lives in a `widgetbook/` Flutter app (`design_system_widgetbook`, `publish_to: none`, macOS and web targets) that depends on `design_system: path: ../`.

- **Manual catalog**: the directory is built by hand (`WidgetbookCategory` per atomic level → `WidgetbookComponent` per component → `WidgetbookUseCase`), not by the generator. Components are taken from `DsComponentCatalog.all`, so the folder and the status label (`"DsButton · stable"`) come from the component's own `info` (R14).
- **Addons**, outermost first (the docs: "The first addon in the list will be the outermost widget"):
  1. `ViewportAddon`, with one phone and one tablet viewport. This is the v3 replacement for the device-frame addon, which is deprecated since 3.15.
  2. `MaterialThemeAddon`, with Light (`DsTheme.light()`) and Dark (`DsTheme.dark()`). It sets the brightness.
  3. `DsBrandAddon`, a custom `WidgetbookAddon<DsBrand>` whose `ObjectDropdownField` lists the brands. It wraps the use case in `DsBrandScope`.
  4. `DsGuidelinesAddon`, a custom `BooleanField` that wraps the use case in `DsGuidelines`.
- **Brands folder**: `widgetbook/brands/` is declared as an asset folder. At startup, the Widgetbook reads `AssetManifest`, parses every `brands/*.json`, and removes the duplicate of the hard-coded white label by id. A "Brand files" category shows a token preview for each valid brand and the problem report for each invalid one (FR-036, FR-037).
- **Single brands folder**: this folder is also where the package's tests read the reference and sample brands (`widgetbook/brands/white_label.json`, `widgetbook/brands/sample_forest.json`). The repository therefore has one brands folder.
- **Coverage check**: a test in `widgetbook/test/` builds the directory and checks that every `DsComponentCatalog.all` entry has a component with at least one use case (FR-039).

**Rationale**:
- A manual catalog avoids `build_runner` and generated files. It also lets the status come from one source, and makes the coverage check a plain Dart test.
- `ViewportAddon` is the non-deprecated way to view phone and tablet sizes.
- The brand addon reuses `DsBrandScope`, so the Widgetbook exercises the same nesting path as a micro app.

**Alternatives considered**:
- *`widgetbook_generator` 3.24.0 with `@UseCase`*: rejected because of the extra build step, and because generated names can't carry the status.
- *`DeviceFrameAddon`*: deprecated. It is still available if bezels are wanted.
- *Widgetbook 4 beta*: not stable.
- *A brands folder at the repository root*: Flutter assets must live inside the project that declares them.

**To verify during implementation**: the exact constructor parameters of `ObjectDropdownField` (label builder) and of the manual `Widgetbook*` node classes, and the names of the tablet entries in `IosViewports`/`AndroidViewports`. The docs used here don't show them.

## R10. Golden tests and the 5% comparator

**Decision**:
- **Comparator**:
  - `test/flutter_test_config.dart` replaces the per-file `LocalFileComparator`, which flutter_test sets before `testExecutable` runs, with `DsGoldenComparator extends LocalFileComparator`.
  - It calls `GoldenFileComparator.compareLists(...)` and passes when `result.passed || result.diffPercent <= DsGoldenComparator.threshold`, where `threshold = 0.05`, defined in one place (FR-041).
  - Otherwise it calls `generateFailureOutput(...)`, which writes the diff images to `failures/` next to the test (git-ignored), and throws a message with the measured percentage (FR-042).
  - `diffPercent` is a fraction from 0 to 1 (`ComparisonResult.diffPercent`, `goldens.dart:273`).
- **Fonts**:
  - `test/fonts/` holds Roboto Regular, Medium, and Bold, plus MaterialIcons, all Apache-2.0, with a LICENSE file.
  - They are loaded with `FontLoader` in `flutter_test_config.dart` under the families `Roboto`, `MaterialIcons`, and the sample brand's family.
  - Text is therefore readable, and doesn't depend on the machine's fonts (FR-044). The fonts are never declared as package assets.
- **Image format**:
  - Images are drawn at a fixed logical size per case and a fixed `devicePixelRatio` of 2.0.
  - One image is made per component × state × brand (white label, Forest) × mode, by cropping to a `RepaintBoundary` around the component (FR-040, FR-043).
  - The path is `test/components/goldens/<component>/<state>__<brand>__<mode>.png`.
- **Commands**:
  - Golden tests carry the `golden` tag (in `dart_test.yaml`).
  - Regenerate them with `flutter test --update-goldens --tags golden` (FR-045).
  - Check them on Linux with Docker: `docker run --rm -v "$PWD":/w -w /w ghcr.io/cirruslabs/flutter:3.47.9 flutter test --tags golden` (quickstart V6).

**Rationale**:
- Flutter's own comparator is exact and documents this subclassing pattern.
- Alchemist's research shows that text rasterization differs between macOS and Linux even with the same font. A tolerance is the user's chosen fix, and tight crops keep it from hiding real changes.
- Committing the fonts makes the result independent of the SDK's cache layout. The same files are also in `$FLUTTER_ROOT/bin/cache/artifacts/material_fonts/`.

**Alternatives considered**:
- *Alchemist "CI goldens" with the Ahem font*: platform-independent but unreadable. The user wants readable images and a tolerance.
- *Separate goldens per OS*: double the maintenance.
- *`golden_toolkit`*: discontinued.

## R11. Token lint rules

**Decision**:
- **Package**: `design_system_lints` in `lints/`. It is a Dart package with `analysis_server_plugin: 0.3.26` and `analyzer: 14.7.0` (both require SDK `^3.13.0`, which matches the local 3.13.7), and is tested with `analyzer_testing: 0.4.5` and `test_reflective_loader: 0.6.0`.
- **Entry point**: `lib/main.dart` declares a top-level `plugin` whose `register` adds four **warning** rules. Warning rules are on by default once the plugin is enabled, and each can be turned off under `diagnostics:`.

| Rule | Where it applies | Reports |
|------|------------------|---------|
| `ds_avoid_literal_color` | all `lib/` code | `Color(...)`, `Color.fromARGB/fromRGBO/from`, `Colors.*`, `CupertinoColors.*`. `Colors.transparent` is allowed. |
| `ds_avoid_literal_spacing` | all `lib/` code | Non-zero number literals in `EdgeInsets*`, `SizedBox` width/height/dimension, `Row`/`Column`/`Wrap` `spacing`, `BorderRadius.circular`, and `Radius.circular` |
| `ds_avoid_literal_component_value` | `package:design_system/src/components/**` only | Number literals in `Icon(size:)`, `BorderSide(width:)`, `withValues(alpha:)`, and `withOpacity` |
| `ds_avoid_primitive_color` | `package:design_system/src/components/**` only | Reads of `DsBrand.colors` scales inside components |

- **Corrections**: each report's correction names the replacement, for example `context.dsColors.<role>`, `context.dsSpacing.<step>`, `DsGap(DsSpacingStep.<step>)`, or `context.dsRadius.<step>`.
- **Exemptions**: the rules check `context.isInLibDir`, so `test/` is never reported. The Widgetbook doesn't enable the plugin. Components use `kMinInteractiveDimension` for the 48 × 48 tap target, so there is no literal for the rule to see.
- **Opting in**: consumers opt in with:

  ```yaml
  plugins:
    design_system_lints:
      git:
        url: https://github.com/EugenioTesio/modular-flutter-app-design-system.git
        path: lints
        ref: v0.2.0
  ```

  `git:` is not documented for plugins (only hosted and `path:` are). If quickstart V8 shows that `git:` doesn't resolve, the fallback is `path: ../design-system/lints`, using the container checkout.

**Rationale**: Dart 3.10+ runs first-party plugins in the analysis server, and `dart analyze`/`flutter analyze` report their diagnostics on the command line. That avoids the `custom_lint` runner and its separate process, and keeps the lint dev-only (FR-051).

**Alternatives considered**:
- *`custom_lint`*: a separate runner, and it doesn't show up in `flutter analyze`.
- *Adopting `flutter_design_guard` or `ripplearc_linter`*: generic rules, and they can't name this system's tokens in corrections.
- *Hosting the lint package on pub.dev*: rejected. The repository is `publish_to: none`.

## R12. QA identifiers

**Decision**:
- Every component gets `String? qaIdentifier`.
- `abstract final class DsQaIdentifiers { static const bool enabled = bool.fromEnvironment('DS_QA_IDENTIFIERS'); }`.
- When `enabled` is false, the internal `DsQaSemantics` returns its child unchanged, so the semantics tree is the same as having no identifiers (FR-058, SC-012). When it is true, it puts `Semantics(identifier:)` on the component's own semantics node, never on a new node that carries a label (FR-059).
- **Tests**:
  - The default run checks "off".
  - A second run, `flutter test --dart-define=DS_QA_IDENTIFIERS=true --tags qa`, checks with `find.bySemanticsIdentifier` (available in flutter_test) that each identifier sits on the node that also holds the component's label.

**Rationale**:
- This is the same opt-in-at-build idea as `ueno_bank`'s `QAUtils`/`SemanticIdentifier`, but with a compile-time constant instead of a singleton.
- Two test runs exercise both build modes without adding test hooks to `lib/`.

**Alternatives considered**:
- *A runtime toggle in an `InheritedWidget`*: rejected. It would let a production build turn identifiers on, and FR-058 requires a build switch.
- *Using `Key`s*: not visible to UI automation tools.

## R13. Developer guidelines overlay

**Decision**:
- **Turning it on**: `DsGuidelines(enabled:, child:)` is an `InheritedWidget` that the App Shell or the Widgetbook addon can place. Alternatively, `DsGuidelines.buildEnabled = !kReleaseMode && bool.fromEnvironment('DS_GUIDELINES')` turns it on everywhere.
- **Never in release** (the user's choice): every way of turning it on is ANDed with `!kReleaseMode`. Because `kReleaseMode` is `const`, release builds tree-shake the painter. The rule lives in one internal static method that takes the build mode as a parameter (`DsGuidelinesMode.isActive(releaseMode:, requested:)`), so the release case is unit-tested even though `flutter test` runs in debug mode.
- **Outlines**: every component wraps its output in the internal `DsGuidelinesFrame(level:)`. It paints a dashed outline in the level's color with `CustomPaint(foregroundPainter:)`, which never affects layout (FR-062).
- **Tints**: `DsGap` tints its own box.
- **Colors**: the level colors are fixed debug colors in one file, with `// ignore_for_file: design_system_lints/ds_avoid_literal_color` and a comment saying why. Atoms are pink, molecules blue, organisms orange, and templates green.

**Rationale**:
- This is the `ueno_bank` `DevGuidelines` idea without the `dotted_border` dependency or the singleton.
- A foreground painter guarantees that the layout doesn't change (SC-013).
- Ignoring release builds means a startup toggle wired to a remote flag can't reach end users.

**Alternatives considered**:
- *Brand-colored outlines*: these would disappear in some brands, and debug colors must look the same in every brand.
- *Flutter's `debugPaintSizeEnabled`*: global, and has no idea of atomic levels.

## R14. Component metadata: level, status, catalog

**Decision**:
- Each component declares `static const DsComponentInfo info = DsComponentInfo(name: 'DsButton', level: DsComponentLevel.atom, status: DsComponentStatus.stable, since: '0.1.0');`.
- `DsComponentCatalog.all` lists every component's `info`.
- The class documentation includes a `**Status**: stable.` line.
- A deprecated component also carries Dart's `@Deprecated('Use X. Removed in 0.N.0.')`, which produces use-site warnings (FR-067).
- An architecture test in the package checks the following, using regex over the source:
  - every exported component class has `info` and is in the catalog;
  - the documented status matches `info.status`;
  - every deprecated status has an `@Deprecated` annotation that names a removal version.

**Rationale**: One value, `info`, drives the overlay level, the Widgetbook grouping and label, the coverage check, and the agent's checklist. The architecture test keeps the documentation in step with it.

**Alternatives considered**:
- *Custom annotations read by the analyzer*: they need a generator.
- *A status kept only in the documentation*: not checkable.

## R15. Versioning, changelog, and brand-schema versions

**Decision**:
- **Versions**: semver. Before 1.0, a minor bump means breaking and a patch means compatible. Tags are `vX.Y.Z`.
- **Changelog**: `CHANGELOG.md` uses Keep a Changelog sections (Added / Changed / Deprecated / Removed / Fixed). Each release with a breaking change has a **Migration** subsection.
- **Supported schema versions**: `DsBrandFile.supportedSchemaVersions` (const `{1}` in `0.2.0`) is stated in the changelog (FR-068).
- **Deprecations**: a deprecated element stays for at least one minor release.
  - `0.2.0` keeps `DsBrand.standard` as a deprecated alias of `DsBrand.whiteLabel`, removed in `0.3.0`. This is the first use of the policy.
- **Foundations**: its changelog states the delivered Design System version, which is already a Foundations rule.

**Rationale**: This is standard practice for design-system packages, and it is what FR-063 to FR-068 ask for.

**Alternatives considered**: Versioning each component separately. Rejected: it is too heavy for 8 components, and Foundations pins one version.

## R16. Component agent and repository conventions

**Decision**: The new repository contains:
- `.claude/agents/ds-component-builder.md`, a Claude Code subagent. Its frontmatter sets the name, the description, and the tools Read, Write, Edit, Grep, Glob, and Bash. Its body is the workflow.
- `CLAUDE.md`, with this container's conventions (dartdoc house style, naming, Gherkin tests, the flutter `widget-composition` skill with no `Widget`-returning methods) plus the Design System's rules.
- `doc/component-checklist.md`, the single checklist that the agent and human reviewers both use.
- `tool/check.sh`, which runs analysis (package, lints, and widgetbook), tests (default, then `qa`), `dart doc --dry-run`, and the Widgetbook coverage test.

The agent's workflow:
1. Read the checklist and an existing component of the same level.
2. Create or review the component and its tests, goldens, Widgetbook use cases, and changelog entry.
3. Run `tool/check.sh`.
4. Report each checklist item as pass or fail (FR-052 to FR-056).

**Rationale**:
- Claude Code is the assistant the user works with.
- One checklist file keeps the agent and human review identical.
- One script gives the agent a single command to verify its work.

**Alternatives considered**:
- *A skill instead of an agent*: the user asked for an agent. A skill could be added later.
- *Duplicating the checklist inside the agent prompt*: it would drift from the human checklist.

## R17. Startup cost of parsing

**Decision**:
- A unit test parses `sample_forest.json` 20 times and asserts a median under 100 ms, which is generous in debug JIT mode.
- Quickstart V3 also measures one parse in a profile build of the Widgetbook on a phone-class device (SC-006).

**Rationale**: About 60 tokens with a few aliases is trivial work. The test only guards against accidental quadratic resolution.

## Sources

- W3C DTCG Format Module 2025.10: https://www.designtokens.org/tr/2025.10/format/ and the Color module https://www.designtokens.org/tr/2025.10/color/
- Widgetbook docs (custom addons, overview and ordering, viewport, theme, fields, knobs): https://docs.widgetbook.io/addons/custom-addon, https://docs.widgetbook.io/addons/overview, https://docs.widgetbook.io/addons/viewport-addon. Versions are from pub.dev (`widgetbook` 3.25.0; 4.0.0-beta.15 is a prerelease).
- Dart analyzer plugins: https://dart.dev/tools/analyzer-plugins and `pkg/analysis_server_plugin/doc/` (writing_a_plugin, writing_rules, testing_rules, writing_fixes)
- Golden comparators with a threshold: https://pub.dev/documentation/adaptive_test/0.4.1/adaptive_test/LocalFileComparatorWithThreshold-class.html. Cross-platform text rendering: https://pub.dev/documentation/alchemist/latest/
- Lint prior art: https://pub.dev/packages/flutter_design_guard
- Lifecycle and deprecation practice: https://www.magicpatterns.com/blog/design-system-maintenance
- User prior art: `ueno_bank/design_system/lib/src/utils/qa_utils.dart`, `dev_guidelines_widget.dart`, and `core/commons/semantic_identifier.dart`
