# Phase 0 Research: Foundations Package

Every unknown in the plan's Technical Context is resolved here. Each entry gives the decision, the reasoning, and the alternatives that were considered.

## R1. Language, SDK, and package kind

- **Decision**: A plain Flutter library package (not a plugin). It requires `sdk: ^3.13.0` and `flutter: ">=3.47.0"`, the stable toolchain installed on the development machine (Dart 3.13.7, Flutter 3.47.9). The pubspec sets `publish_to: none`.
- **Rationale**:
  - UI endpoints return `Widget` and micro apps declare `LocalizationsDelegate`s, so the package needs the Flutter SDK.
  - There is no native code, so a plugin is not needed.
  - Dart 3 class modifiers and sealed types (R4, R6) need SDK 3.0 or later. Pinning to the current stable avoids old-SDK workarounds.
- **Alternatives considered**:
  - A pure Dart package: rejected because widgets are part of the contract.
  - Splitting into `foundations_core` (pure Dart) and `foundations_flutter`: rejected because the article explicitly keeps one Foundations package, and the split adds a version to coordinate.

## R2. Repository location and branching

- **Decision**: Write the code in the `foundations/` git submodule (`modular-flutter-app-foundations`) on a feature branch named `001-foundations-package`. The spec, plan, and tasks stay in this container repo under `specs/001-foundations-package/`.
- **Rationale**: Each package has its own repository (from `.gitmodules`). The container repo coordinates the specs and pins the submodule commits.
- **Alternatives considered**: Putting the package code in the container repo: rejected because it would bypass the per-package repositories that already exist.

## R3. Design System dependency and re-export

This decision was revised on 2026-11-21. The `design-system` repository is still empty, so the user asked for an interim internal package.

- **Decision (interim, now)**:
  1. The Design System is built as its **own Flutter package** at `foundations/packages/design_system/`, inside the Foundations repository. It has its own `pubspec.yaml` (name `design_system`, `version: 0.1.0`, `publish_to: none`, runtime dependency `flutter` only), tests, `CHANGELOG.md`, and gallery `example/`.
  2. Foundations' `pubspec.yaml` depends on it with `design_system: { path: packages/design_system }`. The relative path stays inside the repository.
  3. `lib/foundations.dart` re-exports the Design System's public library: `export 'package:design_system/design_system.dart';`. There are no wrappers, no `show`/`hide` lists, and no renames (FR-002).
  4. Each Foundations `CHANGELOG.md` entry states the delivered Design System version. Adopting a breaking Design System change bumps Foundations' breaking-change version (FR-038; see R17).
- **Decision (later, when the Design System moves to its own repository; FR-056)**:
  1. Run `git subtree split --prefix=packages/design_system` in the Foundations repository, and push the result to `modular-flutter-app-design-system` so the history is kept. Tag it `v<version>`.
  2. Replace the dependency in Foundations with `design_system: { git: { url: https://github.com/EugenioTesio/modular-flutter-app-design-system.git, ref: v<version> } }`, and delete `packages/design_system/`.
  3. Nothing else changes. The package name, the public library, and the re-export line stay the same, so micro apps change 0 lines (SC-018).
  4. From then on, local development across repositories can use a `pubspec_overrides.yaml` that is git-ignored and never committed, pointing to `../design-system`.
- **Verified behavior** (tested on 2026-11-21 with Dart 3.13.7, in a scratch repo that reproduced this layout): when a consumer depends on Foundations through `git: {url, ref}`, pub resolves the nested `path: packages/design_system` as a git dependency on **the same commit** (`+ ds 0.1.0 from git … at 0c6370 in packages/ds`). Both packages come from one checkout in the pub cache, so the single-version rule (FR-036) holds with no extra tag. The dart.dev page on dependencies does not document this case, so this test is the evidence.
- **Rationale**:
  - A separate package from day one keeps the boundary real: own manifest, Flutter-only dependencies, and no reference to Foundations. Moving it later is then a relocation, not a refactor.
  - A path dependency inside the same repository pins the Design System to the Foundations commit, which is the strongest form of "one version".
  - Consumers resolve `design_system` only through Foundations (FR-037).
  - If a Design System name collides with a Foundations name, Dart reports an ambiguous export at compile time, so collisions can't slip through. The architecture test in [quickstart V1](quickstart.md#v1--package-boundary-sc-001-sc-012) imports the main library to trigger this.
- **Alternatives considered**:
  - Putting the Design System code in Foundations' own `lib/src/`: rejected because it blurs the boundary FR-002 protects and would turn the later move into a refactor.
  - Waiting for the `design-system` repository: rejected by the user, because it blocks User Story 4.
  - Making the `design-system` repository the interim home with git tags right away: rejected because it adds a release step to every change while the component set is still settling.
  - A Dart Pub workspace (`resolution: workspace`): rejected for this package, because workspace members cannot resolve outside the workspace root, which would break consumers that use Foundations through git. The article's workspace recommendation applies to the hosts (App Shell plus examples) and belongs to the App Shell feature.
  - A private hosted pub server: rejected because no such infrastructure exists yet. Revisit if the number of packages grows.

## R4. Contract declaration style

- **Decision**:
  - Every capability and the communication contract are `abstract interface class`es.
  - Value types are `@immutable final class`es with value equality. They use `@immutable`, `mapEquals`, and `setEquals` from `package:flutter/foundation.dart`, so `package:meta` is not needed.
  - Closed alternatives (results, states, flag kinds) are `sealed class`es.
- **Rationale**:
  - `interface` stops consumers from `extends`-coupling to a base implementation. Every implementer, real or fake, implements the full surface.
  - `final` value types keep the public surface deliberate (FR-006).
  - `sealed` gives exhaustive `switch` handling for every outcome, which supports SC-008.
- **Alternatives considered**:
  - Plain `abstract class`: rejected because it allows partial `extends` reuse that couples consumers to defaults.
  - Freezed or code generation: rejected because it adds a runtime and build dependency, which violates FR-003.

## R5. Inter-module communication shape

- **Decision**: Keep the article's published names and members: `InterModuleEndpoint(url, parameters, requiredParameters)`, `InterModuleCommunicationResponse<T>(statusCode, data, errorCode, isSuccess)`, `InterModuleHandler`, `InterModuleWidgetFactory`, and `InterModuleCommunication.register/request/registerWidget/getWidget`. On top of that:
  - Remove `static late InterModuleCommunication instance` (FR-016).
  - Add `InterModuleStatusCode` (`ok`, `badRequest`, `notFound`, `internalError`) and `InterModuleErrorCode` (`missingRequiredParameters`, `endpointNotRegistered`, `unauthenticated`, `forbidden`) as `abstract final class`es with `static const` members (FR-012).
  - Add the named constructors `InterModuleCommunicationResponse.success(data)` and `.failure(statusCode:, errorCode:)` to the response.
  - Add `DuplicateEndpointException` (an `Exception`, kept in `lib/src/exceptions/` like every exception, that carries `url` and `type`) for rejected duplicate registrations (FR-014).
  - Add `isInterModuleValue(Object?)`, which states the payload rule (primitives, lists, and maps with `String` keys) so Core can assert it and tests can check it.
  - Added on 2026-11-21 from `flutter_enterprise_architecture` ([R20](#r20-adapting-flutter_enterprise_architecture)):
    - `isRegistered({url, type})`;
    - an optional `traceName` on `InterModuleEndpoint`;
    - the class-based `DataEndpointProvider` and `WidgetEndpointProvider`;
    - the `registerProviders` bulk extension.
- **Rationale**: Teams that read the article will recognise the API. The additions turn implied behaviour into published values (codes, errors, the payload rule) without changing the article's shape. Duplicate registration throws a `DuplicateEndpointException` (user convention: exceptions are named `…Exception` and live in `lib/src/exceptions/`), and hosts let it stop startup.
- **Alternatives considered**:
  - A sealed success/failure response type: rejected because it diverges from the article and from status-code handling that mirrors HTTP.
  - Returning a failure response from `register` instead of throwing: rejected because a duplicate is a wiring bug that should stop startup, not a runtime outcome.

## R6. Failure modeling for capabilities

- **Decision**:
  - Use one shared `sealed class Result<T, F>` with `Ok<T, F>(value)` and `Err<T, F>(failure)` for capability operations that can fail: the API client and storage.
  - Each capability has its own failure type: `ApiFailure` with `ApiFailureType`, and `StorageFailure`.
  - Navigation uses its own sealed `NavigationResult` (`RouteCompleted`, `RouteCancelled`, `RouteNotFound`), because "cancelled" is a normal ending, not a failure.
- **Rationale**:
  - Explicit, exhaustive outcomes satisfy FR-028 without exceptions crossing the contract.
  - Vendor exceptions stay inside Core's adapters.
  - One generic `Result` avoids per-capability duplicates.
- **Alternatives considered**:
  - Throwing typed exceptions: rejected because callers can forget to catch them and nothing makes them handle every case.
  - A third-party `Result` or `Either` type (fpdart, dartz): rejected because it is a runtime dependency (FR-003).

## R7. Delivering contracts without globals

- **Decision**: `MicroAppContext` is an `@immutable final class` with one **required** field for each contract: `communication`, `apiClient`, `analytics`, `crashReporter`, `logger`, `featureFlags`, `keyValueStore`, `secureKeyValueStore`, `navigator`, `session`, `permissions`, `appInfo`. Core's initialization builds it, and the host passes it to `MicroApp.initialize(context)`. Each micro app decides how to make the context available to its own UI. Foundations declares no `static` mutable fields and no top-level variables.
- **One context per micro app** (2026-11-21): Core builds a separate `MicroAppContext` for each micro app. All services are shared except the two stores, which are limited to that micro app's `storedKeys` ([R12](#r12-storage)). Internal dependencies are built in `initialize`, and overrides go through the micro app's constructor. The user confirmed this instead of the earlier package's `registerDependencies`.
- **State inside micro apps**: Each micro app keeps its own state in stores from the re-exported `switchable_state` contract ([R19](#r19-state-management-contract-switchable_state)). How it passes the `MicroAppContext` to its widgets (through constructors or a scope it owns) is still the micro app's choice.
- **Rationale**:
  - Every field is required, so a host that forgets a contract fails to compile. That is stronger than the runtime error the spec's "Contract not supplied" edge case requires.
  - Tests build a context from fakes, with no shared state, which supports SC-012 and parallel tests.
  - It matches the clarification "No globals. Core provides the initialization."
- **Alternatives considered**:
  - A service locator (such as `get_it`): rejected because it is global state and a third-party dependency.
  - Foundations shipping an `InheritedWidget` scope: rejected because it dictates UI-level architecture, while the article lets each micro app pick its own. A shared scope can still be added later as a compatible change.
  - Separate constructor parameters on each micro app: rejected because hosts would need custom code for each micro app, which violates FR-030.

## R8. Endpoint registration lifecycle

This question was deferred during clarify.

- **Decision** (revised on 2026-11-21, [R20](#r20-adapting-flutter_enterprise_architecture)):
  - Micro apps *declare* their endpoints as `dataEndpoints` and `widgetEndpoints` and never call `register` themselves.
  - The host registers every declared provider in one `registerProviders` call, after all `initialize` steps finish and before the app runs.
  - Core's spec should still require Core to reject `register` calls after startup completes, as a safety net. Foundations' conformance check does not test this, because the contract has no "startup complete" signal.
- **Rationale**:
  - The article says registration happens during bootstrap. A fixed set of endpoints means a 404 reliably means "module absent".
  - Declaring endpoints instead of registering them in code makes late registration structurally impossible for well-behaved micro apps. It also gives the host the full endpoint list before startup completes.
- **Alternatives considered**: Registering at any time, or adding unregistration: rejected because consumers would see endpoints appear and disappear, and nothing in scope needs that.

## R9. Conformance check packaging

- **Decision**:
  - `package:foundations/testing.dart` exports `InterModuleConformance`. It exposes a list of `ConformanceCase`s; each case has a `description` and a `Future<ConformanceResult> run(InterModuleCommunication Function() create)`.
  - It depends on no test framework. A consumer wraps each case in its own `test()` call (a two-line loop, shown in [contracts/conformance.md](contracts/conformance.md)).
  - The cases cover every SC-003 scenario: success, missing, null, invalid value, not registered, item not found, unexpected failure (data and UI), duplicate registration (data and UI), separate registries, synchronous UI return, empty strings passed through, and the `isRegistered` check (C17).
- **Rationale**: Foundations keeps no test framework among its runtime dependencies (FR-003). The same cases run under `flutter_test`, `package:test`, or a debug self-check at app startup.
- **Alternatives considered**: Calling `group`/`test` directly from `lib/`: rejected because `flutter_test` (and `test_api`) would become a runtime dependency of every consumer.

## R10. API client contract

- **Decision**:
  - The contract has one core operation, `Future<Result<ApiResponse, ApiFailure>> send(ApiRequest request)`. `get/post/put/patch/delete` are extension methods on top of it.
  - `ApiRequest` holds `method`, `path` (relative, starting with `/`), `query`, `headers`, and a JSON-encodable `body`.
  - `ApiResponse` holds `statusCode`, the decoded JSON `body`, and `headers`.
  - Implementations attach credentials and resolve the base address for the environment. Micro apps never see either (FR-039).
  - Failures map like this:

| Condition | `ApiFailureType` |
|-----------|------------------|
| No connection or DNS failure | `noConnectivity` |
| The deadline was exceeded | `timeout` |
| HTTP 401 | `unauthorized` |
| Any other HTTP 4xx | `clientError` (the `statusCode` and body are kept) |
| HTTP 5xx, or a response that can't be decoded | `serverError` |

  - The first release is JSON only. Binary and multipart bodies are deferred.
- **Rationale**: One operation keeps fakes trivial to write. The table makes "exactly one category" (FR-039) testable.
- **Alternatives considered**: Exposing a `Dio` or `http.Client`: rejected because it is a vendor type (FR-004). A generic typed decoder in the contract: rejected because decoding belongs to each micro app's data layer.

## R11. Feature flags

- **Decision**:
  - A flag is defined once as a typed object that carries its own default: `sealed class FeatureFlag<T>` with `BoolFlag`, `StringFlag`, and `NumberFlag` (`const` constructors take `key` and `defaultValue`).
  - The contract is `T valueOf<T extends Object>(FeatureFlag<T> flag)`. Implementations return the default when the flag is unknown, has the wrong kind, or the provider is unavailable (FR-040).
  - `Stream<void> get changes` emits after each refresh (FR-051, from `RemoteConfigController` in [R20](#r20-adapting-flutter_enterprise_architecture)). It carries no payload, so consumers re-read typed flags and never handle raw config maps.
  - Keys follow `<microAppId>.<flag_name>`.
- **Rationale**: Carrying the default in the definition keeps call sites from disagreeing about defaults, while still meeting "every read takes a caller-supplied default". The sealed kinds limit values to on/off, text, or number.
- **Alternatives considered**: Separate `getBool/getString/getNumber(key, default)` methods: rejected because it scatters defaults across call sites and invites typos in keys.

## R12. Storage

- **Decision**:
  - `KeyValueStore` exposes `read`, `write`, `delete`, and `containsKey`. Values are `String`, and every operation returns `Future<Result<…, StorageFailure>>`.
  - `SecureKeyValueStore` is a separate interface with the same members (`implements KeyValueStore`), so the type system tells the two apart and secure storage can be passed wherever general storage is expected, but not the other way round.
  - Revised on 2026-11-21, from the user's stored-keys idea:
    - Every micro app declares its keys in `MicroApp.storedKeys`, which the host reads before `initialize`.
    - The host stops startup if two micro apps declare the same key.
    - Core builds each micro app's `KeyValueStore` and `SecureKeyValueStore` limited to its declared keys. Any operation on another key returns `Err(StorageFailure(type: undeclaredKey))`.
    - Keys *should* follow `<microAppId>.<key>`, but it is the declaration that reserves a key. Legacy keys such as `auth_token` can be declared as they are.
- **Rationale**:
  - String values match what secure-storage vendors support and keep the contract vendor-neutral. Each micro app serializes its own models (FR-005).
  - A naming convention alone can't stop one micro app from overwriting another's value. Declaring keys and limiting each store to them makes overwrites impossible and gives an inventory of every stored key.
- **Alternatives considered**:
  - Storing typed values or JSON inside the contract: rejected because it pushes serialization concerns into Foundations.
  - Having the store add the micro app id to every key automatically: rejected because it breaks legacy keys and hides the inventory.
  - Declaring keys without enforcing them (only the startup duplicate check, as in the earlier package): rejected because an undeclared write could still overwrite another micro app's value.

## R13. Session context and identity

The identity fields question was deferred during clarify.

- **Decision**:
  - `SessionContext` exposes `SessionState get current` and `Stream<SessionState> get changes`.
  - `sealed class SessionState` has `Authenticated(UserIdentity identity)` and `Unauthenticated()`.
  - `UserIdentity` holds only `userId` (an opaque `String`) and an optional `displayName`.
  - Fields can be added later, as optional fields, in compatible releases.
- **Rationale**: This is the smallest set the current micro apps need. It never exposes tokens, which matches FR-039: credentials stay inside the API client implementation.
- **Alternatives considered**: Exposing a token or claims map: rejected for security reasons and because it is free-form.

## R14. Permissions

- **Decision**:
  - `@immutable final class Permission` with a `const Permission(this.name)` constructor and value equality. Names follow `<microAppId>.<action>`.
  - The contract is `bool has(Permission permission)` plus `Stream<void> get changes`. Permissions follow the session, so the stream emits whenever entitlements reload.
  - `abstract final class SharedPermissions` holds the `static const` permissions that more than one module checks. It starts empty, and a permission moves in when a second module needs it (FR-025).
  - Permissions used by only one module are declared as `const` values inside that micro app.
- **Rationale**: Callers use constants, not text typed at each call site (FR-025). Answering synchronously keeps UI checks simple. Session changes trigger re-reads through `changes`.
- **Alternatives considered**:
  - A closed enum in Foundations: rejected because every module-only permission would need a Foundations release, which the clarify session ruled out.
  - Dart extension types: rejected because an `as` cast can bypass them.

## R15. Navigation

- **Decision** (revised on 2026-11-21: "Foundations provides a route contract that Core uses to create the implementation"):
  - **Route contract** ([contracts/routing.md](contracts/routing.md)):
    - The sealed `MicroAppRouteNode` has two node types: `MicroAppRoute` (`name`, `path` pattern with `:param` segments, `builder`, `requiresAuthentication`, `presentation`, and child `routes`) and `MicroAppRouteGroup` (a shared layout `builder` and its `routes`).
    - Core builds the real router from these nodes. For go_router, a route becomes `GoRoute` and a group becomes `ShellRoute`.
  - **Navigation**:
    - `AppNavigator.open(routeName, {Map<String, String> parameters})` returns `Future<NavigationResult>`.
    - `go(routeName, …)` returns `NavigationFailure?`.
    - `close({result})` closes the current screen.
  - **Naming**: route names are `<microAppId>.<screen>`. Paths are URLs, absolute at the top level and relative when nested.
  - **Parameters are text**: path parameters plus query parameters, so every route is deep-linkable.
  - **Outcomes**: `NavigationResult` is `RouteCompleted(result)`, `RouteCancelled()`, or a sealed `NavigationFailure` (`RouteNotFound`, `RouteParametersMissing`).
  - **Results**: result maps still follow the inter-module payload rule.
- **Rationale**:
  - The route contract is complete enough for Core to build any router, with nested routes, shell routes, presentation, guards, and deep links, while micro apps never import a router package (SC-015).
  - Navigating by name keeps callers stable when paths change, and paths serve deep links.
  - Text parameters are what URLs can carry, so any screen opened in-app can also be opened from a link or a push notification.
  - Groups give `switchable_state`'s `StateScope.group` a natural home.
- **Alternatives considered**:
  - Routes typed by the router (`MicroApp<GoRoute>`, as in the earlier package): rejected because micro apps would depend on the router vendor.
  - Navigating by path string: rejected because callers break when paths change.
  - `Map<String, Object?>` parameters (the earlier decision): rejected because objects can't survive a deep link.
  - Typed route objects shared through Foundations: rejected because they would pull feature models into Foundations (FR-005).

## R16. Quality tooling

- **Decision**:
  - Linting: `analysis_options.yaml` includes `package:flutter_lints/flutter.yaml` and adds `public_member_api_docs`, `always_declare_return_types`, `prefer_final_fields`, and `avoid_dynamic_calls`. It turns on `strict-casts`, `strict-inference`, and `strict-raw-types`. CI requires `flutter analyze` to report 0 issues.
  - Architecture tests:
    - Parse `pubspec.yaml` with the `yaml` package (a dev dependency) and check that runtime dependencies are a subset of `{flutter, design_system, switchable_state}`. No `switchable_state_*` adapter, no state library, and no router package (such as `go_router`) may appear outside `dev_dependencies` (SC-001, FR-043, FR-053).
    - Scan `lib/` for `static` fields that are not `const` and for top-level `var`/`late` declarations (SC-012).
  - Coverage: at least 90% line coverage of the concrete code in `lib/` (value types, extensions, conformance), using `flutter test --coverage`.
  - Documentation: `dart doc` must build with no warnings.
  - Test notation (user convention, 2026-11-22): every test is written in Gherkin notation. Each file starts with a `// Feature:` comment, `group('Given …')` holds the context, each `test('When …, Then …')` states one behavior, and the body is split into `// Given`, `// When`, and `// Then` sections. Test names alone then read as a specification.
- **Rationale**: These turn SC-001, SC-006, and SC-012 into automated checks instead of review-only rules.
- **Alternatives considered**: `very_good_analysis`: a viable choice, but `flutter_lints` plus explicit additions keeps the rule set small and official.

## R17. Versioning and releases

- **Decision**:
  - Semantic versioning, starting at `0.1.0`. Releases are git tags `vX.Y.Z` in the Foundations repo.
  - While the version is below 1.0.0, a breaking change bumps the minor version (pub convention); from 1.0.0 on, it bumps the major version.
  - `CHANGELOG.md` lists breaking changes with migration notes, the delivered `design_system` tag, and the delivered `switchable_state` version.
  - A public element must be `@Deprecated` for at least one minor release before it is removed (FR-033).
- **Rationale**: Consumers pin a Foundations tag the same way Foundations pins the Design System tag, which keeps one version in effect everywhere.
- **Alternatives considered**: Calendar versioning: rejected because it can't express whether a change is compatible.

## R18. Analytics, crash reporting, and logging shapes

- **Decision**:
  - **Analytics**: `logEvent(String name, {Map<String, Object> properties})` and `logScreenView(String screenName, {Map<String, Object> properties})`. Property values are `String`, `num`, or `bool`.
  - **Crash reporting**: `recordError(Object error, StackTrace? stackTrace, {String? reason, Map<String, Object> context})` and `setUserIdentifier(String? pseudonymousId)`.
  - **Logging**: a single `log(LogLevel level, String message, {Object? error, StackTrace? stackTrace, Map<String, Object> context})`. `debug/info/warning/error` are convenience extension methods.
  - Every method returns `void` or `Future<void>` and never fails toward the caller, so no-op and console implementations are trivial to write (FR-026).
- **Rationale**: Implementers write only one logging method. Callers get familiar shortcuts, and the vendor-neutral properties are primitive-only.
- **Alternatives considered**: Re-exporting `package:logging`: rejected because it is a third-party dependency and brings global loggers.

## R19. State-management contract (`switchable_state`)

This decision was added on 2026-11-20 because the user requires the dependency.

- **What the package is**: `switchable_state` 0.1.0 is Flutter-only and MIT-licensed, and its only runtime dependency is `flutter`. It defines one state-management contract:
  - `StateScope`: the root scope takes an adapter. `StateScope.group` holds state for a route group.
  - `AppState<S>`: declares app-wide or grouped state.
  - `StateStore<S>`: `value`, `replace`, `update`, `watch`, `select`, `close`.
  - `StateStoreOwner`: stores owned by one screen.
  - `StateAdapter`: the interface each adapter implements.
  
  Separate adapter packages map the contract onto Cubit, GetX, MobX, Provider, or Riverpod. `switchable_state_test` checks adapters for conformance. All seven packages are at 0.1.0 and need SDK `^3.13.0` and Flutter `>=3.47.0`.
- **Decision**:
  1. `pubspec.yaml` depends on `switchable_state: 0.1.0`, pinned exactly, so every host and micro app resolves the same version (FR-042).
  2. `lib/foundations.dart` re-exports it unchanged: `export 'package:switchable_state/switchable_state.dart';`. There are no wrappers, no `show`/`hide` lists, and no renames.
  3. Foundations depends on no adapter at runtime (FR-043). `switchable_state_riverpod` and `switchable_state_bloc` are `dev_dependencies` only. They run the sample micro app's widget tests under two libraries (SC-013). Cubit is included on purpose: its stream delivery causes a rebuild one frame later, which catches tests that wrongly assume synchronous rebuilds.
  4. The host chooses the adapter. Core's initialization configuration names it, and the host places the root `StateScope(adapter: …)` above its router ([contracts/micro-app.md](contracts/micro-app.md#host-sequence)).
  5. Foundations' own capability contracts keep exposing changes as Dart `Stream`s (`SessionContext.changes`, `Permissions.changes`). They are not exposed as `StateStore`s.
- **Rationale**:
  - The package follows the same pattern as the Design System: one first-party package, delivered unchanged at one version. That keeps micro apps on one dependency and lets "change one line at startup" apply to every micro app.
  - Keeping adapters out of Foundations keeps vendor state libraries out of its runtime graph (FR-003, FR-004).
  - Session and permissions stay as streams because a `StateStore` exposes `replace` and `update`. Any consumer could then overwrite the session, which only Core may change. A micro app that wants a store can copy the stream into one of its own.
- **Alternatives considered**:
  - Foundations picks one adapter, such as Riverpod: rejected because it defeats the package's purpose and adds a vendor dependency.
  - Foundations wraps `switchable_state` in its own abstraction: rejected because the user asked for the package itself, and a second contract layer would duplicate it.
  - A caret constraint (`^0.1.0`): rejected for now. Separate lockfiles (the App Shell and each example) could resolve different 0.1.x patches, which breaks the single-version rule. Revisit at 1.0.
  - Exposing session and permissions as `AppState`/`StateStore`: rejected for the reason above.
- **Effect on micro apps**: Micro apps keep app-wide, route-group, and screen state only through the re-exported contract. They never depend on `switchable_state`, an adapter, or a state library at runtime. Their widget tests may add any adapter as a development-only dependency.

## R20. Adapting `flutter_enterprise_architecture`

This decision was added on 2026-11-21, at the user's request: take the main idea of the user's earlier package, without the tenant complexity. Its broker corresponds to inter-module communication.

- **What the package is**: version 0.0.11, Flutter only, no runtime dependencies. It has these parts:
  - `AbstractMicroappContainer<T>`: name, guarded and unguarded routes of the router's type `T`, `brokers`, localization delegates, `bootstrap()`, `onTenantLifecycleChanged`, `registerDependencies` (GetIt), `remoteConfig`, `commonStoredKeys`, and `buidActionParsers`.
  - `AppShellContainer`: a singleton that registers tenants and microapps. Its `bootstrap()` initializes every microapp, collects routes, delegates, and keys (checking for duplicates), and calls `broker.registerSet(...)`. It also handles tenant switching.
  - The broker: `IBroker` with `isRegistered`, `registerSet`, `callService`, and `callWidget`. Calls are callback-based. Parameters are `Map<String, String>`, data is `String`, and there is an `ICStatusResponses` enum. `BrokerInterface` classes carry `route`, `requiredParameters`, and `receive`. `ICDestination` carries an optional `trace`, which `PerformanceTraceLogger` times. A `ValueNotifier` observer allows streaming.
- **Adopted**:

| Idea | Foundations decision | Requirements |
|------|----------------------|--------------|
| Microapps declare `brokers`, and the shell registers them all (`registerSet`) | `MicroApp.dataEndpoints` / `widgetEndpoints` hold class-based `DataEndpointProvider` / `WidgetEndpointProvider`. The host calls `registerProviders` once, after every `initialize`. | FR-029, FR-046, FR-047 |
| A `BrokerInterface` is a class with `route` + `requiredParameters` + `receive` | A provider declares an `endpoint` (URL + required parameters) and implements `handle` or `build`. Unit tests call it directly, with no Core (SC-014). | FR-046 |
| `IBroker.isRegistered(route)` | `InterModuleCommunication.isRegistered({url, type})`, covered by conformance case C17 | FR-010 |
| `guardedRoutes` / `unguardedRoutes` | `MicroAppRoute.requiresAuthentication`, default `true`. The host guards these routes with `SessionContext`. | FR-048 |
| Routes of the router's type `T`, which the shell turns into the router | A router-neutral route contract (`MicroAppRoute` with name, path pattern, builder, presentation, and children, plus `MicroAppRouteGroup`). Core builds the real router from it ([contracts/routing.md](contracts/routing.md)). This is the user's stated intent. | FR-053, R15 |
| `commonStoredKeys` with duplicate detection | `MicroApp.storedKeys`: duplicates stop startup, and each micro app's stores are limited to its declared keys, so no micro app can overwrite another's values. This is the user's stated purpose. | FR-052, R12 |
| `bootstrap()` + `onTenantLifecycleChanged(disposed)` | `initialize(context)` + `dispose()`. `dispose` runs in reverse order. | FR-029, FR-049 |
| `ICDestination.trace` + `PerformanceTraceLogger` | `InterModuleEndpoint.traceName` only. Core times traced calls and records their name, duration, and status in its own diagnostics. Per the user (2026-11-21), no performance-tracing capability is added, and the capability set stays as it was. | FR-050 |
| `RemoteConfigController.stream` | `FeatureFlags.changes` | FR-051 |
| `AppShellContainer.bootstrap` aggregation and duplicate checks | The host sequence in [contracts/micro-app.md](contracts/micro-app.md#host-sequence). Its reusable implementation is assigned to **Core** (the article gives "shared startup" to Core). This is input for Core's spec. | Assumptions |

- **Not adopted**:

| Idea | Why not |
|------|---------|
| Tenants: `TenantAbstraction`, `TenantContainer`, `changeTenant`, `TenantLifecycle`, the multitenant broker prefixing | The user asked to leave tenants out. |
| `Broker.shared`, and registering the broker and router in GetIt | These are global access points, which clarify Q4 ruled out. The contracts arrive through `MicroAppContext`. |
| `registerDependencies` / `InjectableDependency` / `DependencyResolver` | The pattern assumes a global container. A micro app builds its internal graph in `initialize` from the context. Standalone examples override pieces through the micro app's constructor, for example `AccountsMicroApp(repository: FakeAccountsRepository())`. This covers the article's "selects dependencies" without a container. |
| Routes typed by the router (`AbstractMicroappContainer<GoRoute>`) | Micro apps would depend on `go_router`, a vendor. Its intent, routes that become the real router, is adopted through Foundations' route contract instead (see the adopted table). |
| Callback-based calls; `Map<String, String>` parameters; `String` data; the `message` field | `Future`s and structured primitive payloads are already in the contract. A free-text message risks leaking internals (FR-011, 500). |
| `ICStatusResponses` values such as `errorInvalidCommunicatorType` | Separate registries mean a call for the wrong kind returns 404. The clarify session fixed the four statuses. |
| `ValueNotifier` observer (provider-to-consumer streaming) | Deferred. It would add a third call style beyond the article's `request` / `getWidget`, and it needs defined cancellation and lifecycle rules (the earlier package's `hasListeners` cleanup is fragile). If a cross-module live-data need appears, propose a `watch` operation that returns a `Stream` as a separate, compatible change. |
| `buidActionParsers` | Product-specific server-driven UI, with no counterpart in this project. |
| `GherkinTestHelper` | A general test utility, not an architecture contract. It belongs in a test-utilities package if it is wanted. |

- **Rationale**: The adopted ideas make integration *declarative*. The host sees every route, provider, and endpoint before startup completes, providers can be tested in isolation, and late registration can't happen. Everything adopted is consistent with the decisions already made: no globals, a router-neutral contract, the four statuses, and the article's two call styles.

## R21. Interim Design System scope

This decision was added on 2026-11-21 (FR-054, FR-055).

- **Decision**:
  - **Structure**: atoms (`DsButton`, `DsTextField`, `DsLoadingIndicator`), molecules (`DsCard`, `DsListTile`, `DsInlineMessage`, `DsEmptyState`), and a template (`DsPageScaffold`). There are no organisms yet. The full surface is in [contracts/design-system.md](contracts/design-system.md).
  - **Tokens, in two layers**:
    - Primitive tokens are fixed: `DsPalette`, `DsSpacing`, `DsRadius`.
    - Semantic tokens are resolved per theme and brand: `DsColors` and `DsTypography`, installed as Flutter `ThemeExtension`s.
    - Components read semantic tokens only, through `context.dsColors` and `context.dsTypography`.
  - **One brand configuration**: `DsBrand(primary, fontFamily?)`. `DsTheme.light(brand:)` and `DsTheme.dark(brand:)` derive every role from it, and `textOnAction` is chosen so it meets 4.5:1 contrast on the brand color.
  - **No third-party runtime dependencies**: no `google_fonts`, because fonts come from the host's assets and fall back to the platform font. No `intl` either; money and date formatting belong to micro apps or a later Design System feature.
  - **Accessibility is tested, not just intended**: every component's widget tests run `flutter_test`'s `androidTapTargetGuideline`, `iOSTapTargetGuideline`, `labeledTapTargetGuideline`, and `textContrastGuideline` in both themes, plus a 200% text-scale overflow check at 320 logical pixels wide (D1–D4).
  - **Gallery**: `packages/design_system/example/` renders every component in every state, with light/dark and brand toggles. It moves with the package.
  - **Naming**: every public name starts with `Ds`, to avoid Material, Foundations, and `switchable_state` clashes. Classes are `final`, so the package is extended by composition, not subclassing.
- **Why these eight components**: they are the smallest set that covers the current micro apps' first screens. Sign-in uses the text field, button, and page template. The Accounts list uses the list tile and card. The Transfers form uses the text field, button, and inline message. The Home fallback and federated banner slot use the empty state, inline message, and card. Every screen needs loading states.
- **Alternatives considered**:
  - Using Material widgets directly with only a theme: rejected because it gives no stable component API to keep when the package moves, and no place to enforce the accessibility rules.
  - A larger first set (date pickers, bottom sheets, tabs): deferred to the Design System's own feature after the move.
  - Golden (screenshot) tests: deferred, because they are brittle across platforms and font versions at this stage. The guideline tests above cover the measurable rules.
