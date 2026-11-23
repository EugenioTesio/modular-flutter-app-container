# Contract: Capability Abstractions

This contract covers FR-019 through FR-028, FR-039 through FR-041, and FR-051. Every capability is an `abstract interface class` that Core implements. Hosts supply implementations through `MicroAppContext` (see [micro-app.md](micro-app.md)), and tests supply hand-written fakes (FR-027). None of these signatures contains a vendor type.

## Common

```dart
sealed class Result<T, F> { const Result(); }
final class Ok<T, F> extends Result<T, F> { const Ok(this.value); final T value; }
final class Err<T, F> extends Result<T, F> { const Err(this.failure); final F failure; }
```

Every error type a contract returns is named `…Exception`, `implements Exception`, and lives in `lib/src/exceptions/`: `ApiException`, `StorageException`, `NavigationException`, and `DuplicateEndpointException`. Contracts return them inside an `Err` rather than throwing them, except `DuplicateEndpointException`, which signals a wiring mistake at startup.

Value equality uses Flutter's own `mapEquals` and `setEquals` for map and set fields (one level, by content). Every other field, including an `Object?` body, is compared with `==`. Foundations has no equality helper of its own (decision 2026-11-23).

## Application information (FR-019)

```dart
enum AppInfoEnvironment { development, staging, production }

abstract interface class AppInfo {
  String get appName;
  String get version;        // e.g. '1.4.0'
  String get buildNumber;
  AppInfoEnvironment get environment;
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
    Map<String, String> queryParams = const {},
    Map<String, String> headers = const {},      // credential headers are set by the implementation
    Object? body,                                // JSON-encodable
  });
  // fields: method, path, queryParams, headers, body
}

@immutable
final class ApiResponse {
  const ApiResponse({required int statusCode, Object? body, Map<String, String> headers = const {}});
  // fields: statusCode, body (decoded JSON), headers
}

enum ApiExceptionType {
  noConnectivity, timeout, badCertificate,          // the connection
  unauthorized, clientError, serverError,           // the backend's answer
  invalidRequest, invalidResponse, unknown,         // the API client itself
}

// lib/src/exceptions/api_exception.dart
@immutable
final class ApiException implements Exception {
  const ApiException({required ApiExceptionType type, int? statusCode, Object? body});
  // fields: type, statusCode, body
}

abstract interface class ApiClient {
  /// Resolves path against the current environment's base address and attaches
  /// the session's credentials. Never throws: every failure becomes an Err with one type.
  Future<Result<ApiResponse, ApiException>> send(ApiRequest request);
}

extension ApiClientShortcuts on ApiClient {
  Future<Result<ApiResponse, ApiException>> get(String path, {Map<String, String> queryParams, Map<String, String> headers});
  Future<Result<ApiResponse, ApiException>> post(String path, {Object? body, Map<String, String> queryParams, Map<String, String> headers});
  Future<Result<ApiResponse, ApiException>> put(String path, {Object? body, Map<String, String> queryParams, Map<String, String> headers});
  Future<Result<ApiResponse, ApiException>> patch(String path, {Object? body, Map<String, String> queryParams, Map<String, String> headers});
  Future<Result<ApiResponse, ApiException>> delete(String path, {Map<String, String> queryParams, Map<String, String> headers});
}
```

Implementations must classify failures this way:

| Condition | `ApiExceptionType` |
|-----------|------------------|
| No connection or DNS failure | `noConnectivity` |
| The deadline was exceeded | `timeout` |
| The TLS handshake failed or the certificate was rejected (for example by pinning) | `badCertificate` |
| HTTP 401 | `unauthorized` |
| Any other HTTP 4xx | `clientError` (the `statusCode` and body are kept) |
| HTTP 5xx | `serverError` (the `statusCode` and body are kept) |
| The request could not be built or encoded (body not JSON-encodable, invalid path, query parameter, or header). No call reached the backend | `invalidRequest` |
| The response could not be decoded (malformed JSON, unexpected content type, wrong shape) | `invalidResponse` (the `statusCode` is kept) |
| Any other error raised inside the client | `unknown` |

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
final class BoolFeatureFlag extends FeatureFlag<bool> { const BoolFeatureFlag({required super.key, required super.defaultValue}); }
final class StringFeatureFlag extends FeatureFlag<String> { const StringFeatureFlag({required super.key, required super.defaultValue}); }
final class NumberFeatureFlag extends FeatureFlag<num> { const NumberFeatureFlag({required super.key, required super.defaultValue}); }

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
// lib/src/exceptions/storage_exception.dart
@immutable
final class StorageException implements Exception {
  const StorageException({required StorageExceptionType type, required String key});
}

enum StorageExceptionType { unavailable, readFailed, writeFailed, deleteFailed, undeclaredKey }

/// General-purpose storage. Core gives each micro app its own instance, limited
/// to the keys that micro app declares in MicroApp.storedKeys (FR-052). Any
/// operation on another key returns Err(StorageException(type: undeclaredKey)).
/// Keys should follow '<microAppId>.<key>'. Declaring a key, not its prefix, is what reserves it.
abstract interface class KeyValueStore {
  Future<Result<String?, StorageException>> read(String key);     // Ok(null) when the key is absent
  Future<Result<void, StorageException>> write(String key, String value);
  Future<Result<void, StorageException>> delete(String key);
  Future<Result<bool, StorageException>> containsKey(String key);
}

/// Storage for sensitive values. It is limited to the same declared keys.
/// Can be used wherever a KeyValueStore is expected, but not the other way round.
abstract interface class SecureKeyValueStore implements KeyValueStore {}
```

## Navigation (FR-023)

`AppNavigator`, `NavigationResult`, and the route declarations it navigates to are defined in [routing.md](routing.md). In short:
- `push(path, body)` and `replace(path, body)` return `Future<Result<NavigationResult, NavigationException>>`. The result is `RouteCompleted` or `RouteCancelled`, and the exception is `RouteNotFoundException`.
- `go(path, body)` and `popUntil(path)` return `Result<void, NavigationException>`.
- `pop(result)` closes the current screen.

A `path` is a concrete location (`/accounts/details/a1?tab=history`), so every route is reachable by deep link. A `body` (`Map<String, Object>`) carries extra values that a deep link doesn't have.

## Session context (FR-024)

```dart
/// Never holds credentials or tokens: Core keeps those and the API client
/// attaches them (FR-039). email, phoneNumber, and customProperties are
/// personal data: shown, never logged, tracked, or reported.
@immutable
final class UserIdentity {
  const UserIdentity({
    required String userId,
    String? displayName,
    String? email,
    String? phoneNumber,
    Map<String, Object?> customProperties = const {},   // payload rule; e.g. address, segment
  });
  // toString shows only userId
}

sealed class SessionState { const SessionState(); }
/// Carries no session, access, or refresh tokens (FR-039).
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

Tokens stay in Core by decision (2026-11-23). If a micro app ever needs one, for example for a third-party SDK, it gets a narrow, explicit call added for that purpose, not a field on the session.

## Device permissions (FR-025)

Core implements this on top of the host's permission plugins, for example `permission_handler`, plus `geolocator` for location. Micro apps never import those plugins. Permissions that need extra options carry them as fields of their variant. Business entitlements, such as "may this user create a transfer", are not part of Foundations: each micro app handles them through its own backend (decision 2026-11-23).

```dart
@immutable
sealed class DevicePermission { const DevicePermission(); }   // == by type and options

// Camera, microphone, and media
final class CameraDevicePermission extends DevicePermission {}
final class MicrophoneDevicePermission extends DevicePermission {}
enum PhotosDevicePermissionAccess { readWrite, addOnly }               // addOnly: iOS
final class PhotosDevicePermission extends DevicePermission { const PhotosDevicePermission({PhotosDevicePermissionAccess access = readWrite}); }
final class VideosDevicePermission extends DevicePermission {}          // Android 13+
final class AudioFilesDevicePermission extends DevicePermission {}      // Android 13+
final class MediaLibraryDevicePermission extends DevicePermission {}    // iOS
final class MediaLocationDevicePermission extends DevicePermission {}   // Android
enum StorageDevicePermissionScope { shared, manageAll }
final class StorageDevicePermission extends DevicePermission { const StorageDevicePermission({StorageDevicePermissionScope scope = shared}); } // Android

// Personal information
final class ContactsDevicePermission extends DevicePermission {}
enum CalendarDevicePermissionAccess { full, writeOnly }
final class CalendarDevicePermission extends DevicePermission { const CalendarDevicePermission({CalendarDevicePermissionAccess access = full}); }
final class RemindersDevicePermission extends DevicePermission {}       // iOS
final class SpeechRecognitionDevicePermission extends DevicePermission {}
final class AssistantDevicePermission extends DevicePermission {}       // iOS (SiriKit)

// Location and motion
enum LocationDevicePermissionAccess { whileInUse, always }
enum LocationDevicePermissionAccuracy { precise, approximate }
final class LocationDevicePermission extends DevicePermission {
  const LocationDevicePermission({
    LocationDevicePermissionAccess access = whileInUse,
    LocationDevicePermissionAccuracy accuracy = precise,
    String? temporaryPrecisePurposeKey,                // iOS 14+ Info.plist purpose key
  });
}
enum SensorsDevicePermissionAccess { whileInUse, always }
final class SensorsDevicePermission extends DevicePermission { const SensorsDevicePermission({SensorsDevicePermissionAccess access = whileInUse}); }
final class ActivityRecognitionDevicePermission extends DevicePermission {}

// Connectivity
enum BluetoothDevicePermissionAccess { scan, connect, advertise }
final class BluetoothDevicePermission extends DevicePermission { const BluetoothDevicePermission({BluetoothDevicePermissionAccess access = connect}); }
final class NearbyWifiDevicesDevicePermission extends DevicePermission {}  // Android 13+
final class LocalNetworkDevicePermission extends DevicePermission {}       // iOS 14+

// Notifications
final class NotificationsDevicePermission extends DevicePermission {}
final class CriticalAlertsDevicePermission extends DevicePermission {}     // iOS
final class NotificationPolicyDevicePermission extends DevicePermission {} // Android (Do Not Disturb)

// Telephony (Android)
final class PhoneDevicePermission extends DevicePermission {}
final class SmsDevicePermission extends DevicePermission {}

// Privacy and system
final class AppTrackingDevicePermission extends DevicePermission {}               // iOS 14+
final class BackgroundRefreshDevicePermission extends DevicePermission {}         // iOS, read only
final class IgnoreBatteryOptimizationsDevicePermission extends DevicePermission {} // Android
final class ScheduleExactAlarmDevicePermission extends DevicePermission {}        // Android 12+
final class SystemAlertWindowDevicePermission extends DevicePermission {}         // Android
final class InstallPackagesDevicePermission extends DevicePermission {}           // Android

enum DevicePermissionStatus { granted, limited, provisional, denied, permanentlyDenied, restricted, unsupported }
enum DevicePermissionServiceStatus { enabled, disabled, notApplicable }

abstract interface class DevicePermissions {
  /// Never prompts. On Android a permanently denied permission reports denied.
  Future<DevicePermissionStatus> status(DevicePermission permission);

  /// Prompts only when the system allows it. No prompt when the status is
  /// granted, permanentlyDenied, restricted, or unsupported.
  Future<DevicePermissionStatus> request(DevicePermission permission);

  /// Whether the system service behind the permission (location, Bluetooth,
  /// phone) is switched on. notApplicable for every other permission.
  Future<DevicePermissionServiceStatus> serviceStatus(DevicePermission permission);

  /// Opens the app's page in the system settings. Returns whether it opened.
  Future<bool> openAppSettings();
}
```

The variants mirror the permissions `permission_handler` exposes (researched on 2026-11-23), with options where the plugin splits one capability into several permissions. A permission that exists on only one platform returns `unsupported` on the other, and the feature decides whether it can work without it. `permission_handler`'s `unknown` value and its deprecated `calendar` and `locationAlways`/`locationWhenInUse` constants are covered by the options above.

Adding a variant (for example a permission a future platform version introduces) is a minor change for micro apps but needs a matching mapping in Core, because Core switches over `DevicePermission` exhaustively.

## Fakes and no-op implementations (FR-026, FR-027)

| Contract | What a no-op or console implementation must do |
|----------|-----------------------------------------------|
| `Analytics`, `CrashReporter`, `AppLogger` | Accept every call and complete normally, with no observable effect (no-op) or by printing to the console. |
| `FeatureFlags` | Always return `flag.defaultValue`. `changes` never emits. |
| All others | Tests implement them with in-memory fakes. The interfaces are small enough to fake in a few lines without a device, platform channels, network access, or Core. |
