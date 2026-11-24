# Contract: Inter-Module Communication

This contract covers FR-007 through FR-018, FR-046, FR-047, and the call-site part of FR-050. The names follow the reference article, with one difference: there is no static `instance` accessor (FR-016).

From the user's earlier `flutter_enterprise_architecture` package (where this contract is called the *broker*), it adopts three ideas: class-based endpoint providers (the counterpart of `BrokerInterface`), bulk registration (`registerSet`), and the `isRegistered` check. It also adds an optional trace name on calls (the counterpart of `ICDestination.trace`).

## Declarations

```dart
/// Describes one call: where it goes, what it sends, and, at registration,
/// which parameters the provider requires.
@immutable
final class InterModuleEndpoint {
  const InterModuleEndpoint({
    required String url,                                  // '/<module>/v<N>/<name>'
    Map<String, Object?> parameters = const {},           // payload rule applies
    Set<String> requiredParameters = const {},            // read only at registration
    String? traceName,                                    // read only on calls (FR-050)
  });

  final String url;
  final Map<String, Object?> parameters;
  final Set<String> requiredParameters;
  /// When set on a request or getWidget call, the implementation times the call
  /// and records its name ('<microAppId>.<name>'), duration, and status in its
  /// own diagnostics. It is a hint, not a capability.
  final String? traceName;
  // == and hashCode compare url, parameters (mapEquals), requiredParameters (setEquals), and traceName.
}

/// The result of every inter-module call.
@immutable
final class InterModuleCommunicationResponse<T> {
  const InterModuleCommunicationResponse({
    required int statusCode,
    T? data,
    String? errorCode,
  });

  /// statusCode 200 with non-null data.
  const InterModuleCommunicationResponse.success(T data);

  /// A failure. statusCode must be outside 200–299 (checked by assert). data is null.
  const InterModuleCommunicationResponse.failure({
    required int statusCode,
    String? errorCode,
  });

  final int statusCode;
  final T? data;
  final String? errorCode;

  bool get isSuccess;                                     // 200 <= statusCode < 300
}

/// The four statuses. They are application statuses, not network responses.
abstract final class InterModuleStatusCode {
  static const int ok = 200;
  static const int badRequest = 400;
  static const int notFound = 404;
  static const int internalError = 500;
}

/// The shared error codes (FR-012).
abstract final class InterModuleErrorCode {
  static const String missingRequiredParameters = 'missing_required_parameters'; // 400, Core
  static const String endpointNotRegistered = 'endpoint_not_registered';         // 404, Core
  static const String unauthenticated = 'unauthenticated';                       // 400, provider
  static const String forbidden = 'forbidden';                                   // 400, provider
}

/// Checks values against the payload rule. Static methods only.
abstract final class InterModuleValidator {
  /// True if [value] follows the payload rule: null, bool, num, String,
  /// a List of conforming values, or a Map<String, …> of conforming values.
  static bool isValidPayload(Object? value);
}

typedef InterModuleHandler =
    Future<InterModuleCommunicationResponse<Map<String, Object?>>> Function(
      Map<String, Object?> parameters,
    );

typedef InterModuleWidgetFactory =
    InterModuleCommunicationResponse<Widget> Function(
      Map<String, Object?> parameters,
    );

enum EndpointType { data, widget }

/// Thrown when a URL is registered twice for the same [type].
final class DuplicateEndpointException implements Exception {
  const DuplicateEndpointException({required String url, required EndpointType type});
  final String url;
  final EndpointType type;
}

/// Implemented by Core. Hosts and micro apps receive it through MicroAppContext.
abstract interface class InterModuleCommunication {
  /// Registers a data endpoint. Called by hosts (normally through
  /// registerProviders) after every micro app has initialized. Micro apps
  /// never call it directly (FR-047).
  /// Throws DuplicateEndpointException if the URL is already registered as a data endpoint.
  void register({
    required InterModuleEndpoint endpoint,
    required InterModuleHandler handler,
  });

  Future<InterModuleCommunicationResponse<Map<String, Object?>>> request({
    required InterModuleEndpoint endpoint,
  });

  /// Registers a UI endpoint. Called by hosts (normally through
  /// registerProviders) after every micro app has initialized.
  /// Throws DuplicateEndpointException if the URL is already registered as a UI endpoint.
  void registerWidget({
    required InterModuleEndpoint endpoint,
    required InterModuleWidgetFactory factory,
  });

  /// Synchronous. Returns the factory's element without waiting for its data.
  InterModuleCommunicationResponse<Widget> getWidget({
    required InterModuleEndpoint endpoint,
  });

  /// True if url is registered for type. Never invokes a provider.
  bool isRegistered({required String url, required EndpointType type});
}

/// A micro app's offer of one data endpoint. It is a class, so it can be
/// unit tested directly, without Core (FR-046).
abstract interface class DataEndpointProvider {
  /// The address and required parameters. parameters is unused here.
  InterModuleEndpoint get endpoint;

  /// Answers a request. Called only after the required-parameter check passes
  /// when it is reached through InterModuleCommunication.
  Future<InterModuleCommunicationResponse<Map<String, Object?>>> handle(
    Map<String, Object?> parameters,
  );
}

/// A micro app's offer of one UI endpoint.
abstract interface class WidgetEndpointProvider {
  InterModuleEndpoint get endpoint;

  /// Synchronous. Returns the element without waiting for its data.
  InterModuleCommunicationResponse<Widget> build(Map<String, Object?> parameters);
}

/// Bulk registration used by hosts (FR-047). The counterpart of the earlier
/// broker's registerSet.
extension InterModuleProviderRegistration on InterModuleCommunication {
  /// Registers every provider: data providers through register(endpoint, handler: p.handle),
  /// and UI providers through registerWidget(endpoint, factory: p.build).
  /// Stops at the first DuplicateEndpointException and lets it propagate.
  void registerProviders({
    Iterable<DataEndpointProvider> data = const [],
    Iterable<WidgetEndpointProvider> widgets = const [],
  });
}
```

## Behavior every implementation must have

The conformance check in [conformance.md](conformance.md) verifies each row.

| # | Situation | `request` / `getWidget` outcome | Provider invoked? |
|---|-----------|---------------------------------|-------------------|
| B1 | URL not registered for this type | `404`, `endpoint_not_registered`, `data == null` | — |
| B2 | A provider-declared required parameter is absent | `400`, `missing_required_parameters`, `data == null` | No |
| B3 | A provider-declared required parameter is `null` | Same as B2 | No |
| B4 | A required parameter is `''` (empty string) | Whatever the provider returns | Yes |
| B5 | The consumer's `requiredParameters` differ from the provider's | Ignored. Only the provider's declaration counts | Depends on the provider's declaration |
| B6 | Extra, undeclared parameters | Passed to the provider unchanged | Yes |
| B7 | The provider returns any response | Returned to the consumer unchanged (status, data, errorCode) | Yes |
| B8 | The handler throws, or its future completes with an error | `500`, `data == null`, errorCode contains no exception detail | Yes |
| B9 | The widget factory throws | `500`, `data == null` | Yes |
| B10 | The same URL is registered twice for the same type | The second `register`/`registerWidget` throws `DuplicateEndpointException` | — |
| B11 | The same URL is registered once as data and once as UI | Both work independently | Yes |
| B12 | `getWidget` succeeds | Returns synchronously. `data` is the exact instance the factory created | Yes |
| B13 | `isRegistered(url, type)` | `true` only after that URL was registered for that type. A URL registered as data is `false` for `widget`, and the other way round | Never |
| B14 | A call whose endpoint has a `traceName` | Timed. The name, duration, and resulting status (including 404 and 500) are recorded in the implementation's own diagnostics. A call without a `traceName` is not traced | Depends on B1–B12 |

B14 is recorded inside the implementation, so the conformance check doesn't cover it. Core's own tests verify it.

## Conventions (documented, not enforced at runtime)

- **Versioning (FR-018)**: an incompatible change to parameters or results is published under a new URL (`/v2/…`). The old URL stays registered until every consumer has migrated.
- **Access refusals**: a provider returns `400` with `unauthenticated` or `forbidden`. No other statuses are added.
- **Payloads**: providers return only values for which `InterModuleValidator.isValidPayload` is true. Core may assert this in debug builds.
- **Errors after `getWidget`** (FR-015): failures while the returned element renders or loads belong to the providing module.
