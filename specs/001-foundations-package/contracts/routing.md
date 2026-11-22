# Contract: Routing

This contract covers FR-023, FR-048, and FR-053. Foundations defines what a route **is** and how modules **navigate**. Core uses these declarations to build the application's actual router (for example go_router). Micro apps never import a router package, so Core can change its router without any micro app changing (SC-015).

This replaces the earlier `flutter_enterprise_architecture` approach, where routes were typed by the router (`AbstractMicroappContainer<GoRoute>`) and the guarded and unguarded routes were separate lists.

## Route declarations

```dart
/// Builds a route's screen. parameters holds the path parameters and the query
/// parameters, all as text, so every route can be opened from a deep link.
typedef MicroAppRouteBuilder =
    Widget Function(BuildContext context, Map<String, String> parameters);

enum RoutePresentation { page, fullscreenDialog }

/// A node in a micro app's route tree. Core matches on it exhaustively.
sealed class MicroAppRouteNode { const MicroAppRouteNode(); }

/// One screen.
final class MicroAppRoute extends MicroAppRouteNode {
  const MicroAppRoute({
    required String name,              // '<microAppId>.<screen>', unique in the app. Used by AppNavigator.
    required String path,              // URL pattern with ':param' segments. See the path rules below.
    required MicroAppRouteBuilder builder,
    bool requiresAuthentication = true,                      // FR-048: secure by default
    RoutePresentation presentation = RoutePresentation.page,
    List<MicroAppRouteNode> routes = const [],               // child routes
  });
  // fields: name, path, builder, requiresAuthentication, presentation, routes
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
- **Query parameters** are every other parameter passed when opening the route. They become the URL's query string.
- **Builder input**: the builder receives path and query parameters merged into one map. Their names must not overlap.
- **Effective sign-in requirement**: a route needs sign-in if it, or any ancestor route, has `requiresAuthentication: true`.

### Startup checks (performed by the host through Core)

| Check | On failure |
|-------|-----------|
| Every route `name` is unique in the app and starts with `<microAppId>.` | Startup stops, naming the duplicate or the wrong prefix |
| Every full path is unique in the app, and top-level paths start with `/<microAppId>` | Startup stops, naming the path |

## Navigation

```dart
sealed class NavigationResult { const NavigationResult(); }

final class RouteCompleted extends NavigationResult {
  const RouteCompleted(this.result);
  final Map<String, Object?> result;            // payload rule applies
}
final class RouteCancelled extends NavigationResult { const RouteCancelled(); }

/// Navigation that could not start.
sealed class NavigationFailure extends NavigationResult { const NavigationFailure(); }
final class RouteNotFound extends NavigationFailure {
  const RouteNotFound(this.routeName);
  final String routeName;
}
final class RouteParametersMissing extends NavigationFailure {
  const RouteParametersMissing({required this.routeName, required this.missing});
  final String routeName;
  final Set<String> missing;                    // path parameters not supplied
}

abstract interface class AppNavigator {
  /// Opens routeName on top of the current screen. Completes when that screen
  /// closes, or immediately with a NavigationFailure.
  Future<NavigationResult> open(String routeName, {Map<String, String> parameters = const {}});

  /// Goes to routeName, replacing the current location (for example "back to
  /// home after a transfer", or switching tabs). Returns null when navigation
  /// started, or the reason it could not.
  NavigationFailure? go(String routeName, {Map<String, String> parameters = const {}});

  /// Closes the current screen. A non-null result completes the opener with
  /// RouteCompleted(result). A null result completes it with RouteCancelled.
  void close({Map<String, Object?>? result});
}
```

### Navigation outcomes

| Situation | `open` | `go` |
|-----------|--------|------|
| Unknown `routeName` | `RouteNotFound` | `RouteNotFound` |
| A path parameter is missing from `parameters` | `RouteParametersMissing(missing: {...})` | `RouteParametersMissing` |
| The route needs sign-in and the session is `Unauthenticated` | The host sends the user to sign-in. If sign-in is abandoned, `RouteCancelled` | The host sends the user to sign-in |
| The opened screen calls `close(result: {...})` | `RouteCompleted(result)` | — |
| The opened screen closes without a result (or the user presses system back) | `RouteCancelled` | — |
| A deep link matches no full path, or lacks a path parameter | The host's "not found" screen. No caller is involved | — |

## How Core implements it (guidance for Core's spec)

| Foundations declaration | go_router equivalent (illustrative) |
|-------------------------|-------------------------------------|
| `MicroAppRoute(name, path, builder, routes)` | `GoRoute(name: name, path: path, builder: (c, s) => builder(c, {...s.pathParameters, ...s.uri.queryParameters}), routes: …)` |
| `presentation: fullscreenDialog` | `pageBuilder` returning a `MaterialPage(fullscreenDialog: true, …)` |
| `MicroAppRouteGroup(builder, routes)` | `ShellRoute(builder: (c, s, child) => builder(c, child), routes: …)` |
| Effective `requiresAuthentication` | A `redirect` that checks `SessionContext.current` |
| `open` / `go` / `close` | `pushNamed` awaiting a result / `goNamed` / `pop(result)`, wrapped so the outcomes above are honored |

The table is only illustrative. Any router that honors the declarations and outcomes satisfies the contract.
