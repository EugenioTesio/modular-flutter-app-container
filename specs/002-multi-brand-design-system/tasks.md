---

description: "Task list for the multi-brand Design System in its own repository"
---

# Tasks: Multi-Brand Design System in Its Own Repository

**Input**: Design documents from `specs/002-multi-brand-design-system/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: The spec asks for tests directly (FR-017, FR-039, FR-040 to FR-045, FR-047, FR-050, and SC-003 to SC-016), so every story phase starts with its test tasks. Write them first, and make sure they fail before you implement.

**Test notation (required, user convention)**: Every test is described in Gherkin notation.
- A file starts with a `// Feature: …` comment that names the rule or story.
- `group('Given …')` holds the shared context.
- Each `test('When …, Then …')` states one behavior.
- The test body is split into `// Given`, `// When`, and `// Then` sections.
- Acceptance scenarios from spec.md keep their Given/When/Then wording.

**Code conventions (required, user conventions)**:
- **Public API documentation**: every public declaration gets a house-style dartdoc comment in the same task that adds it, in the shape the root `CLAUDE.md` describes. Types get a role-noun summary, "Use this…", "This is **not**…", "Built-in:", and a "Usage from a micro app:" label above a ` ```dart ` block. Constructors and methods get `Parameters:` lists. No `@param` tags and no spec IDs in public docs. Components also carry a `**Status**: <status>.` line.
- **Naming**: `…Exception` types in `lib/src/exceptions/`, one file each. "Type", not "kind". Feature-scoped names. Helpers are static methods of a class.
- **Widgets**: load the `flutter:widget-composition` skill before UI work. Never write a `Widget`-returning method; subwidgets are private `StatelessWidget` classes.
- **Tokens**: components read every token from `context.ds*`, use semantic color roles only, and use `kMinInteractiveDimension` for tap targets.

**Outward-facing steps (⚠)**: pushing to a remote and creating tags need the user's go-ahead at that moment. Commits, pushes, and pull requests go through the `git-ship` subagent.

**Organization**: Tasks are grouped by user story so each story can be built and tested on its own.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task)
- **[Story]**: The user story the task belongs to (US1–US9, from spec.md)
- Every task names its exact file paths

## Path Conventions

- `design-system/` is the container submodule for the Design System repository. The package sits at its root, with `widgetbook/` and `lints/` as sibling projects ([plan.md](plan.md#source-code)).
- `foundations/` is the Foundations submodule.
- Paths without one of those prefixes are relative to `design-system/`.
- Constrained fields are cited as `data-model.md` *Entity.field*. The canonical rule lives there; tasks state only what they add.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Get the toolchain and both repositories ready for the move.

- [ ] T001 Verify the toolchain against [plan.md](plan.md#technical-context): `fvm flutter --version` reports Flutter 3.47.9 / Dart 3.13.7, and `docker pull ghcr.io/cirruslabs/flutter:3.47.9` succeeds (needed by the Linux golden check)
- [X] T002 Initialize the container's `design-system` submodule (`git submodule update --init design-system`). Check that `.gitmodules` points it to `modular-flutter-app-design-system.git` and that `design-system/` holds only the remote's README commit on `main`
- [X] T003 Create the branch `002-multi-brand-design-system` in the container and in `foundations/`, and record Foundations' current commit hash as the pre-move baseline for V1 in [quickstart.md](quickstart.md). Foundations has no `v0.1.0` tag, so V1's diff uses this hash.

---

## Phase 2: User Story 1 - Move the Design System to its own repository without breaking anyone (Priority: P1) 🎯 MVP

**Goal**: Release `0.1.1` from the new repository with an unchanged library and its full history. Foundations `0.1.1` switches to it, and consumers change 0 lines ([contracts/repository-and-release.md](contracts/repository-and-release.md#release-011-the-move)).

**Independent Test**: Foundations' tests, its example host, and its git consumer smoke check pass against `design_system` from git tag `v0.1.1`. `git diff` against the baseline touches only the allowed files, and the new repository's log contains every interim commit (quickstart V1).

### Tests for User Story 1

- [X] T004 [US1] Rewrite the checks in `foundations/tool/git_consumer_smoke.sh` so that `design_system` must resolve from git URL `…/modular-flutter-app-design-system.git` at tag `v0.1.1`, with its package root outside `foundations/`, instead of from `packages/design_system`. Run it and confirm it fails before the move.

### Implementation for User Story 1

- [X] T005 [US1] In `foundations/`, run `git subtree split --prefix=packages/design_system -b design-system-split`, and record `git log --oneline -- packages/design_system | wc -l` as N for SC-002
- [X] T006 [US1] In `design-system/`, fetch `design-system-split` from `../foundations` and merge it into `main` with `--allow-unrelated-histories`, keeping the README commit (no force-push). Confirm `git log --oneline | wc -l` ≥ N and that `diff -r lib ../foundations/packages/design_system/lib` is empty
- [X] T007 [P] [US1] Create `design-system/.gitignore` covering `build/`, `.dart_tool/`, `**/failures/`, and `pubspec_overrides.yaml`
- [X] T008 [P] [US1] Rewrite `design-system/README.md`: what the package is, that consumers get it through Foundations, and how to run its tests. Keep it free of `0.2.0` features.
- [X] T009 [P] [US1] Create `design-system/CLAUDE.md` with the container's conventions copied verbatim from the root `CLAUDE.md` (public API documentation, naming, tests). US7 adds the Design System rules.
- [X] T010 [US1] Set `version: 0.1.1` in `design-system/pubspec.yaml`, and add the `0.1.1` entry "Moved to its own repository. No API or behavior change." to `design-system/CHANGELOG.md`. Run `flutter analyze`, `flutter test`, and `(cd example && flutter test)` in `design-system/`, and confirm that all pass.
- [X] T011 [US1] ⚠ With the user's go-ahead, push `design-system` `main` and tag `v0.1.1` on the remote (repository-and-release contract, step 4)
- [X] T012 [US1] In `foundations/pubspec.yaml`, replace `design_system: path: packages/design_system` with the git dependency at `ref: v0.1.1` ([research R1](research.md#r1-repository-move-and-history)), and delete `foundations/packages/design_system/`. Run `flutter pub get` in `foundations/` and `foundations/example/` to refresh both `pubspec.lock` files. Depends on T011.
- [X] T013 [P] [US1] In `foundations/README.md`, replace "Moving the Design System to its own repository" with "Developing against a local Design System checkout". It describes a git-ignored `pubspec_overrides.yaml` that points to `../design-system`, which `foundations/.gitignore` already ignores.
- [X] T014 [US1] Set `version: 0.1.1` in `foundations/pubspec.yaml`, and add the `0.1.1` entry "Delivers design_system 0.1.1 from its own repository." to `foundations/CHANGELOG.md`
- [X] T015 [US1] Verify Foundations: `flutter analyze`, `flutter test`, `(cd example && flutter test)`, and `tool/git_consumer_smoke.sh` all pass, and `git diff <T003 baseline> --stat` touches only `pubspec.yaml`, `pubspec.lock` files, the deleted `packages/design_system/**`, `tool/git_consumer_smoke.sh`, `README.md`, and `CHANGELOG.md` (SC-001)
- [ ] T016 [US1] ⚠ With the user's go-ahead, commit and push Foundations. Then, in the container, check out the `v0.1.1` commit in `design-system`, bump `foundations`, and commit both submodule pointers.
- [ ] T017 [US1] Run [quickstart.md](quickstart.md) V1, using the T003 baseline in place of `v0.1.0`

**Checkpoint**: The Design System lives in its own repository, and nothing that consumes it changed.

---

## Phase 3: Foundational (Blocking Prerequisites for the multi-brand release)

**Purpose**: The `0.2.0` base that US2–US9 build on: the new folder layout, the brand model and the white label, tokens and their semantic mapping, the theme and the `context.ds*` getters, component metadata, and the eight components reading tokens.

**⚠️ CRITICAL**: No US2–US9 work can begin until this phase is complete. Run it after US1, in `design-system/`.

- [ ] T018 [P] Create `dart_test.yaml` declaring the tags `golden` and `qa`, and add an `## [Unreleased]` section to `CHANGELOG.md`
- [ ] T019 [P] Exclude `widgetbook/**` and `lints/**` in `analysis_options.yaml`, and keep `public_member_api_docs`, `comment_references`, and the strict modes on
- [ ] T020 Restructure the package, following the plan's Structure Decision:
  - delete `example/`, which the Widgetbook replaces (FR-034);
  - move `lib/src/{atoms,molecules,templates}/` to `lib/src/components/{atoms,molecules,templates}/`;
  - move `lib/src/theme/ds_contrast.dart` to `lib/src/contrast/ds_contrast.dart`;
  - move `test/helpers/a11y_harness.dart` to `test/support/ds_a11y_harness.dart`;
  - fix the imports and `lib/design_system.dart`.

  `flutter test` still passes.

### Tests for the foundation

- [ ] T021 [P] Write `test/brand/ds_brand_test.dart`: value equality and `hashCode` for `DsBrand` and every value class, `toString` shows the id only, `DsBrand.whiteLabel` holds the values of `data-model.md` *White-label brand*, and the validation asserts of `data-model.md` *DsBrand* (Validation) fire
- [ ] T022 [P] Write `test/tokens/ds_semantic_mapping_test.dart`: every one of the 20 roles in light and dark matches the `data-model.md` *Semantic mapping* table, and `textOnAction` picks `neutral.0` or `neutral.900` by higher contrast without adjusting either
- [ ] T023 [P] Write `test/tokens/ds_tokens_test.dart`: `DsTokens.resolve`, `copyWith`, value equality, and `lerp` returning `this` for t < 0.5 and `other` from t = 0.5, and printing exactly one debug warning per run when called with 0 < t < 1 (`data-model.md` *DsTokens.lerp*; FR-028)
- [ ] T024 [P] Write `test/theme/ds_theme_test.dart`: `DsTheme.light/dark/forBrightness` install `DsTokens`, default to `DsBrand.whiteLabel`, and build a full `ThemeData` whose color scheme, text theme, cursor, and selection colors follow the brand (B6, FR-033)
- [ ] T025 [P] Write `test/theme/ds_theme_context_test.dart`: the getters resolve the nearest `DsTokens` (B1). With no `DsTokens` above, they return the white label for the ambient brightness and print exactly one debug warning per run (B7, FR-032).
- [ ] T026 [P] Write `test/architecture/component_catalog_test.dart`:
  - every exported component class declares `static const DsComponentInfo info` and appears in `DsComponentCatalog.all`, in level order, matching the initial catalog in `data-model.md` (all `stable`, `since: 0.1.0`, FR-064);
  - creating a deprecated `DsComponentInfo` without `data-model.md` *DsComponentInfo.removedIn* fails its assert.
- [ ] T027 [P] Write `test/layout/ds_gap_test.dart`: `DsGap(step)` is a square of the nearest brand's spacing for that step, and follows a nested brand

### Implementation of the foundation

- [ ] T028 [P] Rewrite `lib/src/tokens/ds_spacing.dart` as the `DsSpacing` value class (`xxs` … `xxl`, `operator []`, `all`, `symmetric`, `only`) plus the `DsSpacingStep` enum, replacing the static constants (`data-model.md` *DsBrand.spacing*)
- [ ] T029 [P] Rewrite `lib/src/tokens/ds_radius.dart`, and create `lib/src/tokens/ds_border_widths.dart`, `lib/src/tokens/ds_opacities.dart`, and `lib/src/tokens/ds_icon_sizes.dart` as const value classes (`data-model.md` *DsBrand.radius*, *DsBrand.borderWidths*, *DsBrand.opacities*, *DsBrand.iconSizes*)
- [ ] T030 [P] Create `lib/src/brand/ds_color_scale.dart` (`DsBrandScale`, `DsNeutralScale`, `DsStatusScale`) and `lib/src/brand/ds_brand_colors.dart` (`DsBrandColors`) (`data-model.md` *DsBrand.colors*)
- [ ] T031 [P] Create `lib/src/brand/ds_type_style.dart` (`DsTypeStyle`) and `lib/src/brand/ds_brand_typography.dart` (`DsBrandTypography`, seven styles) (`data-model.md` *DsBrand.typography*)
- [ ] T032 Create `lib/src/brand/ds_brand.dart`, with `DsBrand` (`data-model.md` *DsBrand.id*, whose docs say that an app keeps ids unique, *DsBrand.name*, and the field validation) and the const `DsBrand.whiteLabel` holding the values of `data-model.md` *White-label brand*. Delete `lib/src/theme/ds_brand.dart` and `lib/src/tokens/ds_palette.dart`. Depends on T028–T031.
- [ ] T033 [P] Create `lib/src/tokens/ds_color_role.dart` (20 values), and rewrite `lib/src/tokens/ds_colors.dart` so `DsColors` is a plain value class with `role(DsColorRole)` and is no longer a `ThemeExtension` (`data-model.md` *DsColors*)
- [ ] T034 [P] Rewrite `lib/src/tokens/ds_typography.dart` so that `DsTypography` is built from a `DsBrandTypography` and carries no color (`data-model.md` *DsTypography*)
- [ ] T035 Create the internal `lib/src/theme/ds_semantic_mapping.dart`, which maps a `DsBrand` and a `Brightness` to `DsColors` per the `data-model.md` *Semantic mapping* table. It uses `lib/src/contrast/ds_contrast.dart` only to choose `textOnAction`, and drops the old color adjustment (FR-021). Depends on T032–T034.
- [ ] T036 Create `lib/src/tokens/ds_tokens.dart`: the `DsTokens` `ThemeExtension` with `resolve`, `copyWith`, value equality, and the snapping `lerp`, which warns once inside `assert` when it is called partway through a theme animation (`data-model.md` *DsTokens.brand*, *DsTokens.brightness*, *DsTokens.colors*, *DsTokens.typography*, *DsTokens.lerp*, and *DsTokens.spacing*, *DsTokens.radius*, *DsTokens.borderWidths*, *DsTokens.opacities*, *DsTokens.iconSizes*, which are copied from the brand). Depends on T035.
- [ ] T037 Rewrite `lib/src/theme/ds_theme.dart` as `DsTheme.light/dark/forBrightness`, building a full `ThemeData` from `DsTokens` (B6). Its docs show the App Shell usage with `themeAnimationStyle: AnimationStyle.noAnimation` (B5). Depends on T036.
- [ ] T038 Rewrite `lib/src/theme/ds_theme_context.dart` as the `DsThemeContext` extension with the getters of the API contract, plus the white-label fallback and the warn-once `debugPrint` inside `assert` ([research R7](research.md#r7-debug-only-warnings-warn-once)). Depends on T036.
- [ ] T039 [P] Create `lib/src/layout/ds_gap.dart`: `DsGap(step)` reads `context.dsSpacing` (`data-model.md` *DsGap.step*; FR-025). US8 adds the tint.
- [ ] T040 [P] Create `lib/src/catalog/ds_component_info.dart` (`DsComponentInfo` per `data-model.md` *DsComponentInfo.name*, which must equal the class name, *DsComponentInfo.level*, *DsComponentInfo.status*, *DsComponentInfo.since*, with the assert on *DsComponentInfo.removedIn*; plus `DsComponentLevel` and `DsComponentStatus`) and `lib/src/catalog/ds_component_catalog.dart`
- [ ] T041 [P] Migrate the atoms `lib/src/components/atoms/ds_button.dart`, `ds_text_field.dart`, and `ds_loading_indicator.dart`:
  - every token from `context.ds*`, no literal visual values, and `kMinInteractiveDimension` for tap targets;
  - `static const DsComponentInfo info` (atom, stable, `since: '0.1.0'`) and a `**Status**: stable.` line;
  - `DsLoadingIndicator.size` becomes `double?`, where null means `context.dsIconSizes.md`.

  Update `test/components/ds_button_test.dart`, `ds_text_field_test.dart`, and `ds_loading_indicator_test.dart`.
- [ ] T042 [P] Migrate the molecules `lib/src/components/molecules/ds_card.dart`, `ds_list_tile.dart`, `ds_inline_message.dart`, and `ds_empty_state.dart` the same way (molecule, stable). `DsCard.padding` becomes `EdgeInsetsGeometry?`, where null means `context.dsSpacing.all(DsSpacingStep.md)`. Info tones use the new info roles. Update their four tests in `test/components/`.
- [ ] T043 [P] Migrate the template `lib/src/components/templates/ds_page_scaffold.dart` the same way (template, stable), and update `test/components/ds_page_scaffold_test.dart`
- [ ] T044 Update `lib/design_system.dart`: export the new public names, remove the elements listed in the API contract's "Removed in 0.2.0" table, and fill `DsComponentCatalog.all` with the eight `info` values in level order. Depends on T032–T043.
- [ ] T045 Create `test/support/ds_test_brands.dart`, where `DsTestBrands.all` is `[DsBrand.whiteLabel]` for now. Make `test/support/ds_a11y_harness.dart` run every case for each test brand in light and dark (FR-026). `flutter analyze` reports 0 issues, and `flutter test` passes.

**Checkpoint**: Every component reads the nearest brand. User story work can now begin.

---

## Phase 4: User Story 2 - Brand the whole app from a brand file (Priority: P1)

**Goal**: A brand parsed from a DTCG file, placed at the top of the tree, styles every component. It switches at runtime with no animation and without losing state, and falls back to the white label ([contracts/brand-file.md](contracts/brand-file.md), [contracts/design-system-api.md](contracts/design-system-api.md#placing-a-brand)).

**Independent Test**: Quickstart V2 and V3. Every parsing problem type is reported, `white_label.json` equals `DsBrand.whiteLabel`, and a host with the Forest brand switches to the white label after one frame with typed text and scroll position kept.

### Tests for User Story 2

- [ ] T046 [P] [US2] Create the parser fixtures in `test/brand_file/fixtures/`:
  - one file per `DsBrandFileProblemType`;
  - `seven_problems.json`, with 7 mixed problems;
  - `references.json`, covering an alias chain, a `$ref` into a sub-value, `$extends`, and `$root`;
  - `color_spaces.json`, `units.json`, and `unknown_tokens.json`;
  - `equivalent_a.json` and `equivalent_b.json`.
- [ ] T047 [P] [US2] Write `test/brand_file/ds_brand_file_test.dart` covering every row of the brand-file parsing contract:
  - one test per problem type;
  - `seven_problems.json` yields exactly 7 problems, ordered by path (SC-005);
  - a cycle names every member;
  - unknown tokens are ignored (FR-012), and equivalent files give equal brands (FR-015);
  - `schemaVersion: 2` is rejected with a message naming version 1;
  - `''`, `'[]'`, `'null'`, and truncated JSON never throw (FR-014);
  - messages never contain the raw file contents.
- [ ] T048 [P] [US2] Write `test/brand_file/brand_files_test.dart`:
  - `widgetbook/brands/white_label.json` parses to a brand equal to `DsBrand.whiteLabel` (FR-017);
  - every `widgetbook/brands/*.json` parses;
  - parsing `sample_forest.json` 20 times has a median under 100 ms ([research R17](research.md#r17-startup-cost-of-parsing)).
- [ ] T049 [P] [US2] Write `test/theme/ds_brand_switch_test.dart` for US2 scenarios 1, 4, 5, 7, and 8, using `MaterialApp(theme: DsTheme.light(brand: forest), darkTheme: …, themeAnimationStyle: AnimationStyle.noAnimation)`:
  - every component renders Forest's tokens;
  - after replacing the brand with the white label, a single `pump()` shows white-label values, with no intermediate frame (SC-004);
  - `DsTextField` text and the scroll offset survive the switch;
  - the unbundled `Forest Sans` family renders with no error;
  - dark mode uses the dark semantic colors.
- [ ] T050 [P] [US2] Write `test/architecture/component_constructor_test.dart`: no public component constructor takes a `DsBrand`, a `DsTokens`, or a token value class (FR-023, SC-003, checklist C4)

### Implementation for User Story 2

- [ ] T051 [P] [US2] Create `lib/src/exceptions/ds_brand_file_exception.dart`: `DsBrandFileException implements Exception`, whose `toString` lists every problem (`data-model.md` *DsBrandFileException.problems*)
- [ ] T052 [P] [US2] Create `lib/src/brand_file/ds_brand_file_problem.dart`: `DsBrandFileProblem` and the `DsBrandFileProblemType` enum (`data-model.md` *DsBrandFileProblem.path*, *DsBrandFileProblem.type*, *DsBrandFileProblem.message*)
- [ ] T053 [P] [US2] Create `lib/src/brand_file/ds_brand_parse_result.dart`: the sealed `DsBrandParseResult`, `DsBrandParseSuccess`, and `DsBrandParseFailure` (`data-model.md` *DsBrandParseSuccess.brand*, *DsBrandParseFailure.exception*)
- [ ] T054 [US2] Create the internal `lib/src/brand_file/dtcg_reader.dart`: validation phases 1 and 2 of `data-model.md` *Brand file parsing* (read the JSON, check names and nodes, inherit `$type` from groups without inferring it), collecting problems instead of stopping. Depends on T051–T053.
- [ ] T055 [US2] Create the internal `lib/src/brand_file/dtcg_references.dart`: phase 3, resolving curly aliases, JSON Pointer `$ref`, `$extends` deep merges, and `$root`. It detects cycles and reports every member, per the contract's References section. Depends on T054.
- [ ] T056 [P] [US2] Create the internal `lib/src/brand_file/dtcg_values.dart`: converts `color`, `dimension`, `fontFamily`, `fontWeight`, `number`, and `typography` per the contract's "Supported value types" table (`rem` × 16, generic families become null, keywords map to the nearest `FontWeight`)
- [ ] T057 [US2] Create `lib/src/brand_file/ds_brand_file.dart`:
  - `DsBrandFile.parseJson`, `parseMap`, and `supportedSchemaVersions = {1}`;
  - phase 4: the metadata (`data-model.md` *Metadata.id*, *Metadata.name*, *Metadata.schemaVersion*), then the 59 required paths of `data-model.md` *Brand file schema (version 1)* with the *DsBrand* validation;
  - problems sorted by path, and no exception thrown for bad input.

  Depends on T054–T056.
- [ ] T058 [P] [US2] Create `widgetbook/brands/white_label.json`: all 59 tokens with the values of `data-model.md` *White-label brand*, `id: white_label`, `name: White label`, and `schemaVersion: 1`. Fonts use `system-ui`.
- [ ] T059 [P] [US2] Create `widgetbook/brands/sample_forest.json`: all 59 tokens following `data-model.md` *Sample brand* (`id: forest`, `name: Forest`, family `Forest Sans`). Choose the remaining scale steps so that the brand passes the contrast gate in both modes (checked by US6).
- [ ] T060 [US2] Extend `test/support/ds_test_brands.dart` so that `DsTestBrands.all` loads both files from `widgetbook/brands/` by path through `DsBrandFile`, and the accessibility harness covers white label and Forest × light and dark (SC-016). Fix any component that fails in Forest.
- [ ] T061 [US2] Export `DsBrandFile`, the result types, `DsBrandFileProblem`, `DsBrandFileProblemType`, and `DsBrandFileException` from `lib/design_system.dart`, with house-style docs whose example is the App Shell flow in the brand-file contract
- [ ] T062 [P] [US2] Write `doc/brand-files.md`, the designer guide from the brand-file contract (Figma Variables or Tokens Studio in DTCG mode, the token names, sRGB or `hex`, `px`, and previewing in the Widgetbook)
- [ ] T063 [US2] Run [quickstart.md](quickstart.md) V2 and the test part of V3

**Checkpoint**: The App Shell can brand the whole app from a brand file (P1 complete).

---

## Phase 5: User Story 3 - Brand a micro app's own screens (Priority: P2)

**Goal**: `DsBrandScope` rebrands a subtree. The nearest brand wins, overlays keep it, and it follows the app's mode.

**Independent Test**: Quickstart V4. Inside a Forest scope, components and dialogs use Forest. Outside, they use the top-level brand.

### Tests for User Story 3

- [ ] T064 [P] [US3] Write `test/theme/ds_brand_scope_test.dart`:
  - US3 scenarios 1–5 and rules B2–B5;
  - three nested scopes, where the innermost wins;
  - `showDialog`, `showModalBottomSheet`, and `showMenu` opened inside a scope use its brand;
  - toggling `themeMode` switches the scope to Forest's dark colors;
  - replacing the top-level brand leaves the scope unchanged;
  - changing the scope's brand shows the new brand after one `pump()`;
  - a scope around a nested `Navigator` with a shared layout covers every screen of the group (edge case "Grouped screens").

### Implementation for User Story 3

- [ ] T065 [US3] Create `lib/src/theme/ds_brand_scope.dart`: `DsBrandScope` builds `Theme(data: DsTheme.forBrightness(brand, Theme.of(context).brightness))`. Export it from `lib/design_system.dart`, with a "Usage from a micro app:" example.
- [ ] T066 [US3] Run [quickstart.md](quickstart.md) V4

**Checkpoint**: Micro apps can brand their own screens. Access through Foundations is verified in the release phase (T134).

---

## Phase 6: User Story 4 - Review every component in the Widgetbook (Priority: P2)

**Goal**: The internal `widgetbook/` app shows every component, grouped by level with its status, and has brand, mode, and viewport addons ([contracts/widgetbook.md](contracts/widgetbook.md)).

**Independent Test**: Quickstart V5. The coverage test passes, and a new brand file dropped into `brands/` appears after a restart, with no code change.

### Tests for User Story 4

- [ ] T067 [P] [US4] Write `widgetbook/test/catalog_coverage_test.dart` from the contract's coverage check:
  - every `DsComponentCatalog.all` entry has a component node with at least one use case;
  - a missing node fails the test and names the component (FR-039);
  - a brands folder with one invalid file lists the valid brands and shows the invalid file only under "Brand files".

### Implementation for User Story 4

- [ ] T068 [US4] Create the Widgetbook project:
  - `flutter create --platforms=macos,web --project-name design_system_widgetbook widgetbook`, then delete the generated counter app and test;
  - `widgetbook/pubspec.yaml`: `publish_to: none`, `widgetbook: 3.25.0` exactly, `design_system: path: ../`, and the asset folder `brands/`;
  - `widgetbook/analysis_options.yaml`, without the lint plugin.

  Confirm the APIs that [research R9](research.md#r9-widgetbook-project) lists as "to verify" against the package source, and record any difference in R9.
- [ ] T069 [US4] Create `widgetbook/lib/brand_files/brand_file_loader.dart`, following startup steps 1–4 of the Widgetbook contract:
  - read `AssetManifest`, parse each `brands/*.json`, and sort the valid brands by name;
  - `white_label` must equal `DsBrand.whiteLabel`;
  - a duplicate `id` is reported as invalid with the message "duplicate id";
  - invalid files are kept with their `DsBrandFileException`.

  The core is a pure function over file contents, so tests can call it. In profile mode it prints each file's parse time (quickstart V3).
- [ ] T070 [P] [US4] Create `widgetbook/lib/addons/ds_brand_addon.dart`: a custom `WidgetbookAddon<DsBrand>` whose `ObjectDropdownField` lists White label plus the valid brands by `name`, and wraps the use case in `DsBrandScope`
- [ ] T071 [P] [US4] Create `widgetbook/lib/catalog/catalog_directory.dart`, which builds the Atoms, Molecules, Organisms, and Templates categories from `DsComponentCatalog.all`, names each component `Name · status`, and looks up use cases by component name
- [ ] T072 [P] [US4] Create the atom use cases in `widgetbook/lib/use_cases/ds_button_use_cases.dart`, `ds_text_field_use_cases.dart`, and `ds_loading_indicator_use_cases.dart`, following the contract's catalog layout, with knobs for every parameter except callbacks
- [ ] T073 [P] [US4] Create the molecule use cases in `widgetbook/lib/use_cases/ds_card_use_cases.dart`, `ds_list_tile_use_cases.dart`, `ds_inline_message_use_cases.dart`, and `ds_empty_state_use_cases.dart`, the same way
- [ ] T074 [P] [US4] Create the template use cases in `widgetbook/lib/use_cases/ds_page_scaffold_use_cases.dart`
- [ ] T075 [P] [US4] Create `widgetbook/lib/brand_files/brand_file_use_cases.dart`:
  - a token preview for each valid brand: color scales, semantic roles in light and dark, the type scale, spacing, and radius;
  - a problem report for each invalid file (`path · type · message`).
- [ ] T076 [US4] Create `widgetbook/lib/main.dart`: load the brands, then build `Widgetbook.material` with the addons outermost first:
  - `ViewportAddon` with one phone, one tablet, and `Viewports.none`;
  - `MaterialThemeAddon` with `DsTheme.light()` and `DsTheme.dark()`;
  - `DsBrandAddon`.

  The directory is the catalog plus "Brand files". Depends on T069–T075.
- [ ] T077 [US4] Run [quickstart.md](quickstart.md) V5, including the timed designer flow (SC-007), and the profile-mode parse timing of V3 on a phone-class device (SC-006)

**Checkpoint**: Designers and developers can review every component in every brand.

---

## Phase 7: User Story 5 - Catch visual regressions with golden images (Priority: P2)

**Goal**: Committed golden images for every component, state, brand, and mode, compared with the 5% comparator, passing on macOS and Linux ([contracts/golden-tests.md](contracts/golden-tests.md)).

**Independent Test**: Quickstart V6. The goldens generated on macOS pass in the Linux container, and a changed `color.brand.600` fails them.

### Tests for User Story 5

- [ ] T078 [P] [US5] Write `test/support/ds_golden_comparator_test.dart`, covering the contract's three comparison cases: a 3% difference passes and reports `3.00%`; a 6% difference fails, names the image, and writes `failures/`; a missing golden fails with a message naming `--update-goldens`

### Implementation for User Story 5

- [ ] T079 [P] [US5] Copy Roboto Regular, Medium, and Bold, and MaterialIcons, from `$FLUTTER_ROOT/bin/cache/artifacts/material_fonts/` into `test/fonts/`, with their Apache-2.0 `LICENSE`. Never declare them as package assets.
- [ ] T080 [US5] Create `test/support/ds_golden_comparator.dart`, as in the contract: `DsGoldenComparator extends LocalFileComparator`, with `threshold = 0.05` as the single place the tolerance is set (FR-041)
- [ ] T081 [P] [US5] Create `test/support/ds_test_fonts.dart`: `DsTestFonts.load()` registers Roboto as `Roboto` and `Forest Sans`, and MaterialIcons, through `FontLoader`
- [ ] T082 [US5] Create `test/flutter_test_config.dart`, which installs `DsGoldenComparator` and loads the test fonts. Depends on T080 and T081.
- [ ] T083 [US5] Create `test/support/ds_golden_matrix.dart`:
  - it builds golden cases from the accessibility harness's case list × `DsTestBrands.all` × both modes;
  - each case is cropped to a `RepaintBoundary` with `context.dsSpacing.md` padding on the mode's `background`, at a fixed surface size and a DPR of 2.0;
  - images are named per `data-model.md` *Golden image* (path);
  - indeterminate indicators are captured at a fixed 500 ms tick;
  - every case is tagged `golden`.
- [ ] T084 [P] [US5] Write `test/components/ds_button_golden_test.dart`, `ds_text_field_golden_test.dart`, and `ds_loading_indicator_golden_test.dart` using the matrix
- [ ] T085 [P] [US5] Write `test/components/ds_card_golden_test.dart`, `ds_list_tile_golden_test.dart`, `ds_inline_message_golden_test.dart`, and `ds_empty_state_golden_test.dart` using the matrix
- [ ] T086 [P] [US5] Write `test/components/ds_page_scaffold_golden_test.dart` using the matrix
- [ ] T087 [US5] Run `flutter test --update-goldens --tags golden` on macOS. Review the images in `test/components/goldens/`, and check that each component has one image per state × brand × mode (SC-008).
- [ ] T088 [US5] Write `test/components/golden_sensitivity_test.dart`. It keeps the primary-color list of SC-009 (the filled primary `DsButton`, enabled and loading, and `DsLoadingIndicator`, in light and dark), renders each image with the Forest brand whose `color.brand.600` is changed, and fails if any image on the list passes the comparator. Failure output goes to a temporary folder.
- [ ] T089 [US5] Run the goldens on Linux with `docker run --rm -v "$PWD":/w -w /w ghcr.io/cirruslabs/flutter:3.47.9 flutter test --tags golden`. If an image differs by more than 5%, crop it tighter and regenerate it. Never raise the threshold. Then run [quickstart.md](quickstart.md) V6.

**Checkpoint**: Visual changes are caught on any developer's machine.

---

## Phase 8: User Story 6 - Guardrails that keep brands accessible and components on tokens (Priority: P2)

**Goal**: The contrast gate, its debug warning, and its report in the Widgetbook, plus the `design_system_lints` analyzer plugin with 0 reports in the package ([contracts/lint-rules.md](contracts/lint-rules.md)).

**Independent Test**: Quickstart V7 and V8. A pale brand lists every failing pair, and a literal color in an opted-in project is reported with its replacement.

### Tests for User Story 6

- [ ] T090 [P] [US6] Write `test/contrast/ds_contrast_gate_test.dart`, with the fixture `test/contrast/fixtures/pale_brand.json`:
  - every pair of `data-model.md` *Contrast gate pairs* is checked in both modes at its minimum;
  - the pale fixture reports every failing pair, with mode, roles, ratio, and required ratio;
  - `DsBrand.whiteLabel` and every `widgetbook/brands/*.json` pass (FR-018, FR-047).
- [ ] T091 [P] [US6] Write `test/theme/ds_contrast_warning_test.dart`: in debug, placing the pale brand through `DsTheme` or `DsBrandScope` prints exactly one warning per brand that names the failing pairs, two different failing brands that share an id each warn, and the brand renders unaltered (FR-021, FR-048). The `DsBrandScope` case needs US3.
- [ ] T092 [P] [US6] Write the rule tests `lints/test/ds_avoid_literal_color_test.dart`, `ds_avoid_literal_spacing_test.dart`, `ds_avoid_literal_component_value_test.dart`, and `ds_avoid_primitive_color_test.dart` with `analyzer_testing`, covering the lint contract's positive and negative cases, including the out-of-scope file for the two component-only rules

### Implementation for User Story 6

- [ ] T093 [P] [US6] Create `lib/src/contrast/ds_contrast_report.dart`: `DsContrastReport` and `DsContrastViolation` (`data-model.md` *DsContrastReport.brandId*, *DsContrastReport.violations*, *DsContrastReport.passed*, *DsContrastViolation.brightness*, *DsContrastViolation.foreground*, *DsContrastViolation.background*, *DsContrastViolation.ratio*, *DsContrastViolation.requiredRatio*)
- [ ] T094 [US6] Create `lib/src/contrast/ds_contrast_gate.dart`: `DsContrastGate.check(brand)` runs the pair table in both modes with the WCAG formula in `lib/src/contrast/ds_contrast.dart`. Export the gate and the report types from `lib/design_system.dart`, with a "Usage from a host test" example (US6 scenario 3). Depends on T093.
- [ ] T095 [US6] In `lib/src/theme/ds_theme.dart`, run the gate in `DsTheme.forBrightness` inside `assert` and warn once per brand, keyed by the `DsBrand` value ([research R7](research.md#r7-debug-only-warnings-warn-once)). `DsBrandScope` goes through `forBrightness`, so both placements warn, and release builds never run the gate.
- [ ] T096 [US6] Show the contrast report in the Widgetbook (FR-037, US4 scenario 8). Depends on US4.
  - Add startup step 5 to `widgetbook/lib/brand_files/brand_file_loader.dart`.
  - Prefix failing brands with `⚠ ` in `widgetbook/lib/addons/ds_brand_addon.dart`.
  - Head their preview with the report in `widgetbook/lib/brand_files/brand_file_use_cases.dart`.
  - Add the low-contrast case to `widgetbook/test/catalog_coverage_test.dart`.
- [ ] T097 [US6] Create `lints/pubspec.yaml` (`design_system_lints`, `publish_to: none`, `analysis_server_plugin: 0.3.26`, `analyzer: 14.7.0`; dev: `analyzer_testing: 0.4.5`, `test_reflective_loader: 0.6.0`, `test`) and `lints/analysis_options.yaml`
- [ ] T098 [P] [US6] Create `lints/lib/src/rules/ds_avoid_literal_color.dart` per the lint contract's rule table, with a `static const` `LintCode` and the `context.isInLibDir` check
- [ ] T099 [P] [US6] Create `lints/lib/src/rules/ds_avoid_literal_spacing.dart` the same way
- [ ] T100 [P] [US6] Create `lints/lib/src/rules/ds_avoid_literal_component_value.dart`, scoped to `package:design_system/src/components/**`
- [ ] T101 [P] [US6] Create `lints/lib/src/rules/ds_avoid_primitive_color.dart`, scoped to `package:design_system/src/components/**`
- [ ] T102 [US6] Create `lints/lib/main.dart`, with a top-level `final plugin = DsLintsPlugin();` that registers the four warning rules. `dart test` passes in `lints/`. Depends on T098–T101.
- [ ] T103 [US6] Enable the plugin in the root `analysis_options.yaml` (`plugins: design_system_lints: path: lints`), and fix every report until `flutter analyze` shows 0 issues (FR-050)
- [ ] T104 [US6] Run the consumer opt-in check of quickstart V8 in a scratch app in the container, using the `git:` source. Record whether `git:` works, or whether the `path: ../design-system/lints` fallback is needed, in `specs/002-multi-brand-design-system/research.md` R11 and `contracts/lint-rules.md`. Delete the scratch app afterwards.
- [ ] T105 [US6] Run [quickstart.md](quickstart.md) V7 and V8

**Checkpoint**: Inaccessible brands and off-token code are caught automatically.

---

## Phase 9: User Story 8 - Use QA identifiers and the developer guidelines overlay (Priority: P3)

**Goal**: Opt-in QA identifiers and the level-colored overlay, which never shows in release builds ([research R12](research.md#r12-qa-identifiers), [R13](research.md#r13-developer-guidelines-overlay)). This phase runs before US7 because the component agent's checklist covers its output.

**Independent Test**: Quickstart V9 and V10. Identifiers appear only with the QA switch. The overlay outlines components with no layout change, and does nothing in a release build.

### Tests for User Story 8

- [ ] T106 [P] [US8] Write `test/qa/ds_qa_identifiers_test.dart`: by default no component exposes an identifier, and the semantics tree equals one built without `qaIdentifier` (FR-058, SC-012)
- [ ] T107 [P] [US8] Write `test/qa/ds_qa_identifiers_enabled_test.dart`, tagged `qa` and skipped unless `DsQaIdentifiers.enabled`: `find.bySemanticsIdentifier` finds each component, and its node keeps the accessible label (FR-059)
- [ ] T108 [P] [US8] Write `test/guidelines/ds_guidelines_test.dart`:
  - with `DsGuidelines(enabled: true)`, every component paints a dashed outline in its level's color, and `DsGap` is tinted;
  - every `RenderBox` keeps its size and offset (SC-013);
  - off, nothing is painted;
  - `DsGuidelinesMode.isActive(releaseMode: true, …)` is false whatever the switch and the widget say (FR-061).

### Implementation for User Story 8

- [ ] T109 [P] [US8] Create `lib/src/qa/ds_qa_identifiers.dart` (`DsQaIdentifiers.enabled` from `DS_QA_IDENTIFIERS`) and the internal `lib/src/qa/ds_qa_semantics.dart`, which puts `Semantics(identifier:)` on the component's own node only when enabled (`data-model.md` *QA identifier*)
- [ ] T110 [P] [US8] Create the guidelines files:
  - the internal `lib/src/guidelines/ds_guidelines_mode.dart` (`DsGuidelinesMode.isActive(releaseMode:, requested:)`);
  - `lib/src/guidelines/ds_guidelines.dart` (`data-model.md` *DsGuidelines.enabled*; `buildEnabled = !kReleaseMode && bool.fromEnvironment('DS_GUIDELINES')`);
  - the internal `lib/src/guidelines/ds_guidelines_colors.dart`, with `// ignore_for_file: design_system_lints/ds_avoid_literal_color` and a comment explaining why;
  - the internal `lib/src/guidelines/ds_guidelines_frame.dart`, which draws a dashed outline with a foreground painter.
- [ ] T111 [US8] Tint `DsGap` in `lib/src/layout/ds_gap.dart` when the guidelines are on, without changing its size. Depends on T110.
- [ ] T112 [P] [US8] Add `String? qaIdentifier` (`data-model.md` *QA identifier*) through `DsQaSemantics`, and wrap the output in `DsGuidelinesFrame(level: info.level)`, in `lib/src/components/atoms/ds_button.dart`, `ds_text_field.dart`, and `ds_loading_indicator.dart`. Document the new parameter. Depends on T109 and T110.
- [ ] T113 [P] [US8] Make the same change in `lib/src/components/molecules/ds_card.dart`, `ds_list_tile.dart`, `ds_inline_message.dart`, and `ds_empty_state.dart`
- [ ] T114 [P] [US8] Make the same change in `lib/src/components/templates/ds_page_scaffold.dart`
- [ ] T115 [US8] Export `DsQaIdentifiers` and `DsGuidelines` from `lib/design_system.dart`, with house-style docs. Their "Built-in:" paragraphs say that the overlay is always off in release builds.
- [ ] T116 [US8] Create `widgetbook/lib/addons/ds_guidelines_addon.dart` (`BooleanField('guidelines', initialValue: false)`, which wraps the use case in `DsGuidelines`), and add it as the fourth addon in `widgetbook/lib/main.dart`. Depends on US4.
- [ ] T117 [US8] Run [quickstart.md](quickstart.md) V9 and V10, including the release check `flutter run --release -d macos --dart-define=DS_GUIDELINES=true` in `widgetbook/`

**Checkpoint**: QA engineers and design reviewers have their tools, and end users never see them.

---

## Phase 10: User Story 9 - Follow the component lifecycle and versioning (Priority: P3)

**Goal**: Statuses checked against the docs, the first deprecation (`DsBrand.standard`), and a `0.2.0` changelog with Migration notes and the accepted brand-schema versions ([contracts/repository-and-release.md](contracts/repository-and-release.md#versioning-rules)).

**Independent Test**: Quickstart V11.

### Tests for User Story 9

- [ ] T118 [P] [US9] Write `test/architecture/lifecycle_test.dart`:
  - each component's documented `**Status**:` line matches `info.status`;
  - every deprecated element carries `@Deprecated('Use <X>. Removed in <version>.')`;
  - `DsBrand.standard` is deprecated in favor of `DsBrand.whiteLabel`, with removal in 0.3.0 (FR-067).

### Implementation for User Story 9

- [ ] T119 [US9] Add `@Deprecated('Use DsBrand.whiteLabel. Removed in 0.3.0.') static const DsBrand standard = whiteLabel;` to `lib/src/brand/ds_brand.dart`, with docs
- [ ] T120 [US9] Fill the `Unreleased` section of `CHANGELOG.md` with the `0.2.0` content from the release contract:
  - Added, Changed (including instant brand and mode switches), Deprecated, and Removed;
  - **Migration**, with a before and after snippet for each element in the API contract's removed table, for `DsCard.padding`, for `DsLoadingIndicator.size`, and for `themeAnimationStyle: AnimationStyle.noAnimation`;
  - **Brand-schema versions accepted: 1**.
- [ ] T121 [P] [US9] Add a "Versions and lifecycle" section to `README.md`: semver before 1.0, the component statuses, the deprecation policy, and brand-schema versions
- [ ] T122 [US9] Run [quickstart.md](quickstart.md) V11

**Checkpoint**: Consumers can tell how stable each component is and how to upgrade.

---

## Phase 11: User Story 7 - Build components with the component agent (Priority: P3)

**Goal**: The `ds-component-builder` agent, the shared checklist, the repository's `CLAUDE.md`, and `tool/check.sh` ([contracts/component-agent.md](contracts/component-agent.md)). This phase comes last among the stories because the agent's checklist and checks cover the output of every earlier story.

**Independent Test**: Quickstart V12. The agent builds `DsBadge` in under 1 hour, passing `tool/check.sh` with no convention fixes.

### Implementation for User Story 7

- [ ] T123 [P] [US7] Write `doc/component-checklist.md` with items C1–C13 and their "Checked by" column, from the agent contract
- [ ] T124 [P] [US7] Write `tool/check.sh` (executable, `set -euo pipefail`). It runs the contract's 8 steps in order, stops at the first failure, and prints the step that failed.
- [ ] T125 [US7] Complete `CLAUDE.md` with the Design System rules from the agent contract: tokens, exceptions, lifecycle, changelog, "before finishing, run `tool/check.sh`", and delegation to `ds-component-builder`
- [ ] T126 [US7] Write `.claude/agents/ds-component-builder.md`, with the contract's frontmatter (name, description, and the tools Read, Write, Edit, Grep, Glob, and Bash), its five-step workflow, its refusal table, and its review mode (FR-052 to FR-056)
- [ ] T127 [US7] Run `tool/check.sh`, and confirm that all 8 steps pass on the current repository
- [ ] T128 [US7] Run the agent trial of quickstart V12 on a scratch branch:
  - build `DsBadge`, timing the run (SC-014);
  - ask for a `brand` parameter, and confirm that the agent declines;
  - ask for a review of `DsCard`.

  Improve `.claude/agents/ds-component-builder.md` or `doc/component-checklist.md` if the trial shows gaps, report the results to the user, and delete the scratch branch unless the team wants `DsBadge`.

**Checkpoint**: Every user story is complete in `design-system/`.

---

## Phase 12: Release 0.2.0, Foundations Adoption, and Polish

**Purpose**: Tag `0.2.0`. Foundations `0.2.0` adopts it, marked breaking, and the release is checked from micro apps' side: everything reachable through `package:foundations/foundations.dart` (FR-005; US3 scenario 6; US6 scenario 3).

- [ ] T129 Set `version: 0.2.0` in `pubspec.yaml`, and turn `Unreleased` into `## [0.2.0]` with the date in `CHANGELOG.md`. `tool/check.sh` passes, and so does the Docker golden check.
- [ ] T130 ⚠ With the user's go-ahead, push `design-system` `main` and tag `v0.2.0`
- [ ] T131 Set `ref: v0.2.0` and `version: 0.2.0` in `foundations/pubspec.yaml`, then run `flutter pub get` in `foundations/` and `foundations/example/`. Depends on T130.
- [ ] T132 [P] Migrate the Foundations example:
  - in `foundations/example/lib/cards.dart` and `foundations/example/lib/home.dart`, `SizedBox(… DsSpacing.<step>)` becomes `DsGap(DsSpacingStep.<step>)`;
  - both `MaterialApp`s in `foundations/example/lib/main.dart` get `themeAnimationStyle: AnimationStyle.noAnimation`;
  - check `foundations/example/lib/transfers.dart` for removed names.
- [ ] T133 [P] Migrate the Foundations tests: in `foundations/example/test/design_system_reexport_test.dart`, `extension<DsColors>()` becomes a `DsTokens` or `context.dsColors` read; in `foundations/test/architecture/ambiguous_exports_test.dart`, the static `DsSpacing.md` becomes `DsSpacingStep.md`
- [ ] T134 [P] Write `foundations/example/test/multi_brand_test.dart`, which imports only `package:foundations/foundations.dart` and in Gherkin:
  - parses a brand JSON with `DsBrandFile`;
  - places it at the top level and inside a `DsBrandScope`;
  - runs `DsContrastGate.check`;
  - uses `DsGap` and `context.dsSpacing`.
- [ ] T135 [P] Update the library docs in `foundations/lib/foundations.dart`. They name `DsBrandScope`, `DsBrandFile`, `DsContrastGate`, `DsGap`, and `context.dsSpacing` instead of the static `DsSpacing`.
- [ ] T136 Add the Foundations `0.2.0` CHANGELOG entry to `foundations/CHANGELOG.md`, **marked breaking**: "Delivers design_system 0.2.0 (breaking: see its Migration notes)." Run quickstart V13: `flutter analyze`, `flutter test`, the example tests, and `tool/git_consumer_smoke.sh`, which updates to tag `v0.2.0`. Exactly one `design_system` version resolves.
- [ ] T137 ⚠ With the user's go-ahead, commit and push Foundations. Then, in the container, bump the `design-system` submodule to the `v0.2.0` commit and `foundations` to its new commit, and commit both.
- [ ] T138 [P] Update `specs/002-multi-brand-design-system/plan.md` Risks with what implementation found (Widgetbook API differences, the lint `git:` outcome, any golden crops) and keep `research.md` in step
- [ ] T139 Run every scenario in [quickstart.md](quickstart.md), V1–V13, plus `cd design-system && tool/check.sh`, and report each result
- [ ] T140 Confirm that `grep -rn "packages/design_system" foundations/` finds nothing outside the history entries of `foundations/CHANGELOG.md`, and that `.gitmodules` and both submodule pointers in the container match the `v0.2.0` release

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies.
- **US1 (Phase 2)**: depends on Setup. It is the MVP, and everything after it happens in the new repository.
- **Foundational (Phase 3)**: depends on US1. It blocks US2–US9.
- **US2 (Phase 4)**: depends on Foundational.
- **US3 (Phase 5)**: depends on Foundational. Its tests use the Forest brand from US2's `sample_forest.json`, so run it after T059 (or build Forest in code to run it earlier).
- **US4 (Phase 6)**: depends on US2 (the parser and the brands folder).
- **US5 (Phase 7)**: depends on US2 (Forest and `DsTestBrands`).
- **US6 (Phase 8)**: the gate depends on the Foundational phase. T091 also needs `DsBrandScope` from US3, and T096 needs US4. The lints can start after Foundational.
- **US8 (Phase 9)**: depends on Foundational. T116 depends on US4.
- **US9 (Phase 10)**: depends on Foundational. T120 summarizes every earlier story, so finish it last.
- **US7 (Phase 11)**: depends on US4, US5, US6, US8, and US9, because the checklist and `tool/check.sh` verify their output.
- **Release (Phase 12)**: depends on every story.

### Within Each User Story

- Tests are written first and must fail before implementation.
- Value classes before the classes that use them, then theme wiring, then components, then exports and docs.
- `lib/design_system.dart` is edited by several tasks (T044, T061, T065, T094, T115), so those are never run in parallel with each other.
- ⚠ tasks stop and wait for the user.

### Parallel Opportunities

- **US1**: T007, T008, and T009 (new files), then T013 alongside T012.
- **Foundational**: tests T021–T027 together. Then the value classes T028–T031, T033, T034, T039, and T040, and the three component tasks T041–T043.
- **After US2**: US3, US4, US5, the gate part of US6, US8, and US9's T118 can run side by side if staffed, keeping clear of the shared files noted above.
- **US6**: the four lint rules T098–T101.
- **US8**: the three component tasks T112–T114.

---

## Parallel Example: Foundational component migration

```bash
# Once T028–T040 are done, migrate the three levels together:
Task: "T041 Migrate atoms in lib/src/components/atoms/ and their tests"
Task: "T042 Migrate molecules in lib/src/components/molecules/ and their tests"
Task: "T043 Migrate the template in lib/src/components/templates/ and its test"
```

## Parallel Example: User Story 2

```bash
# Tests first, together:
Task: "T047 Parser tests in test/brand_file/ds_brand_file_test.dart"
Task: "T048 Brand file tests in test/brand_file/brand_files_test.dart"
Task: "T049 Runtime switch tests in test/theme/ds_brand_switch_test.dart"
Task: "T050 Constructor scan in test/architecture/component_constructor_test.dart"

# Then the independent parser pieces:
Task: "T051 DsBrandFileException in lib/src/exceptions/ds_brand_file_exception.dart"
Task: "T052 DsBrandFileProblem in lib/src/brand_file/ds_brand_file_problem.dart"
Task: "T053 DsBrandParseResult in lib/src/brand_file/ds_brand_parse_result.dart"
Task: "T058 white_label.json and T059 sample_forest.json in widgetbook/brands/"
```

## Parallel Example: User Story 6 lints

```bash
Task: "T098 ds_avoid_literal_color in lints/lib/src/rules/"
Task: "T099 ds_avoid_literal_spacing in lints/lib/src/rules/"
Task: "T100 ds_avoid_literal_component_value in lints/lib/src/rules/"
Task: "T101 ds_avoid_primitive_color in lints/lib/src/rules/"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: Setup.
2. Phase 2: US1, the move.
3. **Stop and validate** with quickstart V1, then release `0.1.1`. This alone keeps the Foundations spec's promise that the move costs consumers nothing.

### Incremental Delivery

1. Foundational, then US2: the App Shell can brand the app (the P1 goal for multi-brand).
2. US3, US4, US5, and US6 (P2): micro app brands, the Widgetbook, goldens, and guardrails.
3. US8 and US9, then US7 (P3).
4. Release `0.2.0`, with Foundations `0.2.0` marked breaking.

Nothing between `0.1.1` and `0.2.0` is released, so consumers see one breaking upgrade, with migration notes.

### Parallel Team Strategy

After US2, separate developers can take US3, US4, US5, and US6, then US8 and US9. One person owns `lib/design_system.dart` and `widgetbook/lib/main.dart` merges.

---

## Notes

- [P] tasks touch different files and have no dependency on unfinished tasks.
- [Story] labels map each task to its user story for traceability.
- Commit after each task or logical group through `git-ship`. Pushes and tags are ⚠ steps.
- Stop at any checkpoint to validate a story on its own.
- If a fact in the design documents turns out wrong during implementation (a Widgetbook signature, the plugin `git:` source), fix the document in the same change.
