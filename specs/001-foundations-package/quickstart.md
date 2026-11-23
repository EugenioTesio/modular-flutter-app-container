# Quickstart: Validating the Foundations Package

These are runnable checks that prove the feature works. Each one names the user stories (US) and success criteria (SC) it covers. Exact signatures are in [contracts/](contracts/), and the entities are in [data-model.md](data-model.md).

## Prerequisites

- Flutter 3.47 or later on the stable channel (`flutter --version`). This brings Dart 3.13.
- The Foundations submodule checked out. The interim Design System lives inside it at `packages/design_system/`, so the `design-system` submodule isn't needed yet.

  ```bash
  git submodule update --init foundations
  ```

Setup:

```bash
cd foundations && flutter pub get
```

```bash
cd foundations/packages/design_system && flutter pub get
```

```bash
cd foundations/example && flutter pub get
```

## V1 — Package boundary (SC-001, SC-012)

```bash
cd foundations && flutter pub deps --no-dev --style=compact
```

**Expected**: The only direct runtime dependencies are `flutter`, `design_system` (for now from `packages/design_system`), and `switchable_state` 0.1.0. There are no vendor SDKs, no `switchable_state_*` adapters, no state libraries (`flutter_riverpod`, `flutter_bloc`, and so on), no router package (`go_router` is dev-only), no Core, and no micro apps.

```bash
cd foundations && flutter test test/architecture
```

**Expected**: All tests pass:
- The dependency allow-list holds.
- `lib/` contains no `static` field that is not `const` and no top-level `var` or `late` declaration.
- Importing `package:foundations/foundations.dart` compiles with no `ambiguous_export`.

## V2 — Static quality and documentation (SC-006)

```bash
cd foundations && flutter analyze
```

**Expected**: `No issues found!`. The `public_member_api_docs` lint is on, so this also confirms every public member is documented.

```bash
cd foundations && dart doc --dry-run
```

**Expected**: The command finishes with 0 warnings.

## V3 — Conformance check proves itself (US1, US5, SC-003)

```bash
cd foundations && flutter test test/testing
```

**Expected**:
- Cases C01–C17 all pass against the reference implementation.
- Each deliberately broken variant fails the case that targets its defect, as listed in [contracts/conformance.md](contracts/conformance.md#self-verification-inside-foundations).

## V4 — Every inter-module outcome in a runnable example (US1, US5, FR-035, SC-008)

```bash
cd foundations/example && flutter test
```

**Expected**:
- The example's in-memory implementation passes C01–C17.
- The scenario tests cover the outcomes below. In each one, the consumer tells the outcome apart using only the status and the error code (SC-008).

| Call | Outcome |
|------|---------|
| `/module-b/v1/details` with `{id: '123'}` | 200 and `{id, name}` |
| `/module-b/v1/details` with `{}` | 400, `missing_required_parameters` |
| `/module-b/v1/details` with `{id: ''}` | 400, `invalid_id` |
| `/module-b/v1/details` with `{id: '999'}` | 404, `details_not_found` |
| An unregistered URL | 404, `endpoint_not_registered` |
| A provider that refuses access | 400, `unauthenticated` or `forbidden` |
| A handler that throws | 500 |
| `/cards/v1/home-banner` with `{cardId: 'card-123'}` | 200 and a widget, returned synchronously |
| `/cards/v1/home-banner` with `{}` | 400, `missing_required_parameters` |
| `/cards/v1/home-banner` with `{cardId: ''}` | 400, `invalid_card_id` |
| Cards not registered | 404, and Home renders its fallback |

Manual check:

```bash
cd foundations/example && flutter run -d macos
```

**Expected**: The demo screen shows one result row for each scenario above. The Home section shows the Cards banner, and after you toggle "Cards installed" off, it shows the fallback instead.

## V5 — Capabilities and micro app integration with fakes only (US2, US3, SC-002, SC-005, SC-007, SC-014, SC-016)

```bash
cd foundations && flutter test test/capabilities test/micro_app
```

**Expected**:
- A sample micro app built only on Foundations (in `test/support/`) runs with in-memory fakes for all 12 contracts in `MicroAppContext`. There is no device, network, platform channel, or Core.
- A minimal reference host that knows only `MicroApp` runs the full host sequence on two sample micro apps:
  - It reads ids and `storedKeys`. Two micro apps declaring `session_cache` stop startup with an error naming the key and both micro apps.
  - It builds one context per micro app, whose fake stores are limited to that micro app's keys.
  - It initializes both micro apps.
  - It collects their route trees, localization delegates, and endpoint providers. It detects a duplicate full path.
  - It registers every provider with one `registerProviders` call. A duplicate provider URL stops startup with `DuplicateEndpointException`.
  - It disposes both micro apps in reverse order, and keeps disposing after one teardown throws.
- Accessing an undeclared key (for example Accounts writing `transfers.draft`) returns `Err(StorageException(type: undeclaredKey))`, and Transfers' value is unchanged (SC-016).
- After startup, `isRegistered` is `true` for each declared provider's URL and type.
- Each sample `DataEndpointProvider` and `WidgetEndpointProvider` is unit-tested by calling `handle` or `build` directly, with no communication implementation (SC-014).
- `FeatureFlags.changes` emits on refresh, and a listener re-reads the new value.
- `SessionContext` emits on sign-in, sign-out, and user switch. Reading the context while signed out yields `Unauthenticated`.
- `FeatureFlags` returns the flag's default when the flag is unknown or has the wrong kind.
- Every `ApiExceptionType` can be produced and matched exhaustively.
- Leaving a contract out of `MicroAppContext` does not compile. This is checked by an analyzer-expectation test.

```bash
cd foundations && flutter test --concurrency=8
```

**Expected**: The tests pass when run in parallel, which shows there is no shared global state (SC-012).

## V6 — Interim Design System, delivered through Foundations (US4, SC-009, SC-010, SC-017, SC-018)

The interim package unblocks this check.

**a. The package itself** (it moves with the package later):

```bash
cd foundations/packages/design_system && flutter analyze && flutter test
```

**Expected**:
- 0 analyzer issues.
- For every component in both `DsTheme.light()` and `DsTheme.dark()`:
  - The tap-target, labeled-tap-target, and text-contrast guidelines pass (D1–D3).
  - There is no overflow at 200% text scale on a 320-wide surface (D4).
- The disabled, loading, error, and tone states behave as specified (D5–D7, D9).
- Token tests: every text role reaches at least 4.5:1 on its background in both themes, and `actionPrimary` reaches at least 3:1 against `background`.
- Brand test: `DsTheme.light(brand: DsBrand(primary: X))` changes the primary `DsButton`'s fill and the focus color to colors derived from X, with no component change (D8).
- Architecture test: the runtime dependencies are `{flutter}` only, and no file imports `package:foundations` (FR-056).

**b. Delivery through Foundations:**

```bash
cd foundations/example && flutter pub deps --style=compact | grep design_system
```

**Expected**: `design_system` appears **only** as a transitive dependency through `foundations`, from `packages/design_system`.

```bash
cd foundations/example && flutter test test/design_system_reexport_test.dart
```

**Expected**:
- The test imports only `package:foundations/foundations.dart`.
- It builds a `DsPageScaffold` with a `DsTextField`, a `DsButton`, and a `DsListTile`, using `DsTheme.light()` and `context.dsColors`, all by their original names. The test compiles and renders.

**c. Consumers that use Foundations through git** (an offline smoke check):
- Create a scratch consumer whose `pubspec.yaml` has `foundations: {git: {url: <local path to foundations>, ref: <tag or commit>}}`, then run `flutter pub get`.

**Expected**: `pub get` reports `design_system … from git … in packages/design_system`, at the same commit as `foundations`. `.dart_tool/package_config.json` points both packages into the same pub-cache checkout ([research R3](research.md#r3-design-system-dependency-and-re-export)).

**d. Gallery** (manual):

```bash
cd foundations/packages/design_system/example && flutter run -d macos
```

**Expected**: Every component appears in every state. The light/dark and brand toggles restyle everything.

## V9 — State-management contract through Foundations (US4, SC-002, SC-013)

`switchable_state` 0.1.0 is already on pub.dev, so this check can run now.

```bash
cd foundations && flutter test test/state_management
```

**Expected**:
- A sample micro app in `test/support/` imports only `package:foundations/foundations.dart`. It declares one app-wide `AppState`, one `AppState.grouped` owned by a `StateScope.group`, and one screen-owned store through `StateStoreOwner`.
- The same widget tests run twice, wrapped in `StateScope(adapter: RiverpodStateAdapter())` and in `StateScope(adapter: CubitStateAdapter())`. They pass under both with no code changes. The tests call `pumpAndSettle` after writes, which allows for Cubit's one-frame delay.
- Overriding initial state through `overrides: [state.overrideWith(...)]` works under both adapters.

```bash
cd foundations && grep -nE 'switchable_state_|flutter_riverpod|flutter_bloc' pubspec.yaml
```

**Expected**: Any matches are under `dev_dependencies:` only (FR-043).

## V10 — The route contract builds a real router (US2, US3, SC-015)

```bash
cd foundations && flutter test test/routing
```

**Expected**:
- Route-tree validation rejects a duplicate full path, a top-level path without the `/<microAppId>` prefix, and a repeated path parameter name.
- A reference adapter in `test/support/` builds a go_router from the sample route trees. go_router is a dev dependency only.
  - **Deep link**: the link `/accounts/details/42?tab=movements` shows the details screen with `{accountId: '42', tab: 'movements'}` and an empty body.
  - **Push**: `push('/accounts/details/42?tab=movements', body: {'source': 'transfers'})` shows the same screen, which also receives the body.
  - **Failures**: `push('/nope/screen')` returns `Err(RouteNotFoundException('/nope/screen'))`. `popUntil('/not/in/stack')` returns the same exception and closes nothing.
  - **Results**: a picker opened with `push` (or `replace`) and closed with `pop(result: {...})` completes with `Ok(RouteCompleted(result))`. Closing without a result gives `Ok(RouteCancelled())`.
  - **Go and popUntil**: `go('/home')` replaces the stack and returns `Ok`. After pushing two screens, `popUntil('/home')` closes both, and each opener completes with `RouteCancelled`.
  - **Sign-in**: a route that needs sign-in (directly or through an ancestor) redirects to sign-in while the fake session is `Unauthenticated`. A route with `requiresAuthentication: false` opens.
  - **Presentation**: a `fullscreenDialog` route is presented as a full-screen dialog.
  - **Groups**: moving between routes inside a `MicroAppRouteGroup` keeps the group's layout, and a `StateScope.group` placed in it keeps its state until the user leaves the group.

## V7 — Coverage (FR-034)

```bash
cd foundations && flutter test --coverage
```

**Expected**: `coverage/lcov.info` reports at least 90% line coverage of the concrete code in `lib/` (value types, extensions, and the conformance check). Abstract interfaces have no executable lines.

## V8 — Developer onboarding (SC-004)

This is a timed manual walkthrough. A developer who has not used Foundations before follows only `foundations/README.md` and the example. They register `/module-b/v1/details` in one sample micro app and request it from another.

**Expected**: The developer finishes in under 30 minutes and handles all four statuses.

## Traceability

| Check | User stories | Success criteria |
|-------|--------------|------------------|
| V1 | US6 | SC-001, SC-012 |
| V2 | US6 | SC-006 |
| V3 | US1, US5 | SC-003 |
| V4 | US1, US5 | SC-008 |
| V5 | US2, US3 | SC-002, SC-005, SC-007, SC-011 (the API client is the only backend path), SC-014, SC-016 |
| V10 | US2, US3 | SC-015 |
| V6 | US4 | SC-009, SC-010, SC-017, SC-018 (relocation readiness) |
| V7 | US6 | (FR-034) |
| V8 | US1 | SC-004 |
| V9 | US4 | SC-002, SC-013 |
