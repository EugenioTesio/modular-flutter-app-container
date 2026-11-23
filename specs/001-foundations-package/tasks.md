---

description: "Task list for the Foundations package"
---

# Tasks: Foundations Package

**Input**: Design documents from `specs/001-foundations-package/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: The spec asks for tests in FR-034, FR-017, FR-055, SC-003, SC-014, and SC-017, so every story phase starts with test tasks. Write them first, and make sure they fail before you implement.

**Test notation (required, user convention)**: Every test is described in Gherkin notation.
- A file starts with a `// Feature: …` comment that names the rule or story and its spec references.
- `group('Given …')` holds the shared context.
- Each `test('When …, Then …')` (or a single `test('Given …, When …, Then …')`) states one behavior.
- The test body is split into `// Given`, `// When`, and `// Then` sections.
- Acceptance scenarios from spec.md keep their Given/When/Then wording.

**Code conventions (required, user conventions)**:
- Exceptions are named `…Exception`, `implements Exception`, and live in `lib/src/exceptions/` (one file each), for example `DuplicateEndpointException`.
- Use "type" rather than "kind" for classifications the project introduces, for example `EndpointType`, `isRegistered({url, type})`, `ApiExceptionType` with `ApiException.type`, and `StorageExceptionType` with `StorageException.type`.
- Scope names to their feature (`AppInfoEnvironment`, `BoolFeatureFlag`, `DevicePermissionStatus`). Group helpers as static methods, and prefer SDK helpers (`mapEquals`) over custom ones.
- **Public API documentation**: every declaration micro apps can use gets a proper dartdoc comment in the same task that adds it, in the shape the root `CLAUDE.md` describes (taken from the user's reference classes):
  - Types: a role-noun summary, "Use this…", "This is **not**…", "Built-in:", and a "Usage from a micro app:" label above a ` ```dart ` block.
  - Constructors: "Creates a…" plus a `Parameters:` list of `* [name] - …` lines.
  - Methods: a verb summary, `Parameters:`, then returns and failures.
  - Fields: noun phrases, with "Optional…" for nullable ones.
  - Use `[links]` via `@docImport`, no `@param` tags, and no spec IDs.
  
  Lints and `dart doc --dry-run` enforce it. When `MicroApp` lands (T059), turn the backticked `MicroApp.storedKeys`, `dataEndpoints`, and `widgetEndpoints` mentions in existing docs into links.

**Organization**: Tasks are grouped by user story so each story can be built and tested on its own.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task)
- **[Story]**: The user story the task belongs to (US1–US6, from spec.md)
- Every task names its exact file paths

## Path Conventions

- All paths are relative to the container repository root.
- Foundations code lives in the `foundations/` git submodule (repo `modular-flutter-app-foundations`), on branch `001-foundations-package` (research R2).
- The interim Design System is a separate package at `foundations/packages/design_system/` (research R3, R21).
- Field constraints are cited as `data-model.md Entity.field`. The data model is the canonical source, so the constraints are not repeated here.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create the three Flutter packages (Foundations, its example app, and the interim Design System), lint rules, and CI.

- [X] T001 Initialize the submodule and create the work branch: run `git submodule update --init foundations`, then `git -C foundations switch -c 001-foundations-package`, in `foundations/`
- [X] T002 Create the interim Design System package skeleton in `foundations/packages/design_system/`:
  - `foundations/packages/design_system/pubspec.yaml`: name `design_system`, `version: 0.1.0`, `publish_to: none`, SDK `^3.13.0`, Flutter `>=3.47.0`, runtime dependency `flutter` only, dev dependencies `flutter_test` and `flutter_lints`.
  - `foundations/packages/design_system/lib/design_system.dart`: the library doc comment only.
  - Follow the package rules in contracts/design-system.md.
- [X] T003 Create `foundations/pubspec.yaml` per the plan.md Technical Context: name `foundations`, `publish_to: none`, SDK `^3.13.0`, Flutter `>=3.47.0`.
  - Dependencies: `flutter`; `switchable_state: 0.1.0`, pinned exactly (research R19); `design_system: {path: packages/design_system}` (research R3).
  - Dev dependencies: `flutter_test`, `flutter_lints`, `yaml`, `switchable_state_riverpod`, `switchable_state_bloc`, and `go_router`.
- [X] T004 [P] Create `foundations/analysis_options.yaml` per research R16:
  - Include `package:flutter_lints/flutter.yaml`.
  - Add `public_member_api_docs`, `always_declare_return_types`, `prefer_final_fields`, and `avoid_dynamic_calls`.
  - Turn on `strict-casts`, `strict-inference`, and `strict-raw-types`.
  - Exclude `test/fixtures/**`.
- [X] T005 [P] Create `foundations/packages/design_system/analysis_options.yaml` with the same rule set as T004.
- [X] T006 [P] Create `foundations/.gitignore` covering `.dart_tool/`, `build/`, `coverage/`, and `pubspec_overrides.yaml` (research R3).
- [X] T007 Create the public entry points `foundations/lib/foundations.dart` and `foundations/lib/testing.dart`. Each gets a library doc comment that states its purpose per contracts/public-surface.md. Exports are added by later tasks as the files land.
- [X] T008 [P] Create the example app skeleton: `foundations/example/pubspec.yaml` (depends on `foundations` with `path: ../`, `publish_to: none`) and `foundations/example/lib/main.dart` (a minimal `MaterialApp` with one placeholder screen).
- [X] T009 [P] Create the Design System gallery skeleton: `foundations/packages/design_system/example/pubspec.yaml` (depends on `design_system` with `path: ../`) and `foundations/packages/design_system/example/lib/main.dart` (a minimal placeholder app).
- [X] T010 [P] Create the CI workflow `foundations/.github/workflows/ci.yaml`.
  - For `foundations`, `foundations/example`, and `foundations/packages/design_system`, run `flutter pub get`, `flutter analyze` (which must report 0 issues), and `flutter test`.
  - Run `flutter test --coverage` for `foundations`.
- [X] T011 Run `flutter pub get` in `foundations/`, `foundations/packages/design_system/`, and `foundations/example/`. Confirm three things:
  - `switchable_state` resolves to exactly 0.1.0.
  - `design_system` resolves from `packages/design_system`.
  - The adapters and `go_router` appear only as dev dependencies.

  This depends on T002, T003, T008, and T009.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The architecture guardrails and the shared payload rule that every story relies on.

**⚠️ CRITICAL**: No user story work starts until this phase is complete.

- [X] T012 [P] Write the dependency allow-list test in `foundations/test/architecture/dependency_rules_test.dart`. It parses `foundations/pubspec.yaml` with `package:yaml` and asserts:
  - The runtime `dependencies` are a subset of `{flutter, design_system, switchable_state}`, and `switchable_state` is pinned exactly to `0.1.0`.
  - Every `switchable_state_*`, `flutter_riverpod`, `flutter_bloc`, `provider`, `get`, `mobx`, `flutter_mobx`, and `go_router` entry appears only under `dev_dependencies`.

  This covers FR-003, FR-043, FR-053, and SC-001.
- [X] T013 [P] Write the no-global-state scan in `foundations/test/architecture/no_global_state_test.dart`. It fails on any `static` field under `foundations/lib/` that is not `const`, and on any top-level `var` or `late` declaration (FR-016, SC-012).
- [X] T014 [P] Write the Design System package-rules test in `foundations/packages/design_system/test/architecture/package_rules_test.dart`. It asserts that the runtime dependencies are exactly `{flutter}` and that no file under `lib/` imports `package:foundations` (FR-056).
- [X] T015 [P] Write the payload-rule tests in `foundations/test/inter_module/inter_module_value_test.dart`. Cover the accepted and rejected values from the data-model.md "Shared rules → Payload rule": nested lists and maps, non-`String` map keys, and custom objects.
- [X] T016 Implement `isInterModuleValue(Object?)` in `foundations/lib/src/inter_module/inter_module_value.dart` per the data-model.md payload rule, and export it from `foundations/lib/foundations.dart`. T015 must pass.

**Checkpoint**: The guardrails are in place, so story work can begin.

---

## Phase 3: User Story 1 - Request data from another micro app without importing it (Priority: P1) 🎯 MVP

**Goal**: The inter-module communication contract (endpoint, response, statuses, five operations, endpoint providers), plus the conformance check covering data and registration, proven against a reference implementation and a runnable example.

**Independent Test**: `flutter test test/inter_module test/testing` in `foundations/`, and `flutter test test/inter_module_data_test.dart` in `foundations/example/`. These cover quickstart.md V3 and the data rows of V4.

### Tests for User Story 1 ⚠️

- [X] T017 [P] [US1] Write the endpoint value tests in `foundations/test/inter_module/inter_module_endpoint_test.dart`. Cover the defaults, value equality including content equality of maps and sets, and every field: data-model.md `InterModuleEndpoint.url`, `InterModuleEndpoint.parameters`, `InterModuleEndpoint.requiredParameters`, and `InterModuleEndpoint.traceName`.
- [X] T018 [P] [US1] Write the response tests in `foundations/test/inter_module/inter_module_response_test.dart`.
  - `isSuccess` at the boundaries: 199, 200, 299, and 300.
  - `.success` carries non-null `data`, and `.failure` asserts a status outside 2xx.
  - Cite data-model.md `InterModuleCommunicationResponse.statusCode`, `InterModuleCommunicationResponse.data`, `InterModuleCommunicationResponse.errorCode`, and `InterModuleCommunicationResponse.isSuccess`.
- [X] T019 [P] [US1] Write the status and error-code tests in `foundations/test/inter_module/inter_module_status_test.dart`. They assert the exact published values from the data-model.md "Shared error codes" table and the four status constants.
- [X] T020 [P] [US1] Write the data-provider tests in `foundations/test/inter_module/endpoint_provider_test.dart`:
  - A sample `DataEndpointProvider` for `/module-b/v1/details` is exercised by calling `handle` directly, with no communication implementation (SC-014).
  - Using a recording fake, `registerProviders(data: …)` calls `register` with each `endpoint` and passes `DuplicateEndpointException` through.
- [X] T021 [P] [US1] Write the conformance self-test in `foundations/test/testing/inter_module_conformance_test.dart`. It runs cases C01–C11 and C17 from contracts/conformance.md against the reference implementation, where every case must pass. It also runs them against each broken variant, where the case that targets that variant's defect must fail.

### Implementation for User Story 1

- [X] T022 [P] [US1] Implement `InterModuleEndpoint` in `foundations/lib/src/inter_module/inter_module_endpoint.dart` per contracts/inter-module-communication.md. Cite data-model.md `InterModuleEndpoint.url`, `InterModuleEndpoint.parameters`, `InterModuleEndpoint.requiredParameters`, and `InterModuleEndpoint.traceName`.
- [X] T023 [P] [US1] Implement `InterModuleCommunicationResponse<T>` with `.success` and `.failure` in `foundations/lib/src/inter_module/inter_module_response.dart`. Cite data-model.md `InterModuleCommunicationResponse.statusCode`, `InterModuleCommunicationResponse.data`, `InterModuleCommunicationResponse.errorCode`, and `InterModuleCommunicationResponse.isSuccess`.
- [X] T024 [P] [US1] Implement `InterModuleStatusCode` and `InterModuleErrorCode` in `foundations/lib/src/inter_module/inter_module_status.dart`, following the data-model.md "Shared error codes" table.
- [X] T025 [US1] Implement the `InterModuleCommunication` interface in `foundations/lib/src/inter_module/inter_module_communication.dart`, with `register`, `request`, `registerWidget`, `getWidget`, and `isRegistered`, the `InterModuleHandler` and `InterModuleWidgetFactory` typedefs, and `EndpointType`.
  - Document behavior rows B1–B14 from contracts/inter-module-communication.md, including that micro apps never call `register` directly.
  - Cite data-model.md "Registration": `EndpointType`, `InterModuleHandler`, and `InterModuleWidgetFactory`.
  - Depends on T022 and T023.
- [X] T026 [US1] Implement `DuplicateEndpointException` in `foundations/lib/src/exceptions/duplicate_endpoint_exception.dart`. Cite data-model.md `DuplicateEndpointException.url` and `DuplicateEndpointException.kind`. Depends on T025.
- [X] T027 [US1] Implement `DataEndpointProvider`, `WidgetEndpointProvider`, and the `InterModuleProviderRegistration.registerProviders` extension in `foundations/lib/src/inter_module/endpoint_provider.dart`. Cite data-model.md `DataEndpointProvider.endpoint` and `WidgetEndpointProvider.endpoint`. Depends on T025 and T026.
- [X] T028 [US1] Create the reference implementation and the broken variants in `foundations/test/support/`:
  - `foundations/test/support/reference_inter_module_communication.dart`: a correct, in-memory implementation of every contract behavior row, B1–B13 (B14 is implementation diagnostics).
  - `foundations/test/support/broken_inter_module_communications.dart`, with these variants: one ignores required parameters, one rethrows handler errors, one shares a single registry between kinds, one leaks exception text into `errorCode`, and one accepts duplicates.
  - Depends on T025–T027.
- [X] T029 [US1] Implement `ConformanceResult`, `ConformanceCase`, and `InterModuleConformance.cases`, with C01–C11 and C17, in `foundations/lib/src/testing/inter_module_conformance.dart` per contracts/conformance.md.
  - Each run calls `create()` for a fresh implementation.
  - Cite data-model.md `ConformanceCase.id`, `ConformanceCase.description`, `ConformanceCase.run`, `ConformanceResult.passed`, and `ConformanceResult.message`.
  - Depends on T025 and T026.
- [X] T030 [US1] Export the inter-module files from `foundations/lib/foundations.dart`, and export the conformance file from `foundations/lib/testing.dart`, per contracts/public-surface.md. T017–T021 must pass.
- [X] T031 [US1] Build the example data demo in `foundations/example/lib/`:
  - `foundations/example/lib/example_inter_module_communication.dart`: an in-memory implementation.
  - `foundations/example/lib/module_b.dart`: the `/module-b/v1/details` provider. It returns 400 `invalid_id`, 404 `details_not_found`, 400 `unauthenticated` or `forbidden`, and 500 for a throw.
  - `foundations/example/lib/main.dart`: a Module A screen with one result row per data outcome in quickstart.md V4. It uses Material widgets for now; T081 restyles it.
- [X] T032 [US1] Write the example data tests in `foundations/example/test/inter_module_data_test.dart`. Run C01–C11 and C17 against the example implementation, plus every data row of the quickstart.md V4 table. In each, the consumer tells the outcome apart using only the status and the error code (SC-008).

**Checkpoint**: User Story 1 is the MVP. Teams can publish and consume data endpoints against any implementation that passes the conformance check.

---

## Phase 4: User Story 2 - Consume platform capabilities through application-owned abstractions (Priority: P2)

**Goal**: The eleven capability contracts, including `AppNavigator` and its outcomes. All of them are vendor-neutral and can be faked headless.

**Independent Test**: `flutter test test/common test/capabilities test/routing/navigation_result_test.dart` in `foundations/`. A sample feature that uses every capability runs with in-memory fakes only, covering quickstart.md V5 for capabilities.

### Tests for User Story 2 ⚠️

- [X] T033 [P] [US2] Write the `Result` tests in `foundations/test/common/result_test.dart`: an exhaustive `switch` over `Ok` and `Err`, and value equality.
- [X] T034 [P] [US2] Write the API client tests in `foundations/test/capabilities/api_client_test.dart`:
  - The request defaults and equality.
  - Each `ApiClientShortcuts` method calls `send` with the right `ApiMethod`, `path`, `queryParams`, `headers`, and `body` (using a recording fake).
  - An exhaustive `switch` over `ApiExceptionType`.
  - Cite data-model.md `ApiRequest.path`, `ApiRequest.body`, `ApiException.type`, and `ApiResponse.statusCode`.
- [X] T035 [P] [US2] Write the feature-flag tests in `foundations/test/capabilities/feature_flags_test.dart`:
  - The typed `BoolFeatureFlag`, `StringFeatureFlag`, and `NumberFeatureFlag` definitions.
  - A fake that returns the default for unknown flags and for flags of the wrong kind.
  - `changes` emits after a refresh.
  - Cite data-model.md `FeatureFlag.key` and `FeatureFlag.defaultValue`.
- [X] T036 [P] [US2] Write the logger tests in `foundations/test/capabilities/app_logger_test.dart`. `debug`, `info`, `warning`, and `error` must call `log` with the matching data-model.md `LogLevel`, passing `error`, `stackTrace`, and `context` through.
- [X] T037 [P] [US2] Write the navigation outcome tests in `foundations/test/routing/navigation_result_test.dart`: an exhaustive `switch` over `Result<NavigationResult, NavigationException>`, `RouteNotFoundException.path`, and value equality. Cite data-model.md `NavigationResult` and `NavigationException`. *(Revised 2026-11-23.)*
- [X] T038 [P] [US2] Write the value-type tests in `foundations/test/capabilities/value_types_test.dart`:
  - An exhaustive `switch` over `SessionState`.
  - `UserIdentity` equality, including `customProperties` compared key by key, and a `toString` that hides personal data.
  - An exhaustive `switch` over every `DevicePermission` variant, the defaults and equality of the variants with options, every `DevicePermissionStatus` (including `unsupported`), and every `DevicePermissionServiceStatus`.
  - Every `StorageExceptionType`, including `undeclaredKey`.
  - Every `AppInfoEnvironment` value.
  - Cite data-model.md `UserIdentity.userId`, `UserIdentity.customProperties`, `DevicePermission`, `StorageException.type`, and `AppInfo.environment`.
- [X] T039 [P] [US2] Create in-memory fakes for all eleven capabilities in `foundations/test/support/fakes.dart`, plus `foundations/test/capabilities/sample_feature_test.dart`, in which a sample feature that uses every capability runs with no device, network, or Core. Cover the US2 acceptance scenarios that Foundations can verify:
  - 1–2: the version comes from `AppInfo`, and the feature runs on fakes.
  - 4: no-op analytics, crash reporting, and logging don't change behavior.
  - 5–6: an explicit `Unauthenticated` state, and change notifications.
  - 9: an `unauthorized` API failure.
  - 10 and 14: the flag default, and flag change notifications.
  - 11: console logging.

  This covers SC-005 and SC-007.

### Implementation for User Story 2

- [X] T040 [P] [US2] Implement `Result`, `Ok`, and `Err` in `foundations/lib/src/common/result.dart`, following data-model.md "Result<T, F>".
- [X] T041 [P] [US2] Implement `AppInfo` and `AppInfoEnvironment` in `foundations/lib/src/capabilities/app_info.dart`. Cite data-model.md `AppInfo.appName`, `AppInfo.version`, `AppInfo.buildNumber`, and `AppInfo.environment`.
- [X] T042 [P] [US2] Implement `ApiMethod`, `ApiRequest`, `ApiResponse`, `ApiClient`, and `ApiClientShortcuts` in `foundations/lib/src/capabilities/api_client.dart`, and `ApiException` with `ApiExceptionType` in `foundations/lib/src/exceptions/api_exception.dart`.
  - Document the exception-type mapping table, and that credentials are never exposed (FR-039).
  - Cite data-model.md `ApiRequest.method`, `ApiRequest.path`, `ApiRequest.queryParams`, `ApiRequest.headers`, `ApiRequest.body`, `ApiResponse.statusCode`, `ApiResponse.body`, `ApiResponse.headers`, `ApiException.type`, `ApiException.statusCode`, and `ApiException.body`.
- [X] T043 [P] [US2] Implement `Analytics` in `foundations/lib/src/capabilities/analytics.dart`. Cite data-model.md `Analytics event.name`, `Analytics event.properties`, `Screen view.screenName`, and `Screen view.properties`.
- [X] T044 [P] [US2] Implement `CrashReporter` in `foundations/lib/src/capabilities/crash_reporter.dart`. Cite data-model.md `Error report.error`, `Error report.stackTrace`, `Error report.reason`, and `Error report.context`, plus the pseudonymous user identifier rule.
- [X] T045 [P] [US2] Implement `AppLogger`, `LogLevel`, and `AppLoggerShortcuts` in `foundations/lib/src/capabilities/app_logger.dart`. Document the rule that log entries contain no credentials or personal data. Cite data-model.md `LogLevel`, `Log entry.level`, `Log entry.message`, and `Log entry.context`.
- [X] T046 [P] [US2] Implement `FeatureFlags` (with `valueOf` and `changes`) and the sealed `FeatureFlag`, `BoolFeatureFlag`, `StringFeatureFlag`, and `NumberFeatureFlag` in `foundations/lib/src/capabilities/feature_flags.dart`. Cite data-model.md `FeatureFlag.key` and `FeatureFlag.defaultValue`.
- [X] T047 [P] [US2] Implement `KeyValueStore` and `SecureKeyValueStore` in `foundations/lib/src/capabilities/key_value_store.dart`, and `StorageException` with `StorageExceptionType` in `foundations/lib/src/exceptions/storage_exception.dart`.
  - Document that Core limits each micro app's stores to its declared keys.
  - Cite data-model.md `Storage entry.key`, `Storage entry.value`, `StorageException.type`, and `StorageException.key`.
- [X] T048 [P] [US2] Implement `SessionContext`, `SessionState`, `Authenticated`, `Unauthenticated`, and `UserIdentity` in `foundations/lib/src/capabilities/session_context.dart`. Cite data-model.md `SessionState`, `UserIdentity.userId`, `UserIdentity.displayName`, `UserIdentity.email`, `UserIdentity.phoneNumber`, and `UserIdentity.customProperties`, and the "Session" state transitions. No tokens (FR-039).
- [X] T049 [P] [US2] Implement `DevicePermissions` (`status`, `request`, `serviceStatus`, `openAppSettings`), the sealed `DevicePermission` with one variant per `permission_handler` permission (options enums for photos, storage, calendar, location, sensors, and Bluetooth), `DevicePermissionStatus`, and `DevicePermissionServiceStatus` in `foundations/lib/src/capabilities/device_permissions.dart`. Document that business entitlements are out of scope. Cite data-model.md "Device permissions". *(Revised 2026-11-23.)*
- [X] T050 [P] [US2] Implement `AppNavigator` (`push`, `go`, `replace`, `pop`, `popUntil`) along with `NavigationResult`, `RouteCompleted`, and `RouteCancelled` in `foundations/lib/src/routing/app_navigator.dart`, and the sealed `NavigationException` with `RouteNotFoundException` in `foundations/lib/src/exceptions/navigation_exception.dart`, per contracts/routing.md. Cite data-model.md "Navigation path", "Navigation body", `NavigationResult`, and `NavigationException`. *(Revised 2026-11-23.)*
- [X] T051 [US2] Export the T040–T050 files (including the three exception files) from `foundations/lib/foundations.dart`. T033–T039 must pass.

**Checkpoint**: Micro apps can use every capability through vendor-neutral contracts and fake them in tests.

---

## Phase 5: User Story 3 - Integrate any micro app into any host the same way (Priority: P2)

**Goal**: The `MicroApp` contract (`id`, `storedKeys`, the route tree, localizations, providers, `initialize` and `dispose`), a separate `MicroAppContext` per micro app, and the router-neutral route contract. All of it is proven by a reference host and a reference go_router adapter.

**Independent Test**: `flutter test test/micro_app test/routing` in `foundations/`, covering quickstart.md V5 for integration and V10.

**Depends on**: US1 (the communication interface and providers) and US2 (the capability interfaces that `MicroAppContext` bundles).

### Tests for User Story 3 ⚠️

- [ ] T052 [P] [US3] Write the route-tree validation tests in `foundations/test/routing/route_tree_validation_test.dart`.
  - Rejected: a duplicate full path, a top-level path without the `/<microAppId>` prefix, and a repeated path parameter.
  - Child paths join correctly.
  - The sign-in requirement is inherited from ancestors.
  - Cite data-model.md `MicroAppRoute.path`, `MicroAppRoute.requiresAuthentication`, `MicroAppRoute.routes`, and `MicroAppRouteGroup.routes`.
- [ ] T053 [P] [US3] Write the reference-router tests in `foundations/test/routing/reference_router_test.dart`, following quickstart.md V10 and covering SC-015.
  - The deep link `/accounts/details/42?tab=movements` and `push('/accounts/details/42?tab=movements', body: {...})` show the same screen; only the push delivers the body.
  - `Err(RouteNotFoundException)` is returned for a location that matches no route, and by `popUntil` for a path not in the stack.
  - Picker results come back as `Ok(RouteCompleted)` or `Ok(RouteCancelled)`, for both `push` and `replace`.
  - `go` returns `Ok` on success and replaces the stack. `popUntil` closes screens down to its path.
  - A route that needs sign-in redirects while the user is `Unauthenticated`.
  - A `fullscreenDialog` route is presented as a full-screen dialog.
  - Moving between routes in a group keeps the group's layout and its `StateScope.group` state.
- [ ] T054 [P] [US3] Write the context compile check in `foundations/test/micro_app/micro_app_context_test.dart`:
  - `foundations/test/micro_app/micro_app_context_test.dart` runs `dart analyze` on `foundations/test/fixtures/missing_contract_fixture.dart` and expects a `missing_required_argument` error.
  - The fixture builds a `MicroAppContext` with one contract missing.
  - Cite data-model.md "MicroAppContext", where all fields are required.
- [ ] T055 [P] [US3] Write the reference-host tests in `foundations/test/micro_app/reference_host_test.dart`, covering US3 scenarios 1–9 and SC-016.
  - Startup stops in each of these cases, with an error naming the culprit: a duplicate `id`; the same stored key declared by two micro apps (naming the key and both micro apps); a duplicate full path; a duplicate provider URL (`DuplicateEndpointException`).
  - Each micro app gets its own context, and its stores return `StorageExceptionType.undeclaredKey` for keys it didn't declare, leaving the other micro app's value untouched.
  - Micro apps initialize in order. `isRegistered` is true for every declared provider after startup.
  - `dispose` runs in reverse order and continues after one teardown throws.
  - Cite data-model.md `MicroApp.storedKeys` and the "Stored key" entity.
- [ ] T056 [P] [US3] Write the sample provider unit tests in `foundations/test/micro_app/sample_providers_test.dart`. They call each sample `DataEndpointProvider.handle` and `WidgetEndpointProvider.build` directly (SC-014).

### Implementation for User Story 3

- [ ] T057 [P] [US3] Implement `MicroAppRouteNode`, `MicroAppRoute`, `MicroAppRouteGroup`, `MicroAppRouteBuilder`, and `RoutePresentation` in `foundations/lib/src/routing/micro_app_route.dart`, per contracts/routing.md. Cite data-model.md `MicroAppRoute.path`, `MicroAppRoute.builder`, `MicroAppRoute.requiresAuthentication`, `MicroAppRoute.presentation`, `MicroAppRoute.routes`, `MicroAppRouteGroup.builder`, and `MicroAppRouteGroup.routes`.
- [ ] T058 [P] [US3] Implement `MicroAppContext` in `foundations/lib/src/micro_app/micro_app_context.dart`, with all twelve required fields from the data-model.md "MicroAppContext" table: `MicroAppContext.communication`, `MicroAppContext.apiClient`, `MicroAppContext.analytics`, `MicroAppContext.crashReporter`, `MicroAppContext.logger`, `MicroAppContext.featureFlags`, `MicroAppContext.keyValueStore`, `MicroAppContext.secureKeyValueStore`, `MicroAppContext.navigator`, `MicroAppContext.session`, `MicroAppContext.devicePermissions`, and `MicroAppContext.appInfo`. Document that Core builds one per micro app, with storage limited to that micro app's keys.
- [ ] T059 [US3] Implement the `MicroApp` interface in `foundations/lib/src/micro_app/micro_app.dart`, documenting the host sequence from contracts/micro-app.md.
  - Cite data-model.md `MicroApp.id`, `MicroApp.storedKeys`, `MicroApp.initialize`, `MicroApp.routes`, `MicroApp.localizationsDelegates`, `MicroApp.dataEndpoints`, `MicroApp.widgetEndpoints`, and `MicroApp.dispose`.
  - Depends on T057 and T058.
- [ ] T060 [US3] Export the routing and micro app files from `foundations/lib/foundations.dart`, per contracts/public-surface.md.
- [ ] T061 [US3] Create the reference routing support in `foundations/test/support/`:
  - `foundations/test/support/reference_route_validation.dart`: the validation rules in contracts/routing.md, under "Startup checks".
  - `foundations/test/support/reference_router.dart`: builds a go_router from route trees per the mapping table in contracts/routing.md, and implements `AppNavigator` on top of it.
  - Both are dev-only, and both serve as the reference for Core.
- [ ] T062 [US3] Create the reference host and the sample micro apps in `foundations/test/support/`:
  - `foundations/test/support/reference_host.dart`: host sequence steps 1–7 from contracts/micro-app.md, with fake stores limited to each micro app's declared keys (data-model.md "Stored key", `StorageException.type`).
  - `foundations/test/support/sample_micro_apps.dart`: sample Accounts and Transfers micro apps with `storedKeys`, an `accounts.details` route at `/accounts/details/:accountId`, a tab `MicroAppRouteGroup`, and providers.
  - T052–T056 must pass.

**Checkpoint**: Any host can integrate any micro app with no code specific to that micro app. Storage keys can't collide, and routes are router-neutral.

---

## Phase 6: User Story 4 - Use the unified Design System and state-management contract through Foundations (Priority: P2)

**Goal**: The interim `design_system` package (tokens, a brand configuration, light and dark themes, and eight accessible components), re-exported unchanged together with `switchable_state`. The sample state is proven under two adapters.

**Independent Test**: `flutter test` in `foundations/packages/design_system/`, `flutter test test/state_management test/architecture/ambiguous_exports_test.dart` in `foundations/`, and `flutter test test/design_system_reexport_test.dart` in `foundations/example/`. These cover quickstart.md V6 and V9.

### Tests for User Story 4 ⚠️

- [ ] T063 [P] [US4] Write the token tests in `foundations/packages/design_system/test/tokens/ds_tokens_test.dart`.
  - The `DsSpacing` and `DsRadius` values.
  - Contrast ratios in both themes: every `DsColors` text role on its background reaches at least 4.5:1, `textOnAction` on `actionPrimary` reaches at least 4.5:1, and `actionPrimary` against `background` reaches at least 3:1.
  - Cite the data-model.md Design System rows "Primitive tokens" and "Semantic tokens".
- [ ] T064 [P] [US4] Write the brand tests in `foundations/packages/design_system/test/theme/ds_brand_test.dart`.
  - Changing `DsBrand.primary` changes the primary `DsButton` fill and the `focus` role, with no component change (D8).
  - A missing `fontFamily` falls back to the platform font.
  - Cite the data-model.md Design System row "Brand".
- [ ] T065 [US4] Create the accessibility test harness in `foundations/packages/design_system/test/helpers/a11y_harness.dart`.
  - It pumps a widget under both `DsTheme.light()` and `DsTheme.dark()`.
  - It runs `androidTapTargetGuideline`, `iOSTapTargetGuideline`, `labeledTapTargetGuideline`, and `textContrastGuideline` (D1–D3).
  - It also pumps at `TextScaler.linear(2.0)` on a 320-wide surface and asserts there is no overflow (D4).
- [ ] T066 [P] [US4] Write the atom tests in `foundations/packages/design_system/test/components/`, using T065:
  - `foundations/packages/design_system/test/components/ds_button_test.dart`: variants, disabled, and loading (D1–D6, D8).
  - `foundations/packages/design_system/test/components/ds_text_field_test.dart`: label, helper, error, and disabled (D1–D5, D7).
  - `foundations/packages/design_system/test/components/ds_loading_indicator_test.dart`: D3, D4, and D6.
- [ ] T067 [P] [US4] Write the molecule tests in `foundations/packages/design_system/test/components/`, using T065:
  - `foundations/packages/design_system/test/components/ds_card_test.dart` and `ds_list_tile_test.dart`: D1–D4.
  - `ds_inline_message_test.dart`: D1–D4, and every tone mapped to its role pair (D9).
  - `ds_empty_state_test.dart`: D1–D4, with and without an action.
- [ ] T068 [P] [US4] Write the template test in `foundations/packages/design_system/test/components/ds_page_scaffold_test.dart`: the title, the body padding of `DsSpacing.md`, a pinned full-width primary action, the back behavior, and D1–D4.
- [ ] T069 [P] [US4] Write the state-management tests in `foundations/test/state_management/sample_state_test.dart`, following quickstart.md V9 and covering SC-013.
  - The sample widgets import only `package:foundations/foundations.dart`.
  - They declare an `AppState`, an `AppState.grouped` owned by a `StateScope.group`, and a `StateStoreOwner` store.
  - The same tests run under `RiverpodStateAdapter` and `CubitStateAdapter`, calling `pumpAndSettle` after writes, and `overrides` work under both.
- [ ] T070 [P] [US4] Write the ambiguous-exports test in `foundations/test/architecture/ambiguous_exports_test.dart`. It imports `package:foundations/foundations.dart` and references one symbol each from Foundations, `design_system`, and `switchable_state`, so any name clash fails compilation (contracts/public-surface.md).
- [ ] T071 [P] [US4] Write the re-export test in `foundations/example/test/design_system_reexport_test.dart`, following quickstart.md V6b. It imports only `package:foundations/foundations.dart` and builds a `DsPageScaffold` with a `DsTextField`, `DsButton`, and `DsListTile` under `DsTheme.light()`, reading `context.dsColors`.

### Implementation for User Story 4

- [ ] T072 [P] [US4] Implement the primitive tokens in `foundations/packages/design_system/lib/src/tokens/ds_palette.dart`, `ds_spacing.dart`, and `ds_radius.dart`, per contracts/design-system.md. Cite the data-model.md Design System row "Primitive tokens".
- [ ] T073 [US4] Implement the semantic tokens as `ThemeExtension`s with `copyWith` and `lerp`, in `foundations/packages/design_system/lib/src/tokens/ds_colors.dart` and `ds_typography.dart`. Cite the data-model.md Design System row "Semantic tokens". Depends on T072.
- [ ] T074 [US4] Implement `DsBrand`, `DsTheme.light`, `DsTheme.dark`, and the `DsThemeContext` extension in `foundations/packages/design_system/lib/src/theme/ds_brand.dart`, `ds_theme.dart`, and `ds_theme_context.dart`.
  - `textOnAction` is chosen so it meets 4.5:1 contrast on the brand color.
  - The Material component themes are aligned with the same roles.
  - Cite the data-model.md Design System rows "Brand" and "Themes".
  - Depends on T073.
- [ ] T075 [P] [US4] Implement the atoms `DsButton`, `DsTextField`, and `DsLoadingIndicator` in `foundations/packages/design_system/lib/src/atoms/ds_button.dart`, `ds_text_field.dart`, and `ds_loading_indicator.dart`, per contracts/design-system.md. Cite the data-model.md Design System row "Atoms". Depends on T074.
- [ ] T076 [P] [US4] Implement the molecules `DsCard`, `DsListTile`, `DsInlineMessage` (with `DsMessageTone`), and `DsEmptyState` in `foundations/packages/design_system/lib/src/molecules/ds_card.dart`, `ds_list_tile.dart`, `ds_inline_message.dart`, and `ds_empty_state.dart`. Cite the data-model.md Design System row "Molecules". Depends on T074.
- [ ] T077 [P] [US4] Implement the template `DsPageScaffold` in `foundations/packages/design_system/lib/src/templates/ds_page_scaffold.dart`. Cite the data-model.md Design System row "Templates". Depends on T074 and T075.
- [ ] T078 [US4] Export every token, theme, and component from `foundations/packages/design_system/lib/design_system.dart`, and add the 0.1.0 entry to `foundations/packages/design_system/CHANGELOG.md`. T063–T068 must pass.
- [ ] T079 [US4] Build the Design System gallery in `foundations/packages/design_system/example/lib/main.dart`: every component in every state, with light/dark and brand toggles (quickstart.md V6d).
- [ ] T080 [US4] Add `export 'package:design_system/design_system.dart';` and `export 'package:switchable_state/switchable_state.dart';` to `foundations/lib/foundations.dart`, with no `show`, `hide`, or renames (FR-002, FR-042). T069 and T070 must pass.
- [ ] T081 [US4] Restyle the example screens in `foundations/example/lib/main.dart` and `foundations/example/lib/module_b.dart` with `Ds` components under `DsTheme.light()` and `DsTheme.dark()`, wrapped in a root `StateScope` whose adapter is chosen in `main.dart`. T071 must pass.
- [ ] T082 [US4] Write the git-consumer smoke script `foundations/tool/git_consumer_smoke.sh`, following quickstart.md V6c and research R3.
  - It creates a temporary consumer that depends on Foundations through `git: {url: <local repo>, ref: HEAD}`, then runs `flutter pub get`.
  - It asserts that `design_system` resolves "from git … in packages/design_system" at the same commit as Foundations.

**Checkpoint**: Micro apps build real screens with the interim Design System and manage state through one contract, all through their single Foundations dependency.

---

## Phase 7: User Story 5 - Embed UI that another micro app owns (Priority: P3)

**Goal**: The UI-endpoint part of inter-module communication, proven by conformance cases C12–C16 and the example's Home and Cards demo.

**Independent Test**: `flutter test test/testing` in `foundations/` (C12–C16), and `flutter test test/inter_module_widget_test.dart` in `foundations/example/`. These cover the UI rows of quickstart.md V4.

**Depends on**: US1 (the communication interface, providers, the reference implementation, and the conformance check).

### Tests for User Story 5 ⚠️

- [ ] T083 [US5] Extend `foundations/test/testing/inter_module_conformance_test.dart` to run C12–C16 against the reference implementation, where every case passes, and against the broken variants. The "shared registry" variant must fail C16.
- [ ] T084 [US5] Extend `foundations/test/inter_module/endpoint_provider_test.dart`:
  - A sample `WidgetEndpointProvider` for `/cards/v1/home-banner` is exercised by calling `build` directly. It returns 200 with a widget for `cardId = "card-123"`, and 400 `invalid_card_id` for an empty `cardId`.
  - `registerProviders(widgets: …)` calls `registerWidget`.

### Implementation for User Story 5

- [ ] T085 [US5] Add cases C12–C16 to `InterModuleConformance.cases` in `foundations/lib/src/testing/inter_module_conformance.dart`, per contracts/conformance.md and behavior rows B9–B12. Cite data-model.md "Registration": `InterModuleWidgetFactory`. T083 must pass.
- [ ] T086 [US5] Build the example's Home and Cards demo in `foundations/example/lib/`:
  - `foundations/example/lib/cards.dart`: the `/cards/v1/home-banner` `WidgetEndpointProvider`, which returns the element synchronously while the banner loads its own data.
  - In `foundations/example/lib/main.dart`: a Home section that uses `getWidget`, shows a `DsEmptyState` or `DsInlineMessage` fallback on 400 or 404, and has a "Cards installed" toggle.
- [ ] T087 [US5] Write the example UI tests in `foundations/example/test/inter_module_widget_test.dart`. Run C12–C16 against the example implementation, plus every home-banner row of the quickstart.md V4 table, including the fallback when Cards is absent.

**Checkpoint**: Teams can compose UI that another team owns, with predictable fallbacks.

---

## Phase 8: User Story 6 - Evolve the shared contracts safely (Priority: P3)

**Goal**: Versioning, changelog, inclusion rules, and documentation checks that protect every consumer.

**Independent Test**: Review the 0.1.0 release against `foundations/CHANGELOG.md`, `foundations/README.md`, and the pull request template. CI fails on missing documentation or analyzer issues.

- [ ] T088 [P] [US6] Write `foundations/CHANGELOG.md` with the 0.1.0 entry. It has Added, Changed, Deprecated, Removed, and Breaking sections, and states which `design_system` 0.1.0 (from `packages/design_system`) and which `switchable_state` 0.1.0 it delivers (FR-033, FR-038, FR-045, research R17).
- [ ] T089 [P] [US6] Write `foundations/README.md`. It covers:
  - the package's purpose and its two entry points;
  - a step-by-step guide to publishing a data endpoint in one micro app and consuming it from another (SC-004);
  - the inclusion rules: no feature models, no business entitlements, no wrapping of `design_system` or `switchable_state`, and a fixed capability set;
  - the versioning and deprecation policy;
  - the Design System relocation steps from research R3.
- [ ] T090 [P] [US6] Write the pull request template `foundations/.github/pull_request_template.md`. Its checklist covers the semver classification (compatible or breaking), migration notes, documentation for new public elements, the inclusion rules, and the changelog entry, which also states the delivered `design_system` and `switchable_state` versions. This covers US6 scenarios 1–4.
- [ ] T091 [US6] Extend `foundations/.github/workflows/ci.yaml` with `dart doc --dry-run`, which must report 0 warnings, and with the coverage gate from T092.
- [ ] T092 [P] [US6] Write the coverage gate `foundations/tool/check_coverage.dart`. It parses `coverage/lcov.info` and fails below 90% line coverage of the concrete code under `lib/` (FR-034, quickstart.md V7).

**Checkpoint**: Every release is classified, documented, and gated.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Final quality pass and handoff to Core.

- [ ] T093 Run `flutter analyze` in `foundations/`, `foundations/packages/design_system/`, and `foundations/example/`, and `dart doc --dry-run` in `foundations/`. Fix every issue until there are 0 issues and 0 warnings (SC-006).
- [ ] T094 [P] Write the Core handoff notes in `specs/001-foundations-package/core-handoff.md`. They list what Core's spec must cover:
  - the reusable host sequence, which replaces `AppShellContainer`;
  - one context per micro app, with stores limited to its keys;
  - a router built from contracts/routing.md;
  - `traceName` diagnostics;
  - rejecting `register` calls after startup;
  - passing `package:foundations/testing.dart`;
  - running `switchable_state_test` on the chosen adapter.
- [ ] T095 Run every check in quickstart.md, V1–V10, including the V8 manual onboarding walkthrough and the T082 smoke script. Record each outcome in `specs/001-foundations-package/validation-report.md`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies. Within it, T011 waits for T002, T003, T008, and T009.
- **Foundational (Phase 2)**: Depends on Setup, and blocks every story.
- **US1 (Phase 3)**: Depends on Foundational only. This is the MVP.
- **US2 (Phase 4)**: Depends on Foundational only. It can run in parallel with US1.
- **US3 (Phase 5)**: Depends on US1 (T025–T027) and US2 (T040–T050), because `MicroAppContext` and `MicroApp` reference their types.
- **US4 (Phase 6)**: Depends on Foundational only. It can run in parallel with US1 and US2. T081 also touches the example from US1 (T031), so do T081 after T031.
- **US5 (Phase 7)**: Depends on US1. T086 uses `Ds` components, so it also depends on US4 (T078 and T080).
- **US6 (Phase 8)**: T088–T090 and T092 can start at any time. T091 needs T092.
- **Polish (Phase 9)**: After all the stories you want to ship.

### User Story Dependency Graph

```text
Setup ─► Foundational ─┬─► US1 (P1, MVP) ─┬─► US3 (P2) ◄─┐
                       │                  └─► US5 (P3) ◄─┼── US4 (for Ds fallbacks)
                       ├─► US2 (P2) ──────────────────────┘ (US2 also feeds US3)
                       ├─► US4 (P2)
                       └─► US6 (P3, docs/gates; any time)
```

### Within Each Story

- Write the tests first, and confirm they fail.
- Build value types before interfaces, interfaces before support code and reference implementations, and the exports last.
- A story is complete when its own tests pass and its checkpoint holds.

---

## Parallel Examples

### User Story 1

```bash
# Tests together:
Task: "T017 endpoint value tests in foundations/test/inter_module/inter_module_endpoint_test.dart"
Task: "T018 response tests in foundations/test/inter_module/inter_module_response_test.dart"
Task: "T019 status/error code tests in foundations/test/inter_module/inter_module_status_test.dart"
Task: "T020 data provider tests in foundations/test/inter_module/endpoint_provider_test.dart"
Task: "T021 conformance self-test in foundations/test/testing/inter_module_conformance_test.dart"
# Then value types together:
Task: "T022 InterModuleEndpoint in foundations/lib/src/inter_module/inter_module_endpoint.dart"
Task: "T023 InterModuleCommunicationResponse in foundations/lib/src/inter_module/inter_module_response.dart"
Task: "T024 status and error codes in foundations/lib/src/inter_module/inter_module_status.dart"
```

### User Story 2

```bash
# All eleven contract files are independent:
Task: "T041 AppInfo in foundations/lib/src/capabilities/app_info.dart"
Task: "T042 ApiClient in foundations/lib/src/capabilities/api_client.dart"
Task: "T046 FeatureFlags in foundations/lib/src/capabilities/feature_flags.dart"
Task: "T050 AppNavigator in foundations/lib/src/routing/app_navigator.dart"
# … T043–T045, T047–T049 likewise
```

### User Story 3

```bash
Task: "T052 route-tree validation tests in foundations/test/routing/route_tree_validation_test.dart"
Task: "T053 reference router tests in foundations/test/routing/reference_router_test.dart"
Task: "T055 reference host tests in foundations/test/micro_app/reference_host_test.dart"
Task: "T057 route contract types in foundations/lib/src/routing/micro_app_route.dart"
Task: "T058 MicroAppContext in foundations/lib/src/micro_app/micro_app_context.dart"
```

### User Story 4

```bash
# Design System tests together:
Task: "T063 token contrast tests in foundations/packages/design_system/test/tokens/ds_tokens_test.dart"
Task: "T064 brand tests in foundations/packages/design_system/test/theme/ds_brand_test.dart"
Task: "T069 state-management tests in foundations/test/state_management/sample_state_test.dart"
# After T074, components together:
Task: "T075 atoms in foundations/packages/design_system/lib/src/atoms/"
Task: "T076 molecules in foundations/packages/design_system/lib/src/molecules/"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1, Setup: T001–T011.
2. Phase 2, Foundational: T012–T016.
3. Phase 3, US1: T017–T032.
4. **Stop and validate**: quickstart.md V1, V3, and the data rows of V4. Teams can already collaborate through data endpoints.

### Incremental Delivery

1. Setup and Foundational give you the guardrails.
2. US1: inter-module data (the MVP).
3. US2 and US4, in parallel: capabilities, and the Design System with the state contract.
4. US3: micro app integration, storage keys, and routing. This needs US1 and US2.
5. US5: shared UI. This needs US1, plus US4 for the fallbacks.
6. US6, then Polish, ending with the full V1–V10 run (T095).

### Parallel Team Strategy

- Developer A: US1, then US5.
- Developer B: US2, then US3, once US1's interfaces exist.
- Developer C: US4 (the interim Design System and the state contract).
- Anyone: US6 documents and gates.

---

## Notes

- [P] means a different file with no dependency on an unfinished task. Tasks that extend an existing file (T083, T084) aren't marked [P].
- The interim Design System must stay movable: never import `foundations` from it, and keep its name and public library unchanged (FR-056).
- Commit after each task or logical group, on the `001-foundations-package` branch of the `foundations` submodule.
