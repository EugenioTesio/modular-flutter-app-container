# Data Model: Foundations Package

Foundations stores no data. Its "data model" is the set of value types, states, and rules that its contracts exchange. Exact signatures are in [contracts/](contracts/). Each entry below names the spec requirements it satisfies.

## Shared rules

- **Payload rule** (`isInterModuleValue`): a value passes if it is `null`, a `bool`, `num`, or `String`, a `List` whose elements all pass, or a `Map<String, …>` whose values all pass. This rule applies to inter-module parameters, inter-module data, and navigation results (FR-009, FR-023). Navigation *parameters* are stricter: they are text only, so routes stay deep-linkable.
- **Immutability**: every value type is `@immutable` with value equality. Maps and sets are compared by content.
- **Key and identifier convention**: identifiers that a module owns are prefixed with the micro app id: `<microAppId>.<name>` for flags, storage keys, permissions, and route names, and `/<microAppId>/…` for route paths and endpoint URLs. Storage keys are also *declared* (`storedKeys`), and the declaration is what enforces ownership.

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
| `forbidden` | 400 | The provider, when the user lacks a permission |

### Registration

| Entity | Fields | Rules |
|--------|--------|-------|
| `EndpointKind` | `data`, `widget` | Each kind has its own registry, so the same `url` can appear once in each (FR-014). |
| `DuplicateEndpointError` | `url`, `kind` | Thrown by `register` or `registerWidget` when the `url` already exists in the registry for that kind. Hosts should let it stop startup. |
| `InterModuleHandler` | function `(Map<String, Object?> parameters) → Future<Response<Map<String, Object?>>>` | It is invoked only after the required-parameter check passes. When it throws or its future fails, the caller gets a 500. |
| `InterModuleWidgetFactory` | function `(Map<String, Object?> parameters) → Response<Widget>` | It runs synchronously and returns its element without waiting for the element's data (FR-015). When it throws, the caller gets a 500. |
| `DataEndpointProvider` | `endpoint: InterModuleEndpoint`, `handle(parameters)` | A class that a micro app declares in `dataEndpoints`. `endpoint.url` and `endpoint.requiredParameters` are its registration. It can be called directly in unit tests (FR-046). |
| `WidgetEndpointProvider` | `endpoint: InterModuleEndpoint`, `build(parameters)` | The same, for UI endpoints, declared in `widgetEndpoints`. |

**Lifecycle**: Hosts register every declared provider in one step (`registerProviders`), after all micro apps have initialized and before the app runs. A registration lasts for the life of the app. There is no unregistration, and micro apps never call `register` directly ([research R8](research.md#r8-endpoint-registration-lifecycle)). `isRegistered(url, kind)` reports registrations without invoking any provider.

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
| `environment` | `AppEnvironment` | One of `development`, `staging`, or `production` |

### API client (FR-039)

| Entity | Fields | Rules |
|--------|--------|-------|
| `ApiMethod` | `get`, `post`, `put`, `patch`, `delete` | |
| `ApiRequest` | `method`, `path`, `query: Map<String, String>`, `headers: Map<String, String>`, `body: Object?` | `path` is relative and starts with `/`. `body` must be JSON-encodable. The implementation sets credential headers and overrides any header with the same name that the caller supplies. |
| `ApiResponse` | `statusCode: int`, `body: Object?` (decoded JSON), `headers: Map<String, String>` | It is returned only for 2xx and 3xx responses. |
| `ApiFailure` | `kind: ApiFailureKind`, `statusCode: int?`, `body: Object?` | Each failure has exactly one kind (see the table below). |
| `ApiFailureKind` | `noConnectivity`, `timeout`, `unauthorized`, `clientError`, `serverError` | See the mapping table below. |

How failures map to `ApiFailureKind` ([research R10](research.md#r10-api-client-contract)):

| Condition | `ApiFailureKind` |
|-----------|------------------|
| No connection or DNS failure | `noConnectivity` |
| The deadline was exceeded | `timeout` |
| HTTP 401 | `unauthorized` |
| Any other HTTP 4xx | `clientError` (the `statusCode` and body are kept) |
| HTTP 5xx, or a response body that can't be decoded | `serverError` |

### Analytics, crash reporting, logging (FR-020, FR-021, FR-041)

| Entity | Fields | Rules |
|--------|--------|-------|
| Analytics event | `name: String`, `properties: Map<String, Object>` | Property values are `String`, `num`, or `bool` only. |
| Screen view | `screenName: String`, `properties` | Same rule as events |
| Error report | `error: Object`, `stackTrace: StackTrace?`, `reason: String?`, `context: Map<String, Object>` | The user identifier is a pseudonymous `String?` set with `setUserIdentifier`. |
| `LogLevel` | `debug`, `info`, `warning`, `error` | Ordered from lowest to highest. |
| Log entry | `level`, `message`, `error?`, `stackTrace?`, `context` | It never contains credentials or personal data (a documented rule). |

### FeatureFlag\<T\> (FR-040)

`sealed`. Its variants are `BoolFlag` (`T = bool`), `StringFlag` (`T = String`), and `NumberFlag` (`T = num`).

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
| `StorageFailure` | `kind: StorageFailureKind`, `key: String` | `StorageFailureKind` is one of `unavailable`, `readFailed`, `writeFailed`, `deleteFailed`, or `undeclaredKey`. `undeclaredKey` is returned for any operation on a key the calling micro app did not declare. |

`SecureKeyValueStore` holds entries the same way, but in secure storage. It implements `KeyValueStore`, so a secure store can be used where a general store is expected. The reverse is not allowed. Core gives each micro app its own instances of both stores, limited to its declared keys.

### Navigation (FR-023)

The route declarations are in [Routing](#routing-fr-023-fr-048-fr-053). Navigation uses them as follows:

| Entity | Fields | Rules |
|--------|--------|-------|
| Route name | `String` | Follows `<microAppId>.<screen>` and is unique in the app |
| Navigation parameters | `Map<String, String>` | Text only. Path parameters are filled by name, and every other parameter becomes a query parameter, so the same route is reachable by deep link. |
| `NavigationResult` | `sealed`: `RouteCompleted(result)`, `RouteCancelled()`, or a `NavigationFailure`. A `NavigationFailure` is `sealed`: `RouteNotFound(routeName)` or `RouteParametersMissing(routeName, missing)`. | `result` must follow the payload rule. `go` returns only `NavigationFailure?`. |

State transitions of an `open` call:

```text
open(name) ──► name unknown ────────────────────────────────► RouteNotFound
          ├──► path parameter missing ──────────────────────► RouteParametersMissing
          ├──► needs sign-in while Unauthenticated ──► sign-in abandoned ──► RouteCancelled
          └──► screen shown ──► close(result: {...}) ───────► RouteCompleted(result)
                           └──► close() / system back ──────► RouteCancelled
```

### Session (FR-024)

| Entity | Fields | Rules |
|--------|--------|-------|
| `SessionState` | `sealed`: `Authenticated(identity)` or `Unauthenticated()` | It is never null or absent. |
| `UserIdentity` | `userId: String` (opaque), `displayName: String?` | It never holds credentials or tokens. |

State transitions (each one emits on `changes`):

```text
Unauthenticated ──sign-in──► Authenticated(A)
Authenticated(A) ──sign-out / expiry──► Unauthenticated
Authenticated(A) ──user switch──► Authenticated(B)
```

### Permission (FR-025)

| Field | Type | Rules |
|-------|------|-------|
| `name` | `String` | Follows `<microAppId>.<action>`, for example `transfers.create`. Two permissions are equal when their names are equal. |

Ownership: a permission checked by two or more modules is declared in `SharedPermissions` in Foundations. A permission checked by only one module is declared as a `const` in that micro app. A permission moves into Foundations as soon as a second module needs it.

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
| `permissions` | `Permissions` |
| `appInfo` | `AppInfo` |

Host lifecycle:

```text
host reads id and storedKeys from every micro app. A duplicate id or key stops startup.
  → Core initialization builds one MicroAppContext per micro app (storage limited to its keys)
  → host calls initialize(itsContext) on each micro app, in order
  → host collects route trees, localization delegates, dataEndpoints, and widgetEndpoints,
    then checks route names and full paths. A duplicate stops startup.
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
| `name` | `String` | Follows `<microAppId>.<screen>` and is unique in the app. Used by `AppNavigator`. |
| `path` | `String` | A URL pattern with `:param` segments. Absolute (`/<microAppId>/…`) when declared at the top level or directly in a group, and relative to the parent when nested. The full path is unique in the app, and parameter names don't repeat within it. |
| `builder` | `Widget Function(BuildContext, Map<String, String> parameters)` | Receives the path and query parameters merged. Their names must not overlap. |
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
