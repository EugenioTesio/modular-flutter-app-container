# Implementation Plan: Foundations Package

**Branch**: `001-foundations-package` (spec-kit feature id; git is still on `main`) | **Date**: 2026-11-19 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-foundations-package/spec.md`

## Summary

This plan builds `foundations`, the shared Flutter package that every micro app, Core, and every host depends on. It provides three things of its own:

1. **Inter-module communication.** Endpoint, response, handler, UI-factory types and the communication interface, using the names from the reference article but without its global `instance` accessor. It also publishes the four statuses and the shared error codes.
2. **Eleven capability contracts.** `AppInfo`, `ApiClient`, `Analytics`, `CrashReporter`, `AppLogger`, `FeatureFlags`, `KeyValueStore`, `SecureKeyValueStore`, `AppNavigator`, `SessionContext`, and `DevicePermissions`. Each is an `abstract interface class`.
3. **A uniform `MicroApp` integration contract.** It adapts the main idea of `flutter_enterprise_architecture` without tenants ([research R20](research.md#r20-adapting-flutter_enterprise_architecture)). A micro app receives its own `MicroAppContext`, whose storage is limited to its keys, and *declares*:
   - the storage keys it owns (`storedKeys`);
   - its routes, as a router-neutral route tree (names, URL paths, groups, presentation, and the sign-in requirement) that Core turns into the real router;
   - its localization delegates;
   - class-based data and UI endpoint providers (the counterpart of the earlier package's brokers);
   - `initialize` and `dispose` steps.

   The host registers every provider in one `registerProviders` call. The communication contract also gains `isRegistered` and an optional per-call `traceName`.

Foundations also re-exports two first-party packages unchanged, each at one pinned version:

- the `design_system` package. For now it is an **interim** package with a minimum set of components and tokens, living at `packages/design_system/` inside the Foundations repository, and it will move to its own repository later ([R3](research.md#r3-design-system-dependency-and-re-export), [R21](research.md#r21-interim-design-system-scope));
- `switchable_state` from pub.dev, the shared state-management contract.

The host, not Foundations or any micro app, chooses the `switchable_state` adapter (Cubit, GetX, MobX, Provider, or Riverpod).

Foundations holds no global state and has no third-party runtime dependencies. It contains no dispatch logic: Core implements the contracts and provides the initialization. To keep Core and standalone stubs consistent, Foundations ships a conformance check, `package:foundations/testing.dart`, that is independent of any test framework.

## Technical Context

**Language/Version**: Dart 3.13 (`sdk: ^3.13.0`) and Flutter 3.47 stable (`flutter: ">=3.47.0"`).

**Primary Dependencies**:

- Runtime:
  - the Flutter SDK (`flutter`);
  - `design_system`, for now a `path: packages/design_system` dependency within the same repository. Pub resolves it from the same Foundations commit for git consumers (verified; [research R3](research.md#r3-design-system-dependency-and-re-export)). After the move it becomes a git dependency pinned to a release tag.
  - `switchable_state`, hosted on pub.dev and pinned exactly to `0.1.0` ([research R19](research.md#r19-state-management-contract-switchable_state)).
- Development:
  - `flutter_test` (SDK) and `flutter_lints`;
  - `yaml`, used only by the architecture tests;
  - two `switchable_state` adapters, `switchable_state_riverpod` and `switchable_state_bloc`, used only to run the sample micro app's widget tests under two libraries (SC-013);
  - `go_router`, used only by a reference router adapter in `test/support/` that builds a real router from the route contract (SC-015). It is never a runtime dependency.

**Storage**: N/A. Foundations defines the storage contracts only; Core supplies the implementations.

**Testing**:

- `flutter test` for value types, the conformance check (run against a correct reference implementation and deliberately broken variants), and the architecture rules (allowed dependencies, no global state).
- The example package runs the same conformance check against its in-memory implementation.

**Target Platform**: Every platform Flutter supports. The package contains no platform-specific code.

**Project Type**: Flutter library package (not a plugin), with a runnable `example/` app.

**Performance Goals**: Not a driver. Foundations has no dispatch or I/O logic. The conformance check should finish in under 1 second.

**Constraints**:

- 0 runtime dependencies other than `flutter`, `design_system`, and `switchable_state` (FR-003, SC-001).
- 0 adapter packages or concrete state-management libraries at runtime (FR-043).
- The interim `design_system` package depends on `flutter` only and never imports `foundations` (FR-056). Its components meet the accessibility rules D1–D4 in both themes (FR-055).
- 0 global access points and 0 global mutable state (FR-016, SC-012).
- 0 vendor types in the public surface (FR-004).
- 100% of public members documented (FR-032, SC-006).
- 0 analyzer issues.

**Scale/Scope**:

- About 14 interfaces (11 capabilities, communication, and two provider types) and about 35 public types.
- Consumers: 3 micro apps today (Home, Accounts, Transfers), plus Core, the App Shell, and 3 standalone examples.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` is still the unfilled template, so there are no ratified principles to enforce. Until a constitution exists, the plan uses the spec's own architecture rules as provisional gates:

| # | Provisional gate (source) | Pre-research | Post-design |
|---|---------------------------|--------------|-------------|
| G1 | Foundations depends only on the Flutter SDK, `design_system`, and `switchable_state`. It never depends on Core, the App Shell, micro apps, vendor SDKs, adapter packages, or state libraries (FR-003, FR-043) | PASS | PASS: `pubspec.yaml` lists only those three runtime dependencies, and an architecture test enforces this. The adapters are development-only dependencies ([quickstart V1](quickstart.md#v1--package-boundary-sc-001-sc-012)) |
| G2 | The public surface contains no vendor types (FR-004) | PASS | PASS: every contract in [contracts/](contracts/) uses only SDK types and Foundations types |
| G3 | No global access points and no global state (FR-016) | PASS | PASS: contracts arrive through `MicroAppContext`, and there is no `static` mutable state ([R7](research.md#r7-delivering-contracts-without-globals)) |
| G4 | No feature models and no concrete capability implementations (FR-005) | PASS | PASS: `lib/` holds only interfaces, value types, and constants. The reference implementation lives in `example/` and `test/support/` |
| G5 | The Design System and `switchable_state` are re-exported unchanged, never wrapped (FR-002, FR-042) | PASS | PASS: one `export` directive for each package ([contracts/public-surface.md](contracts/public-surface.md)) |
| G6 | Everything public is documented and the analyzer reports nothing (FR-032, SC-006) | PASS | PASS: the `public_member_api_docs` lint is enabled and analysis must report 0 issues ([R16](research.md#r16-quality-tooling)) |

There are no violations, so the Complexity Tracking table is left empty.

**Recommendation**: Ratify a constitution (`/speckit-constitution`) before the next feature, so these gates become binding for Core, the App Shell, and the micro apps.

## Project Structure

### Documentation (this feature)

```text
specs/001-foundations-package/
├── plan.md              # This file
├── research.md          # Phase 0: decisions and rationale
├── data-model.md        # Phase 1: entities, fields, validation, states
├── quickstart.md        # Phase 1: runnable validation scenarios
├── contracts/           # Phase 1: public API contracts
│   ├── public-surface.md
│   ├── inter-module-communication.md
│   ├── capabilities.md
│   ├── routing.md
│   ├── design-system.md
│   ├── micro-app.md
│   └── conformance.md
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks; not created here)
```

### Source Code (the `foundations/` submodule, repo `modular-flutter-app-foundations`)

```text
foundations/
├── pubspec.yaml                     # name: foundations, publish_to: none
│                                    #   deps: flutter, design_system (path: packages/design_system), switchable_state 0.1.0
│                                    #   dev: flutter_test, flutter_lints, yaml, switchable_state_riverpod, switchable_state_bloc, go_router
├── analysis_options.yaml            # flutter_lints + public_member_api_docs + strict modes
├── CHANGELOG.md                     # each release states the delivered design_system version
├── README.md
├── lib/
│   ├── foundations.dart             # main entry point: every contract plus the design_system and switchable_state re-exports
│   ├── testing.dart                 # conformance check (no test framework dependency)
│   └── src/
│       ├── common/
│       │   └── result.dart          # sealed Result<T, F> (Ok / Err)
│       ├── inter_module/
│       │   ├── inter_module_endpoint.dart
│       │   ├── inter_module_response.dart
│       │   ├── inter_module_status.dart       # status codes and shared error codes
│       │   ├── inter_module_value.dart        # payload rule (primitives/lists/maps)
│       │   ├── inter_module_communication.dart
│       │   └── endpoint_provider.dart         # DataEndpointProvider, WidgetEndpointProvider, registerProviders
│       ├── exceptions/                    # every exception lives here (user convention)
│       │   ├── duplicate_endpoint_exception.dart
│       │   ├── api_exception.dart             # ApiException, ApiExceptionType
│       │   ├── storage_exception.dart         # StorageException, StorageExceptionType
│       │   └── navigation_exception.dart      # sealed NavigationException, RouteNotFoundException
│       ├── capabilities/
│       │   ├── app_info.dart
│       │   ├── api_client.dart
│       │   ├── analytics.dart
│       │   ├── crash_reporter.dart
│       │   ├── app_logger.dart
│       │   ├── feature_flags.dart
│       │   ├── key_value_store.dart           # KeyValueStore + SecureKeyValueStore
│       │   ├── session_context.dart
│       │   └── device_permissions.dart        # DevicePermissions, DevicePermission variants, statuses
│       ├── routing/
│       │   ├── micro_app_route.dart           # MicroAppRouteNode, MicroAppRoute, MicroAppRouteGroup, RoutePresentation
│       │   └── app_navigator.dart             # AppNavigator (push/go/replace/pop/popUntil), NavigationResult
│       ├── micro_app/
│       │   ├── micro_app.dart                 # MicroApp (id, storedKeys, routes, providers, initialize, dispose)
│       │   └── micro_app_context.dart
│       └── testing/
│           └── inter_module_conformance.dart
├── test/
│   ├── architecture/                # dependency rules, no global state, ambiguous exports
│   ├── inter_module/                # endpoint, response, status, payload rule
│   ├── capabilities/                # Result, flags, API exception types, navigation, session, device permissions
│   ├── micro_app/                   # context construction; minimal reference host built only on the contracts:
│   │                                #   storedKeys check → per-app scoped contexts → initialize → route checks
│   │                                #   → registerProviders → dispose; provider unit tests
│   ├── routing/                     # route-tree validation rules, NavigationResult matching, a reference
│   │                                #   router adapter in test/support that proves the contract is buildable
│   ├── state_management/            # sample micro app state: same widget tests under Riverpod and Cubit adapters
│   ├── testing/                     # conformance vs. correct and broken implementations
│   └── support/                     # reference + mutant implementations, fakes
├── example/
│   ├── pubspec.yaml                 # depends on foundations via path: ../
│   ├── lib/
│   │   ├── main.dart                # Module A/B data demo + Home/Cards UI demo (built with Ds components)
│   │   └── example_inter_module_communication.dart
│   └── test/                        # conformance run + every-outcome scenarios (FR-035) + re-export test
└── packages/
    └── design_system/               # INTERIM: its own package; moves to modular-flutter-app-design-system later
        ├── pubspec.yaml             # name: design_system, version 0.1.0, publish_to: none, deps: flutter ONLY
        ├── CHANGELOG.md
        ├── analysis_options.yaml    # same rules as Foundations
        ├── lib/
        │   ├── design_system.dart   # single public library (name kept after the move)
        │   └── src/
        │       ├── tokens/          # ds_palette, ds_spacing, ds_radius, ds_colors, ds_typography
        │       ├── theme/           # ds_brand, ds_theme (light/dark), ds_theme_context
        │       ├── atoms/           # ds_button, ds_text_field, ds_loading_indicator
        │       ├── molecules/       # ds_card, ds_list_tile, ds_inline_message, ds_empty_state
        │       └── templates/       # ds_page_scaffold
        ├── test/
        │   ├── tokens/              # contrast ratios per theme, brand derivation
        │   ├── components/          # D1–D9 per component, light + dark, 200% text scale
        │   └── architecture/        # deps allow-list {flutter}; no import of package:foundations
        └── example/                 # gallery: every component and state, light/dark and brand toggles
```

**Structure Decision**: The code goes in the existing `foundations` submodule as one Flutter library package with an `example/` app. The specs stay in this container repo. The code is split into `inter_module/`, `capabilities/`, `routing/`, and `micro_app/`, matching the spec's areas. The Design System and `switchable_state` are each delivered by one `export` in `lib/foundations.dart`, and neither is copied into `src/`. The interim Design System is a **separate package** under `packages/design_system/`, with its own manifest, tests, and gallery, so moving it later is a relocation, not a refactor.

## Cross-Repository Dependencies and Risks

| Item | Impact | Mitigation |
|------|--------|-----------|
| The `design-system` repo is empty (README only). | It would block User Story 4. | **Resolved for now**: an interim `design_system` package with the minimum set (FR-054) lives at `foundations/packages/design_system/`, and Foundations depends on it by relative path. User Story 4 is no longer blocked. |
| The interim Design System must move to its own repository later. | If it isn't kept separate, the move turns into a refactor that touches micro apps. | It has Flutter-only dependencies, never imports Foundations (an architecture test checks this), and keeps its package name and public library. The move is `git subtree split`, tag, and a one-line dependency change in Foundations, with 0 micro app changes (FR-056, SC-018; [R3](research.md#r3-design-system-dependency-and-re-export)). |
| Git consumers of Foundations rely on pub resolving the nested path dependency. | If pub stopped supporting this, consumers couldn't resolve `design_system`. | Verified on Dart 3.13.7 (pub resolves it from the same commit). V6 includes a git-consumer smoke check so a regression is caught. |
| Core, which implements the contracts, does not exist yet. | Nothing outside Foundations exercises the contracts yet. | The `example/` reference implementation and the conformance check prove the contracts in isolation. Core's spec must require Core to pass `testing.dart`. |
| `switchable_state` is new: 0.1.0 is its only release. | Breaking changes before 1.0 would become breaking Foundations releases (FR-045). | Pin exactly `0.1.0`, and upgrade only on purpose, with a changelog entry. Hosts choose an adapter version that accepts the pinned version. Core or the host runs `switchable_state_test` against its chosen adapter. |
| The adapters pull in vendor libraries: `flutter_riverpod ^3.5.0`, `flutter_bloc ^9.1.1`, `provider ^6.1.5+1`, `get ^4.7.3`, and `mobx ^2.7.0` with `flutter_mobx ^2.4.0`. | A runtime adapter dependency would bring a vendor library into Foundations. | Adapters stay `dev_dependencies` in Foundations, used only for the SC-013 tests. All five adapters and `switchable_state_test` are on pub.dev at 0.1.0, accept `switchable_state ^0.1.0`, and need SDK `^3.13.0` and Flutter `>=3.47.0`, the same as Foundations (checked on 2026-11-20). |
| The registration lifecycle (deferred in clarify). | Core's registries might accept late registrations. | Resolved by design: micro apps declare providers, and the host registers them once, after every `initialize` ([R8](research.md#r8-endpoint-registration-lifecycle)). Core rejects later `register` calls as a safety net, and its spec should say so. |
| The reusable host assembly (the counterpart of `AppShellContainer`) must not be duplicated in the App Shell and every example. | Each host could drift in ordering, duplicate checks, or route guarding. | Assign it to Core's shared startup in Core's spec: check `storedKeys` and ids, build per-app scoped contexts, initialize, collect, check full paths, `registerProviders`, build the router from the route contract, guard routes, and dispose in reverse order. Foundations' tests include a minimal host that follows the same sequence and serves as the reference. |

## Complexity Tracking

No constitution violations, so there is nothing to justify.
