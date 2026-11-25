# Core Handoff: What Core's Spec Must Cover

Foundations 0.1.0 defines contracts only. Every implementation belongs to Core. This note lists what Core's specification must cover so that Core fulfills those contracts. Each item names its source in this feature's documents. The example host in `foundations/example/lib/host/` is a minimal working version of most of it, and a useful starting point.

## 1. The host sequence, through `ModularAppContainer`

Foundations provides `ModularAppContainer`, the counterpart of `AppShellContainer` in `flutter_enterprise_architecture`, without tenants, a route type parameter, or a global instance ([contracts/micro-app.md](contracts/micro-app.md#modularappcontainer); FR-057). It runs steps 1–5 and 7 below and throws a `ModularAppStartupException` for wiring mistakes. Core doesn't re-implement it. Core's startup:

- supplies the `MicroAppContextFactory` (section 2);
- calls `bootstrap(communication:, contextFor:, hostPaths:)`, with the paths of Core's own screens (sign-in, not-found) as `hostPaths`;
- builds the router from `container.routes`, and guards it with `container.requiresAuthentication(state.fullPath)` (section 3);
- passes `container.localizationsDelegates` to the app;
- calls `container.dispose()` on teardown.

The sequence, for reference:

1. Read every micro app's `id` and `storedKeys`. A duplicate id, or a key declared by two micro apps, stops startup, naming the key and both micro apps.
2. Build one `MicroAppContext` per micro app (see section 2).
3. `await initialize(context)` for each micro app, in a fixed, documented order.
4. Collect `routes`, `localizationsDelegates`, `dataEndpoints`, and `widgetEndpoints`. Validate the full paths of the route trees: no duplicate full path, every top-level path starts with `/<microAppId>`, and no repeated path parameter name.
5. Call `communication.registerProviders(data: …, widgets: …)` once. A `DuplicateEndpointException` stops startup. From here on, the set of endpoints is fixed (section 5).
6. Build the router (section 3) and the localizations, and place the root `StateScope(adapter: …)` above the router. The adapter comes from Core's configuration (section 7).
7. On teardown, `await dispose()` for each micro app in reverse order. A failure is reported, and the remaining teardowns still run.

`foundations/test/micro_app/modular_app_container_test.dart` and `modular_app_route_checks_test.dart` cover these rules. The reference host in `foundations/test/support/reference_host.dart` and the example host show the container wired to a go_router router.

## 2. One context per micro app, with stores limited to its keys

- Core builds a separate `MicroAppContext` for each micro app. The shared services are the same instances for every micro app: communication, API client, analytics, crash reporter, logger, feature flags, navigator, session, device permissions, and app info.
- `keyValueStore` and `secureKeyValueStore` are built per micro app, and accept only the keys that micro app declared in `storedKeys`. Any other key returns `Err(StorageException(type: undeclaredKey))` and leaves the stored value unchanged ([contracts/capabilities.md](contracts/capabilities.md); FR-022, FR-052; SC-016).
- There are no global instances or service locators. Each environment (development, staging, production) chooses its implementations in Core's initialization configuration (FR-016).
- Credentials and tokens stay in Core. The API client attaches them and overrides any header of the same name. They never appear in `SessionState`, logs, analytics, or crash reports ([contracts/capabilities.md](contracts/capabilities.md); FR-041).
- Core maps every `DevicePermission` variant onto the host's permission plugins with an exhaustive `switch`. Adding a variant in Foundations therefore needs a matching Core release.

## 3. A router built from the route contract

Core builds the application's router from the micro apps' route trees, following [contracts/routing.md](contracts/routing.md). Micro apps never import a router package. The router must:

- Merge path parameters and query parameters into one map, and pass the `push` body separately; a deep link gets an empty body.
- Present `RoutePresentation.fullscreenDialog` routes as full-screen dialogs.
- Keep a `MicroAppRouteGroup`'s layout, and any `StateScope.group` inside it, while the user moves between its routes, and release them when the user leaves.
- Guard every route whose effective `requiresAuthentication` is true (its own flag, or an ancestor's) by sending an unauthenticated user to sign-in. The default is `true`.
- Implement `AppNavigator` with the outcomes in routing.md: `RouteNotFoundException` for an unknown path and for `popUntil` on a path not in the stack, `RouteCompleted` or `RouteCancelled` for `push` and `replace`, and the host's not-found screen for an unmatched deep link.
- Read the stack from the router (`routerDelegate.currentConfiguration` for go_router) instead of keeping a copy, and complete with `RouteCancelled` the opener of every screen that `go`, `replace`, or `popUntil` closes. go_router completes a pushed screen's future only when that screen is popped.

The go_router mapping in routing.md and the reference adapter in `foundations/test/support/reference_router.dart` show one way to do it; its tests in `test/routing/reference_router_test.dart` list the behaviors Core's router must pass.

## 4. `traceName` diagnostics

When a call's endpoint has a `traceName`, Core times the call and records the name, the duration, and the resulting status (including 404 and 500) in its own diagnostics. A call without a `traceName` is not traced (behavior B14 in [contracts/inter-module-communication.md](contracts/inter-module-communication.md); FR-050; research R20).

- The conformance check can't observe Core's diagnostics, so Core's own tests must cover B14.
- There is no performance-tracing capability in Foundations, by decision. Core must not add one to `MicroAppContext`.
- Recorded traces follow the privacy rule: the name, duration, and status only, never parameter values or response data.

## 5. Rejecting `register` calls after startup

After the host's single `registerProviders` call (step 5), the set of endpoints is fixed. Core's communication implementation must reject any later `register` or `registerWidget` call, so a 404 `endpoint_not_registered` always means "that micro app isn't installed" ([research R8](research.md#r8-endpoint-registration-lifecycle)).

- The Foundations contract has no "startup complete" signal, so the conformance check doesn't test this. Core's spec must define how its implementation learns that startup finished, and Core's tests must cover the rejection.
- Rejecting a late registration is a wiring mistake, so throwing is appropriate, as it is for `DuplicateEndpointException`.

## 6. Passing `package:foundations/testing.dart`

Core's `InterModuleCommunication` implementation must pass every case in `InterModuleConformance.cases` (C01–C17) in Core's own test suite ([contracts/conformance.md](contracts/conformance.md); FR-017; SC-003):

```dart
import 'package:foundations/testing.dart';

for (final c in InterModuleConformance.cases) {
  test('When conformance case ${c.id} runs, Then it passes', () async {
    final result = await c.run(CoreInterModuleCommunication.new);
    expect(result.passed, isTrue, reason: result.message);
  });
}
```

Each run must build a fresh instance, as the factory argument requires. Standalone examples that ship their own stub implementation run the same cases.

## 7. Running `switchable_state_test` on the chosen adapter

The host chooses the state-management adapter, and Core's initialization configuration names it. Foundations depends on no adapter ([research R19](research.md#r19-state-management-contract-switchable_state); FR-043).

- Whichever adapter Core's configuration supports, Core's tests must run the `switchable_state_test` conformance suite against it, so every micro app gets the behavior the contract promises.
- Core pins the adapter at the version that matches Foundations' pinned `switchable_state` (0.1.0 today). A breaking `switchable_state` release means a breaking Foundations release (FR-045), and Core must upgrade the adapter in the same step.
- Widget tests should allow for adapters that rebuild a frame later, such as Cubit, by calling `pumpAndSettle` after writes.

## Also carry over

- **Versioning**: Core depends on a Foundations tag (`ref: vX.Y.Z`), never a branch, so Core, the App Shell, and every micro app resolve one version of Foundations, `design_system`, and `switchable_state` (research R3, R17).
- **Payload rule**: providers return only values for which `InterModuleValidator.isValidPayload` is true. Core may assert this in debug builds, and must not convert payloads.
- **Failures**: a provider that throws becomes status 500 with no exception text for the caller (B8, B9). Core's spec should decide where the original error goes, for example its crash reporter, without parameter values or response data.
