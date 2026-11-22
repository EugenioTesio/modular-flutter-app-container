# Contract: Public Surface

Consumers can import only these two libraries (FR-006). Everything under `lib/src/` is private to the package.

## `package:foundations/foundations.dart`

This is the main entry point for micro apps, Core, and hosts.

```dart
library;

// Design System: re-exported unchanged at the pinned tag (FR-002, FR-036, FR-037).
export 'package:design_system/design_system.dart';

// State-management contract: re-exported unchanged at the pinned 0.1.0 (FR-042, FR-044).
// Adapters (switchable_state_bloc/_getx/_mobx/_provider/_riverpod) are NOT exported or depended on (FR-043).
export 'package:switchable_state/switchable_state.dart';

// Common
export 'src/common/result.dart';                           // Result, Ok, Err

// Inter-module communication: see inter-module-communication.md
export 'src/inter_module/inter_module_endpoint.dart';      // InterModuleEndpoint
export 'src/inter_module/inter_module_response.dart';      // InterModuleCommunicationResponse
export 'src/inter_module/inter_module_status.dart';        // InterModuleStatusCode, InterModuleErrorCode
export 'src/inter_module/inter_module_value.dart';         // isInterModuleValue
export 'src/inter_module/inter_module_communication.dart'; // InterModuleCommunication, InterModuleHandler,
                                                           // InterModuleWidgetFactory, EndpointKind
export 'src/inter_module/endpoint_provider.dart';          // DataEndpointProvider, WidgetEndpointProvider,
                                                           // InterModuleProviderRegistration (registerProviders)
export 'src/inter_module/duplicate_endpoint_error.dart';   // DuplicateEndpointError

// Capabilities: see capabilities.md
export 'src/capabilities/app_info.dart';
export 'src/capabilities/api_client.dart';
export 'src/capabilities/analytics.dart';
export 'src/capabilities/crash_reporter.dart';
export 'src/capabilities/app_logger.dart';
export 'src/capabilities/feature_flags.dart';
export 'src/capabilities/key_value_store.dart';
export 'src/capabilities/session_context.dart';
export 'src/capabilities/permissions.dart';

// Routing: see routing.md. AppNavigator and NavigationResult live with the route contract.
export 'src/routing/micro_app_route.dart';     // MicroAppRouteNode, MicroAppRoute, MicroAppRouteGroup,
                                               // MicroAppRouteBuilder, RoutePresentation
export 'src/routing/app_navigator.dart';       // AppNavigator, NavigationResult, RouteCompleted, RouteCancelled,
                                               // NavigationFailure, RouteNotFound, RouteParametersMissing

// Micro app integration: see micro-app.md
export 'src/micro_app/micro_app.dart';
export 'src/micro_app/micro_app_context.dart';
```

For now, `package:design_system/design_system.dart` comes from the interim package at `packages/design_system/` inside this repository ([design-system.md](design-system.md)). When the Design System moves to its own repository, this `export` line does not change. Only the dependency source in `pubspec.yaml` does (FR-056).

## `package:foundations/testing.dart`

This library is for Core's tests and for standalone stubs. It never needs to be imported in production code.

```dart
library;

export 'src/testing/inter_module_conformance.dart'; // InterModuleConformance, ConformanceCase, ConformanceResult
```

## Rules

| Rule | How it is verified |
|------|--------------------|
| No vendor type appears in any exported signature (FR-004) | Review, plus the dependency test: no vendor package is resolvable at runtime |
| Exports from the Design System, `switchable_state`, and Foundations never collide | The compiler reports an `ambiguous_export` error. The architecture test imports `foundations.dart` and references one symbol from each of the three sources. |
| No adapter package or state library is a runtime dependency (FR-043) | The dependency allow-list test (adapters appear only under `dev_dependencies`) |
| Every exported member has a doc comment (FR-032) | The `public_member_api_docs` lint is on, and `flutter analyze` must report 0 issues |
| No exported type has `static` mutable state (FR-016) | The architecture scan in `test/architecture/` |
| Adding an export is a minor change. Removing or renaming one is a breaking change (FR-033) | `CHANGELOG.md` review in every release |

## Naming

Capability names avoid clashes with Flutter SDK types that micro apps import alongside Foundations: `AppNavigator` and not `Navigator`, `AppLogger` and not `Logger`, `Permissions` and not `Permission` (which is the value type).

They also avoid the names `switchable_state` exports: `AppState`, `StateScope`, `StateStore`, `StateAdapter`, `StateOverride`, `StateRegistry`, `StateStoreOwner`, `StateStoreUpdates`, and `closedStoreError`. For example, the session type is `SessionState`, not `AppState`.
