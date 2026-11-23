# Contract: Routing

This contract covers FR-023, FR-048, and FR-053. Foundations defines what a route **is** and how modules **navigate**. Core uses these declarations to build the application's actual router (for example go_router). Micro apps never import a router package, so Core can change its router without any micro app changing (SC-015).

This replaces the earlier `flutter_enterprise_architecture` approach, where routes were typed by the router (`AbstractMicroappContainer<GoRoute>`) and the guarded and unguarded routes were separate lists.

Navigation is by **path** (decision 2026-11-23): callers pass a concrete location such as `/accounts/details/a1?tab=history`, and may pass a `body` of extra values. Routes have no separate name.

## Route declarations

```dart
/// Builds a route's screen.
/// - parameters: the path parameters and query parameters, all as text and
///   merged into one map, so every route can be opened from a deep link.
/// - body: the extra values the caller passed. It is empty when the route was
///   opened from a deep link, so the screen must work, or recover, without it.
typedef MicroAppRouteBuilder =
    Widget Function(
      BuildContext context,
      Map<String, String> parameters,
      Map<String, Object> body,
    );

enum RoutePresentation { page, fullscreenDialog }

/// A node in a micro app's route tree. Core matches on it exhaustively.
sealed class MicroAppRouteNode { const MicroAppRouteNode(); }

/// One screen.
final class MicroAppRoute extends MicroAppRouteNode {
  const MicroAppRoute({
    required String path,              // URL pattern with ':param' segments. See the path rules below.
    required MicroAppRouteBuilder builder,
    bool requiresAuthentication = true,                      // FR-048: secure by default
    RoutePresentation presentation = RoutePresentation.page,
    List<MicroAppRouteNode> routes = const [],               // child routes
  });
  // fields: path, builder, requiresAuthentication, presentation, routes
}

/// Routes that share a surrounding layout (for example a tab scaffold) that
/// stays in place while the user moves between them. The builder may also
/// place a StateScope.group(owns: [...]) so grouped state lives exactly as
/// long as the user stays in the group.
final class MicroAppRouteGroup extends MicroAppRouteNode {
  const MicroAppRouteGroup({
    required Widget Function(BuildContext context, Widget child) builder,
    required List<MicroAppRouteNode> routes,
  });
  // fields: builder, routes
}
```

### Path rules

| Where the route is declared | Path form | Example |
|-----------------------------|-----------|---------|
| Directly in `MicroApp.routes`, or directly inside a `MicroAppRouteGroup` | Absolute, starting with `/<microAppId>` | `/accounts` |
| Inside another route's `routes` | Relative to the parent (no leading `/`) | `details/:accountId`, which gives `/accounts/details/:accountId` |

- **Path parameters** are the `:name` segments. A full path can't repeat a parameter name.
- **Query parameters** are the location's query string (`?tab=history`).
- **Builder input**: the builder receives path and query parameters merged into one map. Their names must not overlap. The `body` arrives separately.
- **Effective sign-in requirement**: a route needs sign-in if it, or any ancestor route, has `requiresAuthentication: true`.
- **Public paths**: a full path that another micro app navigates to is part of the owning micro app's public surface, like an endpoint URL. Changing it is a breaking change for its callers.

### Startup checks (performed by the host through Core)

| Check | On failure |
|-------|-----------|
| Every full path is unique in the app, and top-level paths start with `/<microAppId>` | Startup stops, naming the path |

## Navigation

```dart
/// How a screen opened with push or replace closed.
@immutable
sealed class NavigationResult { const NavigationResult(); }

final class RouteCompleted extends NavigationResult {
  const RouteCompleted(this.result);
  final Map<String, Object> result;            // payload rule applies across micro apps
}
final class RouteCancelled extends NavigationResult { const RouteCancelled(); }

// lib/src/exceptions/navigation_exception.dart
/// Navigation that could not start. Sealed, so its variants share its file.
@immutable
sealed class NavigationException implements Exception { const NavigationException(); }
final class RouteNotFoundException extends NavigationException {
  const RouteNotFoundException(this.path);
  final String path;
}

abstract interface class AppNavigator {
  /// Opens path on top of the current screen. Completes when that screen
  /// closes: Ok(RouteCompleted) or Ok(RouteCancelled).
  Future<Result<NavigationResult, NavigationException>> push(String path, {Map<String, Object> body = const {}});

  /// Goes to path, replacing the whole stack (for example "back to home after
  /// a transfer", or switching tabs).
  Result<void, NavigationException> go(String path, {Map<String, Object> body = const {}});

  /// Replaces the current screen with path. Completes like push.
  Future<Result<NavigationResult, NavigationException>> replace(String path, {Map<String, Object> body = const {}});

  /// Closes the current screen. A non-null result completes the opener with
  /// RouteCompleted(result). A null result completes it with RouteCancelled.
  void pop({Map<String, Object>? result});

  /// Closes screens until the one at path is on top. Each closed screen's
  /// opener completes with RouteCancelled. If no screen in the stack is at
  /// path, closes nothing and returns Err(RouteNotFoundException(path)).
  Result<void, NavigationException> popUntil(String path);
}
```

`body` values sent to, and results returned from, another micro app's screen follow the payload rule (`isInterModuleValue`), so the caller imports none of that micro app's types. Within one micro app, a body may carry that micro app's own types.

### Navigation outcomes

| Situation | `push` / `replace` | `go` | `popUntil` |
|-----------|--------------------|------|------------|
| The location matches no declared route | `Err(RouteNotFoundException(path))` | `Err(RouteNotFoundException(path))` | — |
| No screen in the stack is at `path` | — | — | `Err(RouteNotFoundException(path))`, nothing closed |
| The route needs sign-in and the session is `Unauthenticated` | The host sends the user to sign-in. If sign-in is abandoned, `Ok(RouteCancelled())` | The host sends the user to sign-in | — |
| The opened screen calls `pop(result: {...})` | `Ok(RouteCompleted(result))` | — | — |
| The opened screen closes without a result (or the user presses system back) | `Ok(RouteCancelled())` | — | — |
| A deep link matches no full path | The host's "not found" screen. No caller is involved | — | — |

## How Core implements it (guidance for Core's spec)

| Foundations declaration | go_router equivalent (illustrative) |
|-------------------------|-------------------------------------|
| `MicroAppRoute(path, builder, routes)` | `GoRoute(path: path, builder: (c, s) => builder(c, {...s.pathParameters, ...s.uri.queryParameters}, (s.extra as Map<String, Object>?) ?? const {}), routes: …)` |
| `presentation: fullscreenDialog` | `pageBuilder` returning a `MaterialPage(fullscreenDialog: true, …)` |
| `MicroAppRouteGroup(builder, routes)` | `ShellRoute(builder: (c, s, child) => builder(c, child), routes: …)` |
| Effective `requiresAuthentication` | A `redirect` that checks `SessionContext.current` |
| `push` / `go` / `replace` | `push(path, extra: body)` awaiting a result / `go(path, extra: body)` / `pushReplacement(path, extra: body)`, after checking the location matches a route |
| `pop` / `popUntil` | `pop(result)` / popping while the top location differs from `path` |

The table is only illustrative. Any router that honors the declarations and outcomes satisfies the contract.
