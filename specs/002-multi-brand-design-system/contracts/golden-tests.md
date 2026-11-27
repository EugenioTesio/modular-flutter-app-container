# Contract: Golden Tests and Accessibility Harness

Covers FR-040 to FR-045, FR-026, User Story 5, and SC-009 and SC-016. The rationale is in [research R10](../research.md#r10-golden-tests-and-the-5-comparator).

## Comparator

`test/support/ds_golden_comparator.dart` is test-only and never exported.

```dart
/// Comparator that accepts golden images differing by at most [threshold] of their pixels.
final class DsGoldenComparator extends LocalFileComparator {
  DsGoldenComparator(super.testFile);

  /// The single place the tolerance is set: 5% of the pixels.
  static const double threshold = 0.05;

  @override
  Future<bool> compare(Uint8List imageBytes, Uri golden) async {
    final result = await GoldenFileComparator.compareLists(imageBytes, await getGoldenBytes(golden));
    if (result.passed || result.diffPercent <= threshold) {
      // print measured difference when > 0, e.g. "ds_button/primary__white_label__light.png: 1.84% (tolerated)"
      result.dispose();
      return true;
    }
    final error = await generateFailureOutput(result, golden, basedir);   // writes failures/*.png
    result.dispose();
    throw FlutterError('$error\nMeasured ${(result.diffPercent * 100).toStringAsFixed(2)}% > 5.00% allowed.');
  }
}
```

`test/flutter_test_config.dart` sets everything up once per test file:

```dart
Future<void> testExecutable(FutureOr<void> Function() testMain) async {
  goldenFileComparator = DsGoldenComparator(
      (goldenFileComparator as LocalFileComparator).basedir.resolve('placeholder_test.dart'));
  await DsTestFonts.load();   // Roboto (as 'Roboto' and 'Forest Sans') and MaterialIcons from test/fonts/
  await testMain();
}
```

## Golden matrix

| Dimension | Values |
|-----------|--------|
| Component | every entry of `DsComponentCatalog.all` |
| State | every state that the accessibility harness lists for the component (enabled, disabled, loading, error, each tone, …) |
| Brand | `white_label` and `forest`, read from `widgetbook/brands/` |
| Mode | `light`, `dark` |

- **Path**: `test/components/goldens/<component>/<state>__<brand>__<mode>.png`, for example `ds_button/loading__forest__dark.png`.
- **Rendering**:
  - Each image holds one component in one state. It is cropped to a `RepaintBoundary` around the component, with `context.dsSpacing.md` padding, on the mode's `background` (FR-043).
  - The surface size is fixed per component, and `devicePixelRatio` is 2.0 (FR-044).
  - Animations are settled before capture. Indeterminate progress indicators are captured at a fixed tick with `tester.pump(Duration(milliseconds: 500))`.
- **Tags**: golden tests carry the tag `golden`, which is declared in `dart_test.yaml`.

## Commands

| Purpose | Command |
|---------|---------|
| Run everything (goldens included) | `flutter test` |
| Run only goldens | `flutter test --tags golden` |
| Regenerate after an intentional change (FR-045) | `flutter test --update-goldens --tags golden` |
| Check goldens on Linux | `docker run --rm -v "$PWD":/w -w /w ghcr.io/cirruslabs/flutter:3.47.9 flutter test --tags golden` |

`failures/` folders are git-ignored (`**/failures/` in `.gitignore`).

## Accessibility harness

`test/support/ds_a11y_harness.dart` extends the 0.1.0 harness:

- **Coverage**: it runs every case for each brand × mode, where the brands are white label and forest (FR-026, SC-016).
- **Checks**:
  - `androidTapTargetGuideline`, `iOSTapTargetGuideline`, `labeledTapTargetGuideline`, and `textContrastGuideline`;
  - no overflow at `TextScaler.linear(2)` on a 320-wide surface.
- **Golden cases**: the same case list feeds the golden matrix, so every accessibility-checked state also has a golden image.

## Required tests (in Gherkin, per the house rules)

```text
// Feature: Golden comparison tolerates cross-platform rendering
Given two images differing in 3% of pixels   When compared, Then the comparison passes and reports 3.00%
Given two images differing in 6% of pixels   When compared, Then it fails, names the image, reports 6.00%, and writes failures/
Given a golden that does not exist           When compared, Then it fails and the message names --update-goldens
// Feature: A real visual change is caught
Given the forest brand with brand.600 changed  When the DsButton primary golden runs, Then it fails (SC-009)
```
