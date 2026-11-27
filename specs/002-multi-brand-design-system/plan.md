# Implementation Plan: Multi-Brand Design System in Its Own Repository

**Branch**: `002-multi-brand-design-system` (spec-kit feature id; git is still on `001-foundations-package`) | **Date**: 2026-11-26 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/002-multi-brand-design-system/spec.md`

## Summary

The interim `design_system` package leaves `foundations/packages/design_system/` and moves to `modular-flutter-app-design-system`, with its history ([R1](research.md#r1-repository-move-and-history)). It then becomes a multi-brand system in a second release ([R2](research.md#r2-two-releases-an-unchanged-move-then-breaking-multi-brand)):

1. **`0.1.1`, the move.**
   - The library is unchanged. Foundations switches to a git dependency on tag `v0.1.1` and deletes the interim folder.
   - The container's `design-system` submodule points to the tag.
   - Consumers change 0 lines.
2. **`0.2.0`, multi-brand (breaking).**
   - **Brands**: a brand comes from a W3C Design Tokens (2025.10) file and is parsed into a `DsBrand` with 59 primitive tokens ([R4](research.md#r4-brand-file-format-and-parser), [R5](research.md#r5-primitive-schema-brand-schema-version-1-and-semantic-mapping)).
   - **Placement**: the App Shell places the top-level brand through `DsTheme.light/dark(brand:)` on `MaterialApp`. Micro apps wrap screens in `DsBrandScope`.
   - **Reading tokens**: components read every token from the nearest `DsTokens` theme extension, through `context.ds*` getters ([R3](research.md#r3-how-components-find-the-nearest-brand)). The hard-coded `DsBrand.whiteLabel` is the fallback.
   - **Tooling**: the release also adds the following, each in the same repository:
     - a contrast gate ([R6](research.md#r6-contrast-gate));
     - spacing helpers ([R8](research.md#r8-spacing-helpers));
     - QA identifiers ([R12](research.md#r12-qa-identifiers));
     - a guidelines overlay ([R13](research.md#r13-developer-guidelines-overlay));
     - component metadata and a catalog ([R14](research.md#r14-component-metadata-level-status-catalog));
     - a Widgetbook project with brand, mode, viewport, and guidelines addons ([R9](research.md#r9-widgetbook-project));
     - golden tests with a 5% comparator ([R10](research.md#r10-golden-tests-and-the-5-comparator));
     - an analyzer-plugin lint package ([R11](research.md#r11-token-lint-rules));
     - a Claude Code component agent with a shared checklist ([R16](research.md#r16-component-agent-and-repository-conventions)).
   - **Foundations adoption**: Foundations `0.2.0` adopts this release, marked breaking.

## Technical Context

**Language/Version**: Dart 3.13 (`sdk: ^3.13.0`) and Flutter 3.47 stable (`flutter: ">=3.47.0"`). Local SDK: Flutter 3.47.9 / Dart 3.13.7.

**Primary Dependencies**:

- **Package `design_system`**:
  - runtime: `flutter` only (FR-004);
  - dev: `flutter_test`, `flutter_lints`, and `yaml` (architecture tests).
- **`widgetbook/`** (`design_system_widgetbook`, unpublished): `widgetbook: 3.25.0` and `design_system: path: ../`.
- **`lints/`** (`design_system_lints`):
  - runtime: `analysis_server_plugin: 0.3.26` and `analyzer: 14.7.0`;
  - dev: `analyzer_testing: 0.4.5`, `test_reflective_loader: 0.6.0`, and `test`.
- **Foundations**: `design_system` as a git dependency (`ref: v0.1.1`, then `v0.2.0`). Its other dependencies are unchanged.

**Storage**: N/A. Brand files are passed in as content, and the Design System never reads storage, assets, or the network (FR-013). Only the Widgetbook reads its own `brands/` assets.

**Testing**:

- **Unit and widget tests**: `flutter test`.
- **Golden tests**: tagged `golden`, compared with the 5% comparator.
- **QA tests**: tagged `qa`, run with `--dart-define=DS_QA_IDENTIFIERS=true`.
- **Lint rule tests**: `dart test` in `lints/`, with `analyzer_testing`.
- **Widgetbook coverage test**: `flutter test` in `widgetbook/`.
- **Linux check**: the golden tests also run on Linux through the `ghcr.io/cirruslabs/flutter:3.47.9` Docker image.
- **One command**: `tool/check.sh` runs all of the above.

**Target Platform**: Every platform Flutter supports. The package has no platform code. The Widgetbook targets macOS and web, for local use only.

**Project Type**: A Flutter library package at the repository root, plus two internal projects: the Widgetbook app and the analyzer-plugin package.

**Performance Goals**:
- Parsing and placing a brand takes under 100 ms on a mid-range phone (SC-006).
- Replacing the brand updates components on the next frame, with no animation between brands (SC-004).
- The guidelines overlay causes no layout change (SC-013).

**Constraints**:
- **Dependencies**: Flutter-only at runtime, and no reference to Foundations (FR-004).
- **Visual values**: 0 hard-coded in components, except the SDK's `kMinInteractiveDimension` (FR-024).
- **Components**: 0 constructors that take a brand (FR-023).
- **Quality**: 100% of public members documented, and 0 analyzer issues, including 0 plugin reports (FR-050).
- **Goldens**: they pass on macOS and Linux (FR-045).

**Scale/Scope**:
- 8 components carried over.
- About 25 new public types: brand model, parser and result, tokens, gate, catalog, QA, guidelines, and spacing helpers.
- 59 brand tokens.
- About 160 golden images: 8 components × about 5 states × 2 brands × 2 modes.
- 4 lint rules.
- 1 agent.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` is still the unfilled template, so as in feature 001, the plan uses provisional gates drawn from the Foundations spec and this spec:

| # | Provisional gate (source) | Pre-research | Post-design |
|---|---------------------------|--------------|-------------|
| G1 | The Design System depends at runtime on the Flutter SDK only, and never on Foundations or another product package. The Widgetbook, the lints, and the test fonts never become runtime dependencies (FR-004, and FR-056 of the Foundations spec) | PASS | PASS. `pubspec.yaml` has only `flutter`. The Widgetbook and lints are separate projects. Fonts live under `test/`. The architecture test from 0.1.0 is kept ([contracts/repository-and-release.md](contracts/repository-and-release.md)). |
| G2 | Foundations re-exports the Design System unchanged at one pinned version (FR-005; FR-002 and FR-036 of the Foundations spec) | PASS | PASS. There is still one `export`, and only the dependency source changes. |
| G3 | No global access points and no global mutable state (FR-016 of the Foundations spec) | PASS | PASS with one recorded exception. The brand comes from the tree (`Theme`), and the QA and guidelines build switches are `const`. The debug-only "warn once" sets live inside `assert` and are removed in release builds ([R7](research.md#r7-debug-only-warnings-warn-once)). See Complexity Tracking. |
| G4 | Components take no brand and hard-code no visual values (FR-023, FR-024) | PASS | PASS. Tokens come only from `context.ds*`. The lint rules and the architecture test enforce this ([contracts/lint-rules.md](contracts/lint-rules.md)). |
| G5 | Every public member is documented in the house style, and analysis reports 0 issues (CLAUDE.md) | PASS | PASS. `public_member_api_docs` and `comment_references` stay on, and `tool/check.sh` runs `dart doc --dry-run`. |
| G6 | Every component meets the accessibility rules in every repository brand and mode (FR-026, SC-016) | PASS | PASS. The accessibility harness iterates the brands and both modes ([contracts/golden-tests.md](contracts/golden-tests.md#accessibility-harness)). |
| G7 | Consumers change 0 lines on the move (FR-002, SC-001) | PASS | PASS. `0.1.1` is the unchanged library. The multi-brand changes wait for `0.2.0` ([R2](research.md#r2-two-releases-an-unchanged-move-then-breaking-multi-brand)). |

**Recommendation** (carried over from feature 001): Ratify a constitution with `/speckit-constitution`, so these gates become binding.

## Project Structure

### Documentation (this feature)

```text
specs/002-multi-brand-design-system/
├── plan.md              # This file
├── research.md          # Phase 0: decisions R1–R17
├── data-model.md        # Phase 1: brand schema v1, entities, semantic mapping, gate pairs
├── quickstart.md        # Phase 1: runnable validation scenarios V1–V11
├── contracts/
│   ├── repository-and-release.md   # repository layout, the move, versions, changelog, Foundations adoption
│   ├── brand-file.md               # brand-schema v1 file format and parsing rules
│   ├── design-system-api.md        # public Dart surface of design_system 0.2.0
│   ├── widgetbook.md               # Widgetbook project, addons, brands folder, coverage
│   ├── golden-tests.md             # comparator, fonts, naming, accessibility harness, commands
│   ├── lint-rules.md               # design_system_lints rules and opt-in
│   └── component-agent.md          # agent definition, checklist, check script
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks; not created here)
```

### Source Code

**`design-system/`**: the container submodule for repository `modular-flutter-app-design-system`, as of `0.2.0`.

```text
design-system/
├── pubspec.yaml                 # name: design_system, version 0.2.0, publish_to: none, deps: flutter ONLY
├── analysis_options.yaml        # flutter_lints + strict modes + doc lints; plugins: design_system_lints (path: lints);
│                                #   excludes widgetbook/** and lints/**
├── dart_test.yaml               # tags: golden, qa
├── CHANGELOG.md                 # Keep a Changelog sections + Migration; supported brand-schema versions
├── README.md                    # what it is, how consumers get it (through Foundations), how to run the tools
├── CLAUDE.md                    # container conventions + Design System rules
├── .claude/agents/
│   └── ds-component-builder.md  # the component agent (Claude Code subagent)
├── doc/
│   ├── component-checklist.md   # the single checklist (agent + human review)
│   └── brand-files.md           # designer guide: exporting a brand (Figma Variables / Tokens Studio DTCG)
├── tool/
│   └── check.sh                 # analyze (root, lints, widgetbook) → test → test --tags qa → doc dry-run → widgetbook test
├── lib/
│   ├── design_system.dart       # single public library (same name as 0.1.0)
│   └── src/
│       ├── brand/               # ds_brand.dart (+ white label), ds_brand_colors.dart, ds_color_scale.dart,
│       │                        #   ds_brand_typography.dart, ds_type_style.dart
│       ├── brand_file/          # ds_brand_file.dart (parseJson/parseMap), ds_brand_parse_result.dart,
│       │                        #   ds_brand_file_problem.dart, internal: dtcg_reader, dtcg_references, dtcg_values
│       ├── exceptions/          # ds_brand_file_exception.dart (user convention: one file per exception)
│       ├── tokens/              # ds_tokens.dart (ThemeExtension), ds_colors.dart, ds_color_role.dart, ds_typography.dart,
│       │                        #   ds_spacing.dart (+ DsSpacingStep), ds_radius.dart, ds_border_widths.dart,
│       │                        #   ds_opacities.dart, ds_icon_sizes.dart
│       ├── theme/               # ds_theme.dart (light/dark/forBrightness), ds_brand_scope.dart,
│       │                        #   ds_theme_context.dart (context.ds* getters), internal: ds_semantic_mapping.dart
│       ├── contrast/            # ds_contrast_gate.dart, ds_contrast_report.dart, internal: ds_contrast.dart (moved)
│       ├── layout/              # ds_gap.dart
│       ├── catalog/             # ds_component_info.dart (+ DsComponentLevel, DsComponentStatus), ds_component_catalog.dart
│       ├── qa/                  # ds_qa_identifiers.dart, internal: ds_qa_semantics.dart
│       ├── guidelines/          # ds_guidelines.dart, internal: ds_guidelines_frame.dart, ds_guidelines_colors.dart
│       └── components/
│           ├── atoms/           # ds_button, ds_text_field, ds_loading_indicator
│           ├── molecules/       # ds_card, ds_list_tile, ds_inline_message, ds_empty_state
│           ├── organisms/       # (empty; level exists for future components)
│           └── templates/       # ds_page_scaffold
├── test/
│   ├── flutter_test_config.dart # installs DsGoldenComparator, loads test fonts
│   ├── fonts/                   # Roboto Regular/Medium/Bold, MaterialIcons + LICENSE (test-only)
│   ├── support/                 # ds_golden_comparator.dart, ds_test_brands.dart (reads widgetbook/brands), a11y harness
│   ├── architecture/            # runtime deps {flutter}; no foundations import; every component in catalog;
│   │                            #   doc status == info.status; @Deprecated has removal version
│   ├── brand_file/              # parser: every problem type, aliases, $ref, $extends, $root, color spaces, units,
│   │                            #   white-label reference file == DsBrand.whiteLabel, 100 ms parse budget
│   ├── tokens/                  # semantic mapping per mode, snapping lerp, equality
│   ├── theme/                   # nearest brand, nesting, overlays keep brand, mode follows app, runtime switch keeps state,
│   │                            #   white-label fallback + single debug warning
│   ├── contrast/                # gate pairs and ratios; every repository brand passes
│   ├── qa/                      # off by default (tree identical); tagged qa: identifiers on labelled nodes
│   ├── guidelines/              # outlines per level, DsGap tint, zero layout change
│   └── components/              # behavior + a11y per component; goldens/<component>/<state>__<brand>__<mode>.png
├── widgetbook/                  # internal Flutter app: design_system_widgetbook (publish_to: none)
│   ├── pubspec.yaml             # widgetbook 3.25.0, design_system: path ../ ; assets: brands/
│   ├── brands/                  # THE repository brands folder: white_label.json (reference/template), sample_forest.json
│   ├── lib/
│   │   ├── main.dart            # loads brands/ from AssetManifest, builds Widgetbook.material
│   │   ├── addons/              # ds_brand_addon.dart, ds_guidelines_addon.dart
│   │   ├── catalog/             # directory built from DsComponentCatalog.all (level → component "Name · status" → use cases)
│   │   ├── use_cases/           # one file per component
│   │   └── brand_files/         # token preview + problem report use cases
│   └── test/                    # coverage: every catalog entry has ≥1 use case; invalid brand isolated
└── lints/                       # design_system_lints (analyzer plugin, dev-only for consumers)
    ├── pubspec.yaml             # analysis_server_plugin 0.3.26, analyzer 14.7.0
    ├── lib/main.dart            # top-level `plugin`; registers 4 warning rules
    ├── lib/src/rules/           # one file per rule
    └── test/                    # analyzer_testing rule tests
```

**`foundations/`** (existing submodule). Changes for `0.1.1`:

```text
foundations/
├── pubspec.yaml                 # design_system: git url … ref v0.1.1  (was path: packages/design_system)
├── packages/design_system/      # DELETED
├── tool/git_consumer_smoke.sh   # checks design_system resolves from the git tag, not a nested path
├── README.md                    # "Moving the Design System" → "Developing against a local Design System checkout"
└── CHANGELOG.md                 # 0.1.1: delivers design_system 0.1.1 from its own repository
```

Then for `0.2.0`, which is marked breaking:
- `ref: v0.2.0`;
- the example and tests migrated (`DsSpacing.md` → `context.dsSpacing.md` or `DsGap`; `ambiguous_exports_test.dart`);
- a CHANGELOG entry that names the Design System's migration notes.

**Container (this repository)**:
- `design-system` submodule initialized at `v0.1.1`, then bumped to `v0.2.0`;
- `foundations` submodule bumped;
- spec documents.

**Structure Decision**:
- The package stays at the repository root, so Foundations' git dependency needs no `path:`, and the package keeps the 0.1.0 name and library (FR-002).
- The Widgetbook and the lint plugin are sibling projects in sub-folders, like an `example/`. Each has its own `pubspec.yaml` and analysis context, and the root `analysis_options.yaml` excludes them.
- `widgetbook/brands/` is the only brands folder, because Flutter assets must live inside the declaring project. The package's tests read it by file path ([R9](research.md#r9-widgetbook-project)).
- Components move from `lib/src/{atoms,molecules,templates}` to `lib/src/components/<level>/`, so the lint rules can scope component-only rules by path.
- The interim `example/` gallery remains in `0.1.1` and is removed in `0.2.0` (the Widgetbook replaces it, FR-034).

## Delivery Phases

These are inputs to `/speckit-tasks`:

1. **Move (`0.1.1`)**:
   - Subtree split, then merge into the remote `main`, then add the scaffolding (README, `.gitignore`, `CLAUDE.md` draft), then tag.
   - Foundations `0.1.1` switches its dependency.
   - The container submodule is initialized.

   Pushing to the Design System remote and tagging are outward-facing steps. They need the user's go-ahead at implementation time.
2. **Brand core**:
   - the brand model and white label;
   - the parser and the problem report;
   - `DsTokens`, the semantic mapping, `DsTheme`, `DsBrandScope`, and the `context.ds*` getters;
   - the white-label fallback.
3. **Components**: move them to `components/<level>/`; read everything from tokens; add `info`, `qaIdentifier`, the guidelines frame, and `DsGap`. The accessibility harness iterates brands × modes.
4. **Contrast gate**, plus debug warnings.
5. **Golden tests**: comparator, fonts, the image matrix, and the Linux check.
6. **Widgetbook**: project, addons, brands folder, brand-file use cases, and the coverage test. The gallery is removed.
7. **Lints**: plugin package, rule tests, and the root opt-in with 0 reports.
8. **Lifecycle**:
   - the catalog and architecture checks;
   - the deprecated `DsBrand.standard` alias;
   - the CHANGELOG `0.2.0` entry with a Migration section;
   - `supportedSchemaVersions`.
9. **Agent**: `CLAUDE.md`, the checklist, `tool/check.sh`, `.claude/agents/ds-component-builder.md`, and an agent trial run (SC-014).
10. **Release `0.2.0`**: tag; Foundations `0.2.0` migration; the container bumps both submodules.

## Cross-Repository Dependencies and Risks

| Item | Impact | Mitigation |
|------|--------|-----------|
| The Design System remote has one existing commit (an empty README). | A plain push of the split branch is rejected, and a force-push would discard the remote's commit. | Merge with `--allow-unrelated-histories` locally, then push normally ([R1](research.md#r1-repository-move-and-history)). Pushing and tagging wait for the user's confirmation. |
| `0.2.0` is breaking for every consumer that reads spacing, radius, or `DsColors` through `Theme.of(context).extension`. | Foundations' example and tests, and any micro app code, must migrate. | The separate `0.1.1` move release keeps the move free for consumers. The `0.2.0` Migration section lists every change, and `DsBrand.standard` stays as a deprecated alias for one minor release. The other micro app repositories are still empty, so today only Foundations migrates. |
| The analyzer plugin's `git:` source is undocumented. | Micro apps in other repositories might not be able to opt in through git. | Quickstart V8 verifies `git:`. The fallback is `path: ../design-system/lints` from the container checkout, which is where micro apps are developed ([R11](research.md#r11-token-lint-rules)). |
| `analysis_server_plugin` 0.3.x and `analyzer` move in lockstep with the SDK. | An SDK upgrade can break the plugin. | Pin exact versions (0.3.26 / 14.7.0 for Dart 3.13), and upgrade them together with the SDK. The lints are dev-only, so a broken plugin never affects runtime. |
| Widgetbook APIs (manual node classes, `ObjectDropdownField` label builder, viewport names) are not shown in the docs used here. | Small signature differences during implementation. | Pin `3.25.0` exactly, and confirm the signatures against the package source when it is first added ([R9](research.md#r9-widgetbook-project)). |
| A 5% tolerance can hide real changes in large images. | Regressions slip through. | One component in one state per image, tight crops, and DPR 2.0. A deliberate color change must fail (quickstart V6, SC-009). |
| Text rasterization differs between macOS and Linux. | False golden failures. | Committed test fonts and the 5% threshold, with a Docker run on Linux as the check. If a specific image still differs by more than 5%, crop it tighter. The threshold stays at 5% unless the user decides otherwise. |
| Brand files from design tools may use color spaces or units outside the supported set. | A valid export is rejected. | Fall back to the file's own `hex`, convert `rem` at 16 px, and give a precise problem report with the token path. `doc/brand-files.md` tells designers which export settings to use. |
| The component agent's quality depends on the conventions written down for it. | The agent's output drifts from the conventions. | The agent and human reviewers use the same `doc/component-checklist.md`, and the agent must run `tool/check.sh` before it reports. SC-014 is checked with a trial run. |

## Complexity Tracking

| Exception | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| A debug-only static `Set<Object>` that de-duplicates warnings (G3) | FR-028, FR-032, and FR-048 ask for one warning per app run or per brand, which needs memory | Warning on every build floods the console. The set lives only inside `assert` bodies, is removed in release builds, and holds no product state ([R7](research.md#r7-debug-only-warnings-warn-once)). |
