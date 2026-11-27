# Quickstart: Validating the Multi-Brand Design System

These scenarios prove the feature end to end. Each one names the requirements it covers and what you should see. The details live in the [contracts](contracts/) and the [data model](data-model.md); this file doesn't repeat them.

## Prerequisites

- Flutter 3.47.9 / Dart 3.13.7 (`flutter --version`).
- The container checked out, with its submodules: `git submodule update --init design-system foundations`.
- Docker, for the Linux golden check (V6).
- Claude Code, for the agent trial (V12).

From here on, `DS` means `design-system/`.

## V1 — The move changes no consumer (US1, FR-001–FR-007, SC-001, SC-002)

**Pre-move baseline** (recorded by T003): Foundations commit `42823f45922f70faae2482c3408023e5e8e394b9`. Use it wherever V1 says `<pre-move commit>`.

```bash
cd foundations && git log --oneline -- packages/design_system | wc -l          # N, before the move (on 0.1.0)
```

```bash
cd design-system && git log --oneline | wc -l                                    # ≥ N + scaffolding commits
```

```bash
cd foundations && git diff <pre-move commit> --stat                              # only the files listed in the release contract; no v0.1.0 tag exists, so use the commit recorded before the move
```

```bash
cd foundations && flutter test && (cd example && flutter test)
```

```bash
cd foundations && tool/git_consumer_smoke.sh                                    # design_system resolves from git tag v0.1.1
```

**Expected**:
- Every interim commit appears in the Design System history.
- The Foundations diff touches no file under `example/` or `lib/`.
- All tests pass.
- The smoke check reports the git source and tag.

## V2 — Brand files parse strictly (US2, FR-008–FR-017, SC-005)

```bash
cd design-system && flutter test test/brand_file
```

**Expected**:
- One test per `DsBrandFileProblemType`.
- A fixture with 7 mixed problems reports exactly 7, each with its path.
- `white_label.json` parses to a brand equal to `DsBrand.whiteLabel`.
- Unknown extra tokens are ignored.
- The test with the 100 ms budget passes.

## V3 — The top-level brand applies and switches at runtime (US2, FR-027, FR-028, FR-031–FR-033, SC-003, SC-004, SC-006)

```bash
cd design-system && flutter test test/theme
```

**Expected**:
- Every component under `MaterialApp(theme: DsTheme.light(brand: forest))` renders with Forest's tokens.
- Switching to the white label updates every component after a single `pump()`, with no frame in between, and the text typed into a `DsTextField` and the scroll offset survive the switch.
- Removing the brand falls back to the white label, with exactly one debug warning.
- **Startup cost**: run the Widgetbook in profile mode on a phone-class device (`cd widgetbook && flutter run --profile`). The startup log prints the parse time of each brand file, and each must be under 100 ms.

## V4 — A micro app brands its own screens (US3, FR-029–FR-031)

```bash
cd design-system && flutter test test/theme/ds_brand_scope_test.dart
```

**Expected**:
- Inside `DsBrandScope(brand: forest)`, components use Forest, while components outside keep the top-level brand.
- With three nested levels, the innermost brand wins.
- A `showDialog` and a `showModalBottomSheet` opened from inside the scope render with Forest.
- Toggling the app's `themeMode` switches the scope to Forest's dark colors.
- Replacing the top-level brand leaves the scoped screen unchanged.

## V5 — The Widgetbook shows every component in every brand (US4, FR-034–FR-039, SC-007, SC-008)

```bash
cd design-system/widgetbook && flutter test && flutter run -d macos
```

**Expected**:
- The coverage test passes.
- In the app, the Atoms, Molecules, and Templates folders list each component as `Name · stable`.
- The Brand addon lists "White label" and "Forest".
- The Theme addon switches Light/Dark, the Viewport addon switches phone/tablet, and the Guidelines toggle outlines components.

Designer flow:
1. Copy `brands/sample_forest.json` to `brands/sample_ocean.json`, and change its `id`, `name`, and `color.brand` values.
2. Restart the Widgetbook. "Ocean" appears in the switcher. **Timed**: well under 10 minutes, with no code change.
3. Then break the file by deleting `spacing.md`, and restart. "Ocean" disappears from the switcher, and "Brand files → ⚠ sample_ocean.json" shows `missingToken spacing.md`. The other brands still work.
4. Delete the scratch file.

## V6 — Goldens pass on macOS and Linux, and catch real changes (US5, FR-040–FR-045, SC-009)

```bash
cd design-system && flutter test --update-goldens --tags golden && git status --short test/components/goldens | head
```

```bash
cd design-system && docker run --rm -v "$PWD":/w -w /w ghcr.io/cirruslabs/flutter:3.47.9 flutter test --tags golden
```

**Expected**:
- The goldens are generated on macOS and pass in the Linux container. Any tolerated difference is printed, for example `1.84% (tolerated)`.
- **Negative check**: change `color.brand.600` in `widgetbook/brands/sample_forest.json`, then run `flutter test --tags golden`.
  - Every Forest image on the primary-color list (the filled primary `DsButton`, enabled and loading, and `DsLoadingIndicator`, in both modes) fails with a measured difference above 5%. Thinner uses of the color, such as a focus border, may pass; behavior tests cover them.
  - `failures/` contains the diff images, and `git status` doesn't list them.
  - Revert the change.

## V7 — The contrast gate (US6, FR-046–FR-048, SC-010)

```bash
cd design-system && flutter test test/contrast
```

**Expected**:
- White label and Forest pass in both modes.
- A fixture brand with a pale `brand.600` reports violations such as `light · textOnAction on actionPrimary · 2.31 < 4.5`, and lists every failing pair.
- Copying that fixture into `widgetbook/brands/` makes the repository test fail. While it is there, the Widgetbook lists it as "⚠ <name>", still selectable, and its "Brand files" page lists every failing pair. Remove it afterwards.
- In a debug run, placing the failing brand prints exactly one warning that names the failing pairs.

## V8 — The lint rules (US6, FR-049–FR-051, SC-011)

```bash
cd design-system && flutter analyze && (cd lints && dart test)
```

**Expected**:
- `flutter analyze` reports 0 issues, so the components have no literal values.
- The rule tests pass.

**Consumer opt-in check**:
1. In a scratch Flutter app inside the container, add the `git:` plugin source from [lint-rules.md](contracts/lint-rules.md#opting-in).
2. Write `Container(color: const Color(0xFF00FF00), padding: const EdgeInsets.all(12))`.
3. Run `flutter analyze`. It reports `ds_avoid_literal_color` and `ds_avoid_literal_spacing`, each with its suggested replacement.

If `git:` isn't accepted, repeat with `path: ../design-system/lints`, and record the result in `research.md` R11.

## V9 — QA identifiers (US8, FR-057–FR-059, SC-012)

```bash
cd design-system && flutter test test/qa
```

```bash
cd design-system && flutter test --dart-define=DS_QA_IDENTIFIERS=true --tags qa
```

**Expected**:
- **Default run**: no node carries an identifier, and the semantics tree matches a build without `qaIdentifier`.
- **QA run**: `find.bySemanticsIdentifier('login-submit')` finds the button, and its node still has the label "Sign in".

## V10 — The guidelines overlay (US8, FR-060–FR-062, SC-013)

```bash
cd design-system && flutter test test/guidelines
```

**Expected**:
- With `DsGuidelines(enabled: true)`, each component paints a dashed outline in its level's color, and `DsGap` is tinted.
- Every `RenderBox` keeps the same size and offset as with the overlay off.
- The Widgetbook toggle (V5) shows the same result.
- A build with `--dart-define=DS_GUIDELINES=true` turns the overlay on with no widget.
- The release rule's unit test passes: with `releaseMode: true`, the overlay is off whatever the switch and the widget say.
- **Release check**: `cd widgetbook && flutter run --release -d macos --dart-define=DS_GUIDELINES=true`, then turn on the Guidelines toggle. No component is outlined.

## V11 — Lifecycle and versioning (US9, FR-063–FR-068, SC-015)

```bash
cd design-system && flutter test test/architecture
```

```bash
cd design-system && grep -n "## \[0.2.0\]\|### Migration\|Brand-schema versions" CHANGELOG.md
```

**Expected**:
- Every exported component is in `DsComponentCatalog.all`.
- Each component's documented status matches its `info.status`.
- `0.2.0` has a Migration section and lists brand-schema version 1.
- Code that uses `DsBrand.standard` shows a deprecation warning naming `DsBrand.whiteLabel` and 0.3.0.
- A brand file with `"schemaVersion": 2` is rejected, and the message names the supported version 1.

## V12 — Component agent trial (US7, FR-052–FR-056, SC-014)

1. In `design-system/` with Claude Code, ask: "Use ds-component-builder to create `DsBadge`, an atom showing a short label in info, success, warning, or error tones."
2. Time the run.

**Expected**:
- The agent produces C1–C13 from [component-agent.md](contracts/component-agent.md), runs `tool/check.sh`, and reports the checklist with every item passing, in under 1 hour.
- Asking it to "add a `brand` parameter to `DsBadge`" makes it decline and propose `context.ds*`.
- Asking it to review `DsCard` returns the checklist result for `DsCard`.

Discard the trial branch unless the team wants `DsBadge`.

## V13 — Foundations adopts 0.2.0 (US1 scenario 5, FR-005, FR-066)

```bash
cd foundations && flutter pub get && flutter test && (cd example && flutter test) && tool/git_consumer_smoke.sh
```

**Expected**:
- Exactly one `design_system` version resolves (0.2.0, from git).
- The migrated example and tests pass.
- The Foundations CHANGELOG marks the release as breaking and points to the Design System's Migration notes.

## Full repository check

```bash
cd design-system && tool/check.sh
```

**Expected**: all eight steps pass ([component-agent.md](contracts/component-agent.md#toolchecksh)).
