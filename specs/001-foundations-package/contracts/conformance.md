# Contract: Inter-Module Communication Conformance Check

This contract covers FR-017 and SC-003. It is published from `package:foundations/testing.dart`. It lets any implementation of `InterModuleCommunication` prove that it behaves as [inter-module-communication.md](inter-module-communication.md) requires. That includes Core's implementation and stubs in standalone examples. The check does not depend on any test framework ([research R9](../research.md#r9-conformance-check-packaging)).

## Declarations

```dart
@immutable
final class ConformanceResult {
  const ConformanceResult.pass();
  const ConformanceResult.fail(String message);
  final bool passed;
  final String? message;
}

@immutable
final class ConformanceCase {
  final String id;            // 'C01' …
  final String description;
  /// Calls create() to get a fresh, empty implementation, then exercises it.
  Future<ConformanceResult> run(InterModuleCommunication Function() create);
}

abstract final class InterModuleConformance {
  static const List<ConformanceCase> cases = [/* C01 … C17 */];
}
```

## Cases

| Id | Scenario | Behavior row | Expectation |
|----|----------|--------------|-------------|
| C01 | Data request succeeds | B7 | 200, and data equals what the handler returned |
| C02 | A required parameter is absent | B2 | 400, `missing_required_parameters`, data is null, the handler is not called |
| C03 | A required parameter is null | B3 | Same as C02 |
| C04 | A required parameter is an empty string | B4 | The handler is called with `''`, and its 400 `invalid_id` reaches the caller unchanged |
| C05 | The consumer's `requiredParameters` are ignored | B5 | The consumer declares `{}` and omits the provider's required key: 400. The consumer declares an extra key that the provider doesn't require: the handler is still called |
| C06 | Extra parameters pass through | B6 | The handler receives the undeclared keys unchanged |
| C07 | Data endpoint not registered | B1 | 404, `endpoint_not_registered`, data is null |
| C08 | A provider-defined 404 passes through | B7 | 404 with the provider's own errorCode, such as `details_not_found` |
| C09 | The handler throws synchronously | B8 | 500, data is null, and errorCode does not contain the exception text |
| C10 | The handler's future completes with an error | B8 | Same as C09 |
| C11 | Duplicate data registration | B10 | The second `register` throws `DuplicateEndpointException(type: data)` |
| C12 | UI request succeeds synchronously | B12 | 200, `data` is the exact widget instance, returned without awaiting |
| C13 | UI request with a required parameter absent or null | B2, B3 | 400, `missing_required_parameters`, the factory is not called |
| C14 | UI endpoint not registered | B1 | 404, `endpoint_not_registered` |
| C15 | The UI factory throws | B9 | 500, data is null |
| C16 | Duplicate UI registration, and separate registries | B10, B11 | A second `registerWidget` throws `DuplicateEndpointException(type: widget)`. Registering the same URL as data and as UI does not throw, and both work |
| C17 | Registration check | B13 | `isRegistered` is `false` before registration. After a data registration it is `true` for `data` and still `false` for `widget`. It never invokes the provider |

## Usage (shown as documentation, not as an implementation)

```dart
// In Core's test suite, or in a standalone example's test suite:
for (final c in InterModuleConformance.cases) {
  test('${c.id} ${c.description}', () async {
    final result = await c.run(CoreInterModuleCommunication.new);
    expect(result.passed, isTrue, reason: result.message);
  });
}
```

## Self-verification inside Foundations

Foundations' own tests run every case against:

1. A correct reference implementation in `test/support/`. Every case must pass.
2. One deliberately broken variant per behavior row: one ignores required parameters, one rethrows handler errors, one shares a single registry between kinds, one leaks exception text, one accepts duplicates. Each variant must fail at least the case that targets its defect.

This proves the cases can detect the defect each one targets.
