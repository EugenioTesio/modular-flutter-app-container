# Data Model: Foundations Package

Foundations stores no data. Its "data model" is the set of value types, states, and rules that its contracts exchange. Exact signatures are in [contracts/](contracts/). Each entry below names the spec requirements it satisfies.

## Shared rules

- **Payload rule** (`isInterModuleValue`): a value passes if it is `null`, a `bool`, `num`, or `String`, a `List` whose elements all pass, or a `Map<String, …>` whose values all pass. This rule applies to inter-module parameters, inter-module data, and the navigation bodies and results exchanged with another micro app's screen (FR-009, FR-023). A navigation *path* is stricter: its parameters are text, so every location stays deep-linkable.
- **Immutability**: every value type is `@immutable` with value equality. Map and set fields are compared by content, one level deep, with Flutter's `mapEquals` and `setEquals`. Other fields, including `Object?` bodies and nested collections, are compared with `==`.
- **Exceptions**: every error type is named `…Exception`, `implements Exception`, and lives in `lib/src/exceptions/`. Contracts return them in an `Err` instead of throwing, except `DuplicateEndpointException`.
- **Key and identifier convention**: identifiers that a module owns are prefixed with the micro app id: `<microAppId>.<name>` for flags and storage keys, and `/<microAppId>/…` for route paths and endpoint URLs. Storage keys are also *declared* (`storedKeys`), and the declaration is what enforces ownership.

## Inter-module communication

### InterModuleEndpoint (FR-007, FR-013)

| Field | Type | Rules |
|-------|------|-------|
| `url` | `String` | Not empty and starts with `/`. Convention: `/<module>/v<N>/<name>` (FR-018). It is the key in the in-process registry. |
| `parameters` | `Map<String, Object?>` | Defaults to `{}`. Every value follows the payload rule. Parameters the provider did not declare are passed through unchanged. |
| `requiredParameters` | `Set<String>` | Defaults to `{}`. It only matters on the endpoint passed to `register` or `registerWidget`. On a consumer's call it is **ignored**, because validation uses the provider's stored declaration. |
| `traceName` | `String?` | Optional. It only matters on a consumer's call. When set, the implementation times the call and records the name (`<microAppId>.<name>`), duration, and status in its own diagnostics. It is ignored at registration (FR-050). |

### InterModuleCommunicationResponse\<T\> (FR-008, FR-011)

| Field | Type | Rules |
|-------|------|-------|
| `statusCode` | `int` | One of the published values: 200, 400, 404, or 500. Consumers treat any status outside 200–299 as a failure. |
| `data` | `T?` | `T` is `Map<String, Object?>` for data endpoints and `Widget` for UI endpoints. It is not null when the status is 200 (enforced by `.success`). It is null on any failure. |
| `errorCode` | `String?` | Null on success. On a 400 or 404 produced by Core it is one of the shared codes. Providers may use their own codes, such as `invalid_id` or `details_not_found`. |
| `isSuccess` | `bool` (derived) | `200 <= statusCode < 300` |

**Shared error codes** (FR-012):

| Code | Status | Produced by |
|------|--------|-------------|
| `missing_required_parameters` | 400 | Core, before the provider runs |
| `endpoint_not_registered` | 404 | Core |
| `unauthenticated` | 400 | The provider, when the user is not signed in |
| `forbidden` | 400 | The provider, when the user is not allowed to make the call (the providing micro app decides) |

### Registration

| Entity | Fields | Rules |
|--------|--------|-------|
| `EndpointType` | `data`, `widget` | Each type has its own registry, so the same `url` can appear once in each (FR-014). |
| `DuplicateEndpointException` | `url`, `type` | Thrown by `register` or `registerWidget` when the `url` already exists in the registry for that type. Hosts should let it stop startup. |
| `InterModuleHandler` | function `(Map<String, Object?> parameters) → Future<Response<Map<String, Object?>>>` | It is invoked only after the required-parameter check passes. When it throws or its future fails, the caller gets a 500. |
| `InterModuleWidgetFactory` | function `(Map<String, Object?> parameters) → Response<Widget>` | It runs synchronously and returns its element without waiting for the element's data (FR-015). When it throws, the caller gets a 500. |
| `DataEndpointProvider` | `endpoint: InterModuleEndpoint`, `handle(parameters)` | A class that a micro app declares in `dataEndpoints`. `endpoint.url` and `endpoint.requiredParameters` are its registration. It can be called directly in unit tests (FR-046). |
| `WidgetEndpointProvider` | `endpoint: InterModuleEndpoint`, `build(parameters)` | The same, for UI endpoints, declared in `widgetEndpoints`. |

**Lifecycle**: Hosts register every declared provider in one step (`registerProviders`), after all micro apps have initialized and before the app runs. A registration lasts for the life of the app. There is no unregistration, and micro apps never call `register` directly ([research R8](research.md#r8-endpoint-registration-lifecycle)). `isRegistered(url, type)` reports registrations without invoking any provider.

## Common outcome type

### Result\<T, F\> (FR-028)

`sealed`. It has exactly two variants: `Ok(value: T)` and `Err(failure: F)`. It is used by the API client and by storage.

## Capability types

### AppInfo (FR-019)

| Field | Type | Rules |
|-------|------|-------|
| `appName` | `String` | Not empty |
| `version` | `String` | Semantic version string, such as `1.4.0` |
| `buildNumber` | `String` | Not empty |
| `environment` | `AppInfoEnvironment` | One of `development`, `staging`, or `production` |

### API client (FR-039)

| Entity | Fields | Rules |
|--------|--------|-------|
| `ApiMethod` | `get`, `post`, `put`, `patch`, `delete` | |
| `ApiRequest` | `method`, `path`, `queryParams: Map<String, String>`, `headers: Map<String, String>`, `body: Object?` | `path` is relative and starts with `/`. `body` must be JSON-encodable. The implementation sets credential headers and overrides any header with the same name that the caller supplies. |
| `ApiResponse` | `statusCode: int`, `body: Object?` (decoded JSON), `headers: Map<String, String>` | It is returned only for 2xx and 3xx responses. |
| `ApiException` | `type: ApiExceptionType`, `statusCode: int?`, `body: Object?` | Each failure has exactly one type (see the table below). |
| `ApiExceptionType` | Connection: `noConnectivity`, `timeout`, `badCertificate`. Backend answer: `unauthorized`, `clientError`, `serverError`. The client itself: `invalidRequest`, `invalidResponse`, `unknown` | See the mapping table below. |

How failures map to `ApiExceptionType` ([research R10](research.md#r10-api-client-contract)):

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

### Analytics, crash reporting, logging (FR-020, FR-021, FR-041)

| Entity | Fields | Rules |
|--------|--------|-------|
| Analytics event | `name: String`, `properties: Map<String, Object>` | Property values are `String`, `num`, or `bool` only. |
| Screen view | `screenName: String`, `properties` | Same rule as events |
| Error report | `error: Object`, `stackTrace: StackTrace?`, `reason: String?`, `context: Map<String, Object>` | The user identifier is a pseudonymous `String?` set with `setUserIdentifier`. |
| `LogLevel` | `debug`, `info`, `warning`, `error` | Ordered from lowest to highest. |
| Log entry | `level`, `message`, `error?`, `stackTrace?`, `context` | It never contains credentials or personal data (a documented rule). |

### FeatureFlag\<T\> (FR-040)

`sealed`. Its variants are `BoolFeatureFlag` (`T = bool`), `StringFeatureFlag` (`T = String`), and `NumberFeatureFlag` (`T = num`).

| Field | Type | Rules |
|-------|------|-------|
| `key` | `String` | Follows `<microAppId>.<flag_name>` |
| `defaultValue` | `T` | Returned when the flag is unknown, holds a value of the wrong kind, or the provider is unavailable. |

`FeatureFlags.changes` emits after every refresh of flag values (FR-051). Consumers then re-read the flags they need.

### Storage (FR-022)

| Entity | Fields | Rules |
|--------|--------|-------|
| Storage entry | `key: String`, `value: String` | The key must be one of the owning micro app's `storedKeys` (FR-052). It should follow `<microAppId>.<key>`, but legacy keys may be declared as they are. Values are opaque strings that each micro app serializes itself. |
| Stored key | `String`, declared in `MicroApp.storedKeys` | Declared by exactly one micro app in the host, and covers both general and secure storage. A duplicate stops startup, naming the key and both micro apps. |
| `StorageException` | `type: StorageExceptionType`, `key: String` | `StorageExceptionType` is one of `unavailable`, `readFailed`, `writeFailed`, `deleteFailed`, or `undeclaredKey`. `undeclaredKey` is returned for any operation on a key the calling micro app did not declare. |

`SecureKeyValueStore` holds entries the same way, but in secure storage. It implements `KeyValueStore`, so a secure store can be used where a general store is expected. The reverse is not allowed. Core gives each micro app its own instances of both stores, limited to its declared keys.

### Navigation (FR-023)

The route declarations are in [Routing](#routing-fr-023-fr-048-fr-053). Navigation uses them as follows:

| Entity | Fields | Rules |
|--------|--------|-------|
| Navigation path | `String` | A concrete location, such as `/accounts/details/a1?tab=history`: path parameters filled in, query parameters attached. The same location works as a deep link. |
| Navigation body | `Map<String, Object>` | Defaults to `{}`. Extra values for the destination screen. Follows the payload rule when the screen belongs to another micro app. Empty when the screen is opened from a deep link. |
| `NavigationResult` | `sealed`: `RouteCompleted(result: Map<String, Object>)` or `RouteCancelled()` | How a pushed or replaced screen closed. `result` follows the payload rule across micro apps. |
| `NavigationException` | `sealed`, `implements Exception`: `RouteNotFoundException(path)` | Returned in an `Err` by `push`, `go`, and `replace` when the path matches no route, and by `popUntil` when no screen in the stack is at the path. |

Operation results: `push` and `replace` → `Future<Result<NavigationResult, NavigationException>>`; `go` and `popUntil` → `Result<void, NavigationException>`; `pop({result})` → `void`.

State transitions of a `push` call:

```text
push(path) ──► path matches no route ───────────────────────► Err(RouteNotFoundException)
          ├──► needs sign-in while Unauthenticated ──► sign-in abandoned ──► Ok(RouteCancelled)
          └──► screen shown ──► pop(result: {...}) ─────────► Ok(RouteCompleted(result))
                           ├──► pop() / system back ────────► Ok(RouteCancelled)
                           └──► closed by popUntil ─────────► Ok(RouteCancelled)
```

### Session (FR-024)

| Entity | Fields | Rules |
|--------|--------|-------|
| `SessionState` | `sealed`: `Authenticated(identity)` or `Unauthenticated()` | It is never null or absent. |
| `UserIdentity` | `userId: String` (opaque), `displayName: String?`, `email: String?`, `phoneNumber: String?`, `customProperties: Map<String, Object?>` (defaults to `{}`) | It never holds credentials or tokens; Core keeps them (FR-039). `customProperties` follows the payload rule (for example an address or a customer segment). Profile fields are personal data: never logged, tracked, or reported. `toString` shows only `userId`. |
| `Authenticated` | `identity: UserIdentity` | It carries no session, access, or refresh tokens. |

State transitions (each one emits on `changes`):

```text
Unauthenticated ──sign-in──► Authenticated(A)
Authenticated(A) ──sign-out / expiry──► Unauthenticated
Authenticated(A) ──user switch──► Authenticated(B)
```

### Device permissions (FR-025)

| Entity | Fields | Rules |
|--------|--------|-------|
| `DevicePermission` | `sealed`, one variant per `permission_handler` permission (see the list below) | Equal when the variant and its options are equal. Core maps every variant exhaustively. A platform-only permission returns `unsupported` on the other platform. |
| `PhotosDevicePermission` | `access: PhotosDevicePermissionAccess` (`readWrite` default, `addOnly` iOS) | |
| `StorageDevicePermission` | `scope: StorageDevicePermissionScope` (`shared` default, `manageAll`) | Android only. `shared` is always denied on Android 13+; use photos, videos, or audio files there. |
| `CalendarDevicePermission` | `access: CalendarDevicePermissionAccess` (`full` default, `writeOnly`) | |
| `LocationDevicePermission` | `access: LocationDevicePermissionAccess` (`whileInUse` default, `always`), `accuracy: LocationDevicePermissionAccuracy` (`precise` default, `approximate`), `temporaryPrecisePurposeKey: String?` | The options `geolocator` needs. `always` is requested after `whileInUse`. On iOS 14+, when only approximate location was allowed and `precise` is asked for, Core requests temporary precise location with the purpose key. |
| `SensorsDevicePermission` | `access: SensorsDevicePermissionAccess` (`whileInUse` default, `always`) | |
| `BluetoothDevicePermission` | `access: BluetoothDevicePermissionAccess` (`connect` default, `scan`, `advertise`) | Separate on Android 12+; one permission on iOS and older Android. |
| `DevicePermissionStatus` | `granted`, `limited`, `provisional`, `denied`, `permanentlyDenied`, `restricted`, `unsupported` | `request` shows no prompt when the status is `granted`, `permanentlyDenied`, `restricted`, or `unsupported`. On Android a status check reports `denied` for a permanently denied permission; only a request reveals `permanentlyDenied`. |
| `DevicePermissionServiceStatus` | `enabled`, `disabled`, `notApplicable` | For location, Bluetooth, and phone. A granted permission with a `disabled` service still can't be used. |

Variants without options: `CameraDevicePermission`, `MicrophoneDevicePermission`, `VideosDevicePermission`, `AudioFilesDevicePermission`, `MediaLibraryDevicePermission` (iOS), `MediaLocationDevicePermission` (Android), `ContactsDevicePermission`, `RemindersDevicePermission` (iOS), `SpeechRecognitionDevicePermission`, `AssistantDevicePermission` (iOS), `ActivityRecognitionDevicePermission`, `NearbyWifiDevicesDevicePermission` (Android 13+), `LocalNetworkDevicePermission` (iOS 14+), `NotificationsDevicePermission`, `CriticalAlertsDevicePermission` (iOS), `NotificationPolicyDevicePermission` (Android), `PhoneDevicePermission` (Android), `SmsDevicePermission` (Android), `AppTrackingDevicePermission` (iOS), `BackgroundRefreshDevicePermission` (iOS, read only), `IgnoreBatteryOptimizationsDevicePermission` (Android), `ScheduleExactAlarmDevicePermission` (Android 12+), `SystemAlertWindowDevicePermission` (Android), and `InstallPackagesDevicePermission` (Android).

Business entitlements (what the signed-in user may do) are not modeled in Foundations. Each micro app handles them through its own backend.

## Micro app integration

### MicroApp (FR-029–FR-031)

| Field | Type | Rules |
|-------|------|-------|
| `id` | `String` | Matches `^[a-z][a-z0-9_]*$` and is unique within a host. It is the prefix for every identifier the micro app owns. It is read before `initialize`. |
| `storedKeys` | `Set<String>` | Read before `initialize`. Every storage key the micro app uses, in general or secure storage. Each key is unique across the host (FR-052). |
| `initialize` | `Future<void> Function(MicroAppContext)` | Called exactly once per host, with that micro app's own context, before any declaration below is read. It builds the micro app's internal dependencies. It must not depend on other modules' endpoints, which aren't registered yet. |
| `routes` | `List<MicroAppRouteNode>` | The route tree ([Routing](#routing-fr-023-fr-048-fr-053)). Read by the host after `initialize`. Names and full paths are checked for uniqueness across the app. |
| `localizationsDelegates` | `List<LocalizationsDelegate<Object?>>` | It may be empty. |
| `dataEndpoints` | `List<DataEndpointProvider>` | Read after `initialize` and registered by the host in bulk. URLs follow `/<microAppId>/v<N>/<name>`. |
| `widgetEndpoints` | `List<WidgetEndpointProvider>` | The same, for UI endpoints. |
| `dispose` | `Future<void> Function()` | Called once on host teardown, in reverse initialization order. It releases what `initialize` acquired (FR-049). |

### MicroAppContext (FR-016, FR-029, FR-052)

This bundles every contract. Core builds **one per micro app**: most fields are shared instances, but `keyValueStore` and `secureKeyValueStore` are limited to that micro app's `storedKeys`. **All fields are required**, so a host that forgets a contract fails to compile.

| Field | Contract |
|-------|----------|
| `communication` | `InterModuleCommunication` |
| `apiClient` | `ApiClient` |
| `analytics` | `Analytics` |
| `crashReporter` | `CrashReporter` |
| `logger` | `AppLogger` |
| `featureFlags` | `FeatureFlags` |
| `keyValueStore` | `KeyValueStore` |
| `secureKeyValueStore` | `SecureKeyValueStore` |
| `navigator` | `AppNavigator` |
| `session` | `SessionContext` |
| `devicePermissions` | `DevicePermissions` |
| `appInfo` | `AppInfo` |

Host lifecycle:

```text
host reads id and storedKeys from every micro app. A duplicate id or key stops startup.
  → Core initialization builds one MicroAppContext per micro app (storage limited to its keys)
  → host calls initialize(itsContext) on each micro app, in order
  → host collects route trees, localization delegates, dataEndpoints, and widgetEndpoints,
    then checks full paths. A duplicate stops startup.
  → host calls communication.registerProviders(...) once. A duplicate URL stops startup.
  → Core builds the router from the route trees. The app runs. Registries are fixed from here on,
    and guarded routes need a session.
  → on teardown: dispose() on each micro app, in reverse order
```

## Routing (FR-023, FR-048, FR-053)

Exact signatures and Core's mapping guidance are in [contracts/routing.md](contracts/routing.md).

### MicroAppRoute

| Field | Type | Rules |
|-------|------|-------|
| `path` | `String` | A URL pattern with `:param` segments. Absolute (`/<microAppId>/…`) when declared at the top level or directly in a group, and relative to the parent when nested. The full path is unique in the app, parameter names don't repeat within it, and it is what `AppNavigator` navigates to. |
| `builder` | `Widget Function(BuildContext, Map<String, String> parameters, Map<String, Object> body)` | Receives the path and query parameters merged (their names must not overlap) and the caller's body, which is empty after a deep link. |
| `requiresAuthentication` | `bool` | Defaults to `true`. It takes effect if the route or any ancestor sets it. While the session is `Unauthenticated`, the host redirects to sign-in (FR-048). |
| `presentation` | `RoutePresentation` | `page` (default) or `fullscreenDialog` |
| `routes` | `List<MicroAppRouteNode>` | Child routes. Their paths are relative. |

### MicroAppRouteGroup

| Field | Type | Rules |
|-------|------|-------|
| `builder` | `Widget Function(BuildContext, Widget child)` | The shared layout, such as tabs. It may wrap `child` in `StateScope.group(owns: …)` so grouped state lives while the user stays in the group. |
| `routes` | `List<MicroAppRouteNode>` | Routes declared directly in a group use absolute paths. |

## Delivered packages (re-exported unchanged)

Foundations does not define these types. It re-exports them so micro apps get them through their single Foundations dependency.

### Design System (FR-002, FR-036–FR-038)

For now this is the **interim** `design_system` package at `foundations/packages/design_system/`, which moves to its own repository later (FR-054–FR-056). Exact signatures are in [contracts/design-system.md](contracts/design-system.md).

| Layer | Types | Rules |
|-------|-------|-------|
| Primitive tokens | `DsPalette` (neutral, success, warning, and error scales), `DsSpacing` (4 · 8 · 12 · 16 · 24 · 32 · 48), `DsRadius` (4 · 8 · 16 · pill) | Fixed values. Components never read the palette directly. |
| Brand | `DsBrand(primary, fontFamily?)` | The single brand configuration. Both themes derive from it. |
| Semantic tokens | `DsColors` (background, surface, border, text roles, action roles, focus, and the information, success, warning, and error background and foreground pairs), `DsTypography` (display, headline, title, body, bodyStrong, label, caption) | `ThemeExtension`s, resolved per theme and brand. Text roles reach at least 4.5:1 contrast on their backgrounds, and `actionPrimary` reaches at least 3:1 against `background`. |
| Themes | `DsTheme.light(brand:)`, `DsTheme.dark(brand:)` | They install `DsColors` and `DsTypography` and align the Material component themes. |
| Atoms | `DsButton` (primary, secondary, and tertiary variants; loading; disabled), `DsTextField` (label, hint, helper, error), `DsLoadingIndicator` | Accessibility rules D1–D7 |
| Molecules | `DsCard`, `DsListTile`, `DsInlineMessage` (information, success, warning, and error tones), `DsEmptyState` | Accessibility rules D1–D4 and D9 |
| Templates | `DsPageScaffold` (title, body, optional primary action, back) | Safe area, `DsSpacing.md` padding |

### State-management contract: `switchable_state` 0.1.0 (FR-042–FR-045)

| Type | Role | Who uses it |
|------|------|-------------|
| `StateScope` | Root scope (takes an adapter) or route-group scope (`StateScope.group(owns: …)`) | The host places the root scope. Micro apps may place group scopes, for example in a shell route. |
| `AppState<S>` | Declares a piece of state: `AppState(...)` for the whole app, or `AppState.grouped(...)` for a route group | Micro apps |
| `StateStore<S>` | Holds one immutable snapshot: `value`, `replace`, `update`, `watch`, `select`, `close` | Micro apps |
| `StateStoreOwner` | A mixin on `State`. `createStore` makes a store owned by one screen | Micro apps |
| `StateAdapter` | Maps the contract onto one library | Implemented by adapter packages. The host chooses one, and Foundations never names one |
| `StateOverride` | Seeds a state's initial value, usually in tests | Micro app tests |

Ownership rules (from the package):

| Declared with | Owned by | Closed when |
|---------------|----------|-------------|
| `AppState(...)` | The root scope | The app is disposed |
| `AppState.grouped(...)` | The `StateScope.group` that lists it | The user leaves the group |
| `createStore(...)` | One screen | That screen's `State` is disposed |

Rules that apply to micro apps:
- State values are the micro app's own immutable types. They are never placed in Foundations (FR-005).
- A micro app never constructs an adapter. The adapter is the host's choice (FR-043, FR-044).

## Conformance

| Entity | Fields | Rules |
|--------|--------|-------|
| `ConformanceCase` | `id: String` (`C01`…), `description: String`, `run(create) → Future<ConformanceResult>` | Each run calls `create()` for a fresh implementation, so no state is shared between cases. |
| `ConformanceResult` | `passed: bool`, `message: String?` | `message` explains the failure. |
