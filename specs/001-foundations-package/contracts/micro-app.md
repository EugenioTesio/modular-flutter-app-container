# Contract: Micro App Integration

This contract covers FR-016, FR-029 through FR-031, FR-046 through FR-049, and FR-052. It gives every host, both the App Shell and each standalone example, one uniform way to integrate a micro app. Routes are declared with the route contract in [routing.md](routing.md).

It adapts the main idea of the user's earlier `flutter_enterprise_architecture` package, without tenants:

| `flutter_enterprise_architecture` | Here |
|-----------------------------------|------|
| `AbstractMicroappContainer<T>` | `MicroApp` (not generic over a router type) |
| `microappName` | `id` |
| `guardedRoutes` / `unguardedRoutes` (`List<T>` of the router's own type) | `routes`: a tree of router-neutral `MicroAppRoute` / `MicroAppRouteGroup` nodes, each with `requiresAuthentication` (default `true`). Core builds the real router from them ([routing.md](routing.md)). |
| `commonStoredKeys` (with duplicate checks) | `storedKeys`. Each micro app declares the keys it owns, duplicates stop startup, and each micro app's storage is limited to its own keys. This keeps micro apps from overwriting each other's values. |
| `brokers` (`Set<BrokerInterface>`) | `dataEndpoints` and `widgetEndpoints` (provider classes; see [inter-module-communication.md](inter-module-communication.md)) |
| `localizationsDelegates` | `localizationsDelegates` |
| `bootstrap()` | `initialize(MicroAppContext)` |
| `onTenantLifecycleChanged(resumed/disposed)` | `dispose()`. There are no tenants, so the only lifecycle events are start and teardown. Session changes are observed through `SessionContext.changes`. |
| `registerDependencies` (global GetIt) | Not adopted (no globals). A micro app builds its internal dependencies in `initialize`, from the context, and takes overrides through its own constructor. |
| `remoteConfig` (`RemoteConfigController`) | `MicroAppContext.featureFlags` with `FeatureFlags.changes` |
| `AppShellContainer` and its `bootstrap` | `ModularAppContainer` (below): it runs steps 1–5 and 7 of the host sequence, with no route type parameter and no tenants, and hands the routes and localization delegates over as each micro app declared them. |
| Tenants, `TenantContainer`, multitenant broker | Not adopted (tenants are out of scope) |
| `buidActionParsers` | Not adopted (product-specific) |

## Declarations

```dart
/// The contracts given to ONE micro app. Core's initialization builds one per
/// micro app. Most fields are shared instances. keyValueStore and
/// secureKeyValueStore are limited to that micro app's storedKeys (FR-052).
/// All fields are required, so a host that forgets a contract fails to compile.
@immutable
final class MicroAppContext {
  const MicroAppContext({
    required InterModuleCommunication communication,
    required ApiClient apiClient,
    required Analytics analytics,
    required CrashReporter crashReporter,
    required AppLogger logger,
    required FeatureFlags featureFlags,
    required KeyValueStore keyValueStore,
    required SecureKeyValueStore secureKeyValueStore,
    required AppNavigator navigator,
    required SessionContext session,
    required DevicePermissions devicePermissions,
    required AppInfo appInfo,
  });
  // One final field per parameter. Identity equality (it is a bundle of services, not a value).
}

/// Implemented by each micro app. Hosts depend only on this interface.
abstract interface class MicroApp {
  /// Unique within a host. Matches ^[a-z][a-z0-9_]*$. Used as the prefix for
  /// every identifier the micro app owns. Read before initialize.
  String get id;

  /// Every storage key this micro app uses, in general or secure storage.
  /// Read before initialize, so Core can build storage limited to these keys.
  /// A key may be declared by only one micro app in the host (FR-052).
  Set<String> get storedKeys;

  /// Called exactly once per host, before any declaration below is read.
  /// The micro app builds its internal dependencies from the context and keeps
  /// the context to make the contracts available to its own UI. It must not
  /// rely on other modules' endpoints here, because providers are registered
  /// only after every micro app has initialized.
  Future<void> initialize(MicroAppContext context);

  /// The route tree (see routing.md). Read by the host after initialize.
  List<MicroAppRouteNode> get routes;

  List<LocalizationsDelegate<Object?>> get localizationsDelegates;

  /// The endpoints this micro app offers to other modules. Read by the host
  /// after initialize and registered through registerProviders (FR-047).
  List<DataEndpointProvider> get dataEndpoints;
  List<WidgetEndpointProvider> get widgetEndpoints;

  /// Called once when the host is torn down, in reverse initialization order.
  /// Releases subscriptions (for example to SessionContext.changes) and other
  /// resources acquired in initialize (FR-049).
  Future<void> dispose();
}
```

## Host sequence

The App Shell and the standalone examples run the same sequence. Foundations provides `ModularAppContainer`, the counterpart of `AppShellContainer`, which runs steps 1–5 and 7. The host supplies the contexts (step 2, through a `MicroAppContextFactory`) and builds the router (step 6).

```text
1. Read id and storedKeys from every micro app.
   - A duplicate micro app id, or a key declared by two micro apps, is reported, and startup stops.
     The error names the key and both micro apps.
2. Core initialization (environment-specific) builds one MicroAppContext per micro app.
   - Shared services, plus storage limited to that micro app's storedKeys.
3. For each micro app, in a fixed order:   await microApp.initialize(itsContext)
4. Collect from every micro app: routes, localizationsDelegates, dataEndpoints, widgetEndpoints.
   - Full paths are checked (routing.md). A failure stops startup.
5. communication.registerProviders(data: all dataEndpoints, widgets: all widgetEndpoints)
   - A DuplicateEndpointException names the address, and startup stops.
   - From here on, the set of endpoints is fixed.
6. Core builds the router from the route trees, with sign-in guards based on SessionContext,
   and the localizations from the delegates.
   Wrap them in the root StateScope(adapter: <the adapter Core's configuration names>).
   This is the single line that chooses the state-management library for every micro app.
   Run.
7. On teardown: await microApp.dispose() for each micro app, in reverse order.
   A failure is reported, and the remaining teardowns still run.
```

A standalone example runs steps 1–7 with its own micro app plus any modules it chooses to compose (for example Authentication). Instead of composing a module, it can declare stub providers for the endpoints that module would provide. The stubs are registered in step 5 like any others.

## ModularAppContainer

```dart
typedef MicroAppContextFactory = MicroAppContext Function(MicroApp microApp);

final class ModularAppContainer {
  ModularAppContainer({required List<MicroApp> microApps}); // initialization order
  final List<MicroApp> microApps;
  bool get isInitialized;

  // Available once bootstrap completes; a StateError before. Unmodifiable,
  // in micro-app order; maps are keyed by micro app id.
  List<MicroAppRouteNode> get routes;                     // every micro app's top-level nodes
  Map<String, List<MicroAppRouteNode>> get routesByMicroApp;
  bool? requiresAuthentication(String fullPath);           // own flag or an ancestor's; null if undeclared
  String? microAppIdOf(String fullPath);
  List<LocalizationsDelegate<Object?>> get localizationsDelegates;
  Map<String, List<LocalizationsDelegate<Object?>>> get localizationsDelegatesByMicroApp;
  Map<String, MicroAppContext> get contexts;
  Map<String, Set<String>> get storedKeysByMicroApp;
  Map<String, List<InterModuleEndpoint>> get endpointsByMicroApp;

  Future<void> bootstrap({
    required InterModuleCommunication communication,
    required MicroAppContextFactory contextFor,
    Set<String> hostPaths = const {},
  });
  Future<List<(String microAppId, Object error)>> dispose();
}
```

- It never flattens, merges, or rewrites a route tree. `routes` is each micro app's own top-level nodes, one list after another.
- `bootstrap` throws a `ModularAppStartupException` for a wiring mistake, with a `type`:
  - `duplicateMicroAppId`, `duplicateStoredKey`: thrown before any micro app initializes;
  - `invalidRoutePath`, `duplicateRoutePath`, `repeatedPathParameter`, `reservedRoutePath` (a micro app declares one of `hostPaths`): thrown after initialization, before providers are registered;
  - `alreadyBootstrapped`: a container starts once.
- A failing `initialize` is logged and reported to that micro app's crash reporter, then rethrown. A `DuplicateEndpointException` propagates. After any failure, `dispose` still releases the micro apps that did initialize.
- `dispose` runs in reverse order. Each failure is logged and reported with the micro app id only, and the rest still dispose. A second call does nothing.
- Each micro app's initialization time is logged at debug level, as `microAppId` and `durationMs`.

## Guarantees and responsibilities

| Concern | Owner |
|---------|-------|
| Building one `MicroAppContext` per micro app, with storage limited to its `storedKeys`, and choosing implementations for each environment | Core (initialization) and the host (configuration) |
| Running the host sequence: duplicate checks, initialize, collect, check routes, register providers, dispose | `ModularAppContainer` (Foundations), called by the host |
| Building the router and guarding routes, from `ModularAppContainer.routes` and `ModularAppContainer.requiresAuthentication` | Core, or the host |
| Building the router from the route contract | Core ([routing.md](routing.md)) |
| Declaring `storedKeys`, routes, localizations, and endpoint providers | Micro app |
| Building internal dependencies in `initialize`, and accepting overrides through its own constructor (for example a fake repository in a standalone example) | Micro app |
| Releasing resources in `dispose` | Micro app |
| Passing the contracts to its own widgets (constructors or a scope it owns) | Micro app |
| Choosing the state-management adapter and placing the root `StateScope` | Host (named in Core's initialization configuration) |
| Declaring its state with the re-exported `switchable_state` contract (`AppState`, `StateScope.group` inside a `MicroAppRouteGroup`, `StateStoreOwner`) | Micro app |
| Not calling `register`/`registerWidget` directly, and not importing Core, other micro apps, a router package, the `design_system` package, `switchable_state`, any adapter, or any state library at runtime | Micro app (enforced by Core and host review, and by `pubspec` rules in each micro app's spec). Adapters are allowed as development-only dependencies for widget tests. |
