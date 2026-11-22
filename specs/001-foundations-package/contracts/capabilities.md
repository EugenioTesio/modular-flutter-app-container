# Contract: Capability Abstractions

This contract covers FR-019 through FR-028, FR-039 through FR-041, and FR-051. Every capability is an `abstract interface class` that Core implements. Hosts supply implementations through `MicroAppContext` (see [micro-app.md](micro-app.md)), and tests supply hand-written fakes (FR-027). None of these signatures contains a vendor type.

## Common

```dart
sealed class Result<T, F> { const Result(); }
final class Ok<T, F> extends Result<T, F> { const Ok(this.value); final T value; }
final class Err<T, F> extends Result<T, F> { const Err(this.failure); final F failure; }
```

## Application information (FR-019)

```dart
enum AppEnvironment { development, staging, production }

abstract interface class AppInfo {
  String get appName;
  String get version;        // e.g. '1.4.0'
  String get buildNumber;
  AppEnvironment get environment;
}
```

## API client (FR-039)

```dart
enum ApiMethod { get, post, put, patch, delete }

@immutable
final class ApiRequest {
  const ApiRequest({
    required ApiMethod method,
    required String path,                        // relative, starts with '/'
    Map<String, String> query = const {},
    Map<String, String> headers = const {},      // credential headers are set by the implementation
    Object? body,                                // JSON-encodable
  });
  // fields: method, path, query, headers, body
}

@immutable
final class ApiResponse {
  const ApiResponse({required int statusCode, Object? body, Map<String, String> headers = const {}});
  // fields: statusCode, body (decoded JSON), headers
}

enum ApiFailureKind { noConnectivity, timeout, unauthorized, clientError, serverError }

@immutable
final class ApiFailure {
  const ApiFailure({required ApiFailureKind kind, int? statusCode, Object? body});
  // fields: kind, statusCode, body
}

abstract interface class ApiClient {
  /// Resolves path against the current environment's base address and attaches
  /// the session's credentials. Never throws: every failure becomes an Err with one kind.
  Future<Result<ApiResponse, ApiFailure>> send(ApiRequest request);
}

extension ApiClientShortcuts on ApiClient {
  Future<Result<ApiResponse, ApiFailure>> get(String path, {Map<String, String> query, Map<String, String> headers});
  Future<Result<ApiResponse, ApiFailure>> post(String path, {Object? body, Map<String, String> query, Map<String, String> headers});
  Future<Result<ApiResponse, ApiFailure>> put(String path, {Object? body, Map<String, String> query, Map<String, String> headers});
  Future<Result<ApiResponse, ApiFailure>> patch(String path, {Object? body, Map<String, String> query, Map<String, String> headers});
  Future<Result<ApiResponse, ApiFailure>> delete(String path, {Map<String, String> query, Map<String, String> headers});
}
```

Implementations must classify failures this way:

| Condition | `ApiFailureKind` |
|-----------|------------------|
| No connection or DNS failure | `noConnectivity` |
| The deadline was exceeded | `timeout` |
| HTTP 401 | `unauthorized` |
| Any other HTTP 4xx | `clientError` |
| HTTP 5xx, or a response that can't be decoded | `serverError` |

## Analytics (FR-020)

```dart
abstract interface class Analytics {
  /// Property values: String, num, or bool. Never fails toward the caller.
  Future<void> logEvent(String name, {Map<String, Object> properties = const {}});
  Future<void> logScreenView(String screenName, {Map<String, Object> properties = const {}});
}
```

## Crash and error reporting (FR-021)

```dart
abstract interface class CrashReporter {
  Future<void> recordError(
    Object error,
    StackTrace? stackTrace, {
    String? reason,
    Map<String, Object> context = const {},
  });

  /// Pseudonymous identifier only. Pass null to clear it.
  Future<void> setUserIdentifier(String? pseudonymousId);
}
```

## Logging (FR-041)

```dart
enum LogLevel { debug, info, warning, error }

/// Log entries must never contain credentials or personal data.
abstract interface class AppLogger {
  void log(
    LogLevel level,
    String message, {
    Object? error,
    StackTrace? stackTrace,
    Map<String, Object> context = const {},
  });
}

extension AppLoggerShortcuts on AppLogger {
  void debug(String message, {Map<String, Object> context});
  void info(String message, {Map<String, Object> context});
  void warning(String message, {Object? error, StackTrace? stackTrace, Map<String, Object> context});
  void error(String message, {Object? error, StackTrace? stackTrace, Map<String, Object> context});
}
```

## Feature flags (FR-040)

```dart
sealed class FeatureFlag<T extends Object> {
  const FeatureFlag({required String key, required T defaultValue}); // key: '<microAppId>.<flag_name>'
  final String key;
  final T defaultValue;
}
final class BoolFlag extends FeatureFlag<bool> { const BoolFlag({required super.key, required super.defaultValue}); }
final class StringFlag extends FeatureFlag<String> { const StringFlag({required super.key, required super.defaultValue}); }
final class NumberFlag extends FeatureFlag<num> { const NumberFlag({required super.key, required super.defaultValue}); }

abstract interface class FeatureFlags {
  /// Returns flag.defaultValue when the flag is unknown, has a value of the
  /// wrong kind, or the provider is unavailable. Never throws.
  T valueOf<T extends Object>(FeatureFlag<T> flag);

  /// Emits after flag values are refreshed (for example after a remote config
  /// fetch). Consumers re-read the flags they care about (FR-051).
  /// This is the counterpart of RemoteConfigController.stream in flutter_enterprise_architecture.
  Stream<void> get changes;
}
```

There is no performance-tracing capability, by decision (FR-050). The only tracing is the optional `traceName` on inter-module calls, which Core records in its own diagnostics ([inter-module-communication.md](inter-module-communication.md), row B14).

## Storage (FR-022)

```dart
@immutable
final class StorageFailure {
  const StorageFailure({required StorageFailureKind kind, required String key});
}

enum StorageFailureKind { unavailable, readFailed, writeFailed, deleteFailed, undeclaredKey }

/// General-purpose storage. Core gives each micro app its own instance, limited
/// to the keys that micro app declares in MicroApp.storedKeys (FR-052). Any
/// operation on another key returns Err(StorageFailure(kind: undeclaredKey)).
/// Keys should follow '<microAppId>.<key>'. Declaring a key, not its prefix, is what reserves it.
abstract interface class KeyValueStore {
  Future<Result<String?, StorageFailure>> read(String key);     // Ok(null) when the key is absent
  Future<Result<void, StorageFailure>> write(String key, String value);
  Future<Result<void, StorageFailure>> delete(String key);
  Future<Result<bool, StorageFailure>> containsKey(String key);
}

/// Storage for sensitive values. It is limited to the same declared keys.
/// Can be used wherever a KeyValueStore is expected, but not the other way round.
abstract interface class SecureKeyValueStore implements KeyValueStore {}
```

## Navigation (FR-023)

`AppNavigator`, `NavigationResult`, and the route declarations it navigates to are defined in [routing.md](routing.md). In short:
- `open(routeName, parameters)` returns `Future<NavigationResult>`, which is `RouteCompleted`, `RouteCancelled`, `RouteNotFound`, or `RouteParametersMissing`.
- `go(routeName, parameters)` returns a `NavigationFailure?`.
- `close(result)` closes the current screen.

Route parameters are text, so every route is reachable by deep link.

## Session context (FR-024)

```dart
@immutable
final class UserIdentity {
  const UserIdentity({required String userId, String? displayName});
}

sealed class SessionState { const SessionState(); }
final class Authenticated extends SessionState {
  const Authenticated(this.identity);
  final UserIdentity identity;
}
final class Unauthenticated extends SessionState { const Unauthenticated(); }

abstract interface class SessionContext {
  SessionState get current;                    // never null
  Stream<SessionState> get changes;            // emits on sign-in, sign-out, expiry, and user switch
}
```

## Permissions (FR-025)

```dart
@immutable
final class Permission {
  const Permission(this.name);                 // '<microAppId>.<action>'
  final String name;
  // == and hashCode are based on name
}

/// Permissions that more than one module checks. Starts empty. A
/// module-owned permission moves here when a second module needs it.
abstract final class SharedPermissions {}

abstract interface class Permissions {
  bool has(Permission permission);
  Stream<void> get changes;                    // emits whenever entitlements reload
}
```

## Fakes and no-op implementations (FR-026, FR-027)

| Contract | What a no-op or console implementation must do |
|----------|-----------------------------------------------|
| `Analytics`, `CrashReporter`, `AppLogger` | Accept every call and complete normally, with no observable effect (no-op) or by printing to the console. |
| `FeatureFlags` | Always return `flag.defaultValue`. `changes` never emits. |
| All others | Tests implement them with in-memory fakes. The interfaces are small enough to fake in a few lines without a device, platform channels, network access, or Core. |
