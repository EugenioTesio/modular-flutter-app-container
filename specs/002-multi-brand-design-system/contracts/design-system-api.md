# Contract: `design_system` 0.2.0 Public Surface

This is the public library `package:design_system/design_system.dart`. Foundations re-exports it unchanged (FR-005). Every public name starts with `Ds`.

The signatures below are contracts. Documentation in the house style (from CLAUDE.md) is required for each one.

## Brand

```dart
@immutable
final class DsBrand {
  const DsBrand({required this.id, required this.name, required this.colors, required this.typography,
      required this.spacing, required this.radius, required this.borderWidths, required this.opacities,
      required this.iconSizes});

  static const DsBrand whiteLabel = DsBrand(/* hard-coded values, data-model.md */);

  @Deprecated('Use DsBrand.whiteLabel. Removed in 0.3.0.')
  static const DsBrand standard = whiteLabel;

  final String id;
  final String name;
  final DsBrandColors colors;          // brand / neutral / success / warning / error / info scales
  final DsBrandTypography typography;  // seven DsTypeStyle
  final DsSpacing spacing;
  final DsRadius radius;
  final DsBorderWidths borderWidths;
  final DsOpacities opacities;
  final DsIconSizes iconSizes;
  // value equality, hashCode, toString (id only: never dumps values)
}
// Value classes: DsBrandColors, DsBrandScale, DsNeutralScale, DsStatusScale, DsBrandTypography, DsTypeStyle,
// DsSpacing, DsRadius, DsBorderWidths, DsOpacities, DsIconSizes. All const-constructible with value equality.
```

## Brand files

```dart
abstract final class DsBrandFile {
  static const Set<int> supportedSchemaVersions = {1};
  static DsBrandParseResult parseJson(String json);
  static DsBrandParseResult parseMap(Map<String, Object?> map);
}

sealed class DsBrandParseResult { const DsBrandParseResult(); }
final class DsBrandParseSuccess extends DsBrandParseResult { const DsBrandParseSuccess(this.brand); final DsBrand brand; }
final class DsBrandParseFailure extends DsBrandParseResult { const DsBrandParseFailure(this.exception); final DsBrandFileException exception; }

// lib/src/exceptions/ds_brand_file_exception.dart
final class DsBrandFileException implements Exception {
  const DsBrandFileException(this.problems);   // never empty
  final List<DsBrandFileProblem> problems;
}

@immutable
final class DsBrandFileProblem {
  const DsBrandFileProblem({required this.path, required this.type, required this.message});
  final String path;               // DTCG dot path; '' for whole-file problems
  final DsBrandFileProblemType type;
  final String message;
}

enum DsBrandFileProblemType {
  malformedFile, missingToken, wrongType, invalidValue,
  unresolvedReference, circularReference, unsupportedFormat, unsupportedSchemaVersion,
}
```

## Placing a brand

```dart
abstract final class DsTheme {
  /// The top-level placement: pass both to MaterialApp.theme / darkTheme.
  static ThemeData light({DsBrand brand = DsBrand.whiteLabel});
  static ThemeData dark({DsBrand brand = DsBrand.whiteLabel});
  static ThemeData forBrightness(DsBrand brand, Brightness brightness);
}

/// Nested placement: a micro app wraps one screen or a group of screens.
final class DsBrandScope extends StatelessWidget {
  const DsBrandScope({super.key, required this.brand, required this.child});
  final DsBrand brand;
  final Widget child;
  // build: Theme(data: DsTheme.forBrightness(brand, Theme.of(context).brightness), child: child)
}
```

Usage from the App Shell:

```dart
MaterialApp.router(
  theme: DsTheme.light(brand: brand),
  darkTheme: DsTheme.dark(brand: brand),
  themeMode: themeMode,
  themeAnimationStyle: AnimationStyle.noAnimation, // brand and mode switches are instant
  routerConfig: router,
);
```

Usage from a micro app:

```dart
DsBrandScope(brand: partnerBrand, child: const PartnerOffersScreen())
```

**Behavior** (verified by `test/theme/`):

| # | Rule |
|---|------|
| B1 | Components resolve the nearest `DsTokens`, whether it was placed by `DsTheme` or by `DsBrandScope`. |
| B2 | Nested scopes: the innermost wins. |
| B3 | `showDialog`, `showModalBottomSheet`, and `showMenu` opened inside a scope use that scope's brand, because `Theme` is captured. |
| B4 | A scope follows the enclosing `Theme`'s brightness. |
| B5 | Changing the brand passed to `MaterialApp`, or the brand of a `DsBrandScope`, keeps the `Navigator` and the widget state, and shows the new brand on the next frame. The switch is instant: the App Shell passes `themeAnimationStyle: AnimationStyle.noAnimation`, so `MaterialApp` uses a plain `Theme`, and `DsBrandScope` never animates. `DsTokens.lerp` doesn't interpolate, so components never show values between two brands, even under an `AnimatedTheme`. If `lerp` is called with 0 < t < 1, a theme animation is running, and debug builds print one warning per run that switches aren't instant. |
| B6 | Material widgets inside a scope use that brand's colors and text styles, because a full `ThemeData` is built (FR-033). |
| B7 | With no `DsTokens` above a component, the white label is used, and debug builds print one warning per run. |
| B8 | In debug builds, placing a brand that fails the contrast gate prints one warning per brand, naming the failing pairs. Brands are told apart by value, so two different brands that share an id each warn. |

## Reading tokens

Components read every token through these getters from the nearest brand. Colors and text styles come only from semantic roles (FR-022).

```dart
@immutable
final class DsTokens extends ThemeExtension<DsTokens> {
  const DsTokens({required this.brand, required this.brightness, required this.colors, required this.typography,
      required this.spacing, required this.radius, required this.borderWidths, required this.opacities,
      required this.iconSizes});
  factory DsTokens.resolve(DsBrand brand, Brightness brightness);   // applies the semantic mapping
  // fields as named; copyWith; value equality
  // lerp never interpolates: it returns this for t < 0.5 and other from t = 0.5,
  // and in debug warns once when called with 0 < t < 1 (B5)
}

extension DsThemeContext on BuildContext {
  DsTokens get dsTokens;          // nearest, or white label for the ambient brightness (B7)
  DsBrand get dsBrand;
  DsColors get dsColors;
  DsTypography get dsTypography;
  DsSpacing get dsSpacing;
  DsRadius get dsRadius;
  DsBorderWidths get dsBorderWidths;
  DsOpacities get dsOpacities;
  DsIconSizes get dsIconSizes;
}

@immutable final class DsColors { /* 20 roles, as 0.1.0; no longer a ThemeExtension itself */ Color role(DsColorRole role); }
enum DsColorRole { background, surface, surfaceVariant, border, textPrimary, textSecondary, textDisabled, textOnAction,
  actionPrimary, actionPrimaryPressed, actionDisabled, focus, infoBackground, infoForeground, successBackground,
  successForeground, warningBackground, warningForeground, errorBackground, errorForeground }
@immutable final class DsTypography { /* display, headline, title, body, bodyStrong, label, caption */ }

enum DsSpacingStep { xxs, xs, sm, md, lg, xl, xxl }
@immutable final class DsSpacing {
  const DsSpacing({required this.xxs, required this.xs, required this.sm, required this.md,
      required this.lg, required this.xl, required this.xxl});
  double operator [](DsSpacingStep step);
  EdgeInsets all(DsSpacingStep step);
  EdgeInsets symmetric({DsSpacingStep? horizontal, DsSpacingStep? vertical});
  EdgeInsets only({DsSpacingStep? left, DsSpacingStep? top, DsSpacingStep? right, DsSpacingStep? bottom});
}
final class DsGap extends StatelessWidget { const DsGap(this.step, {super.key}); final DsSpacingStep step; }
```

## Contrast gate

```dart
abstract final class DsContrastGate {
  static DsContrastReport check(DsBrand brand);   // both brightnesses; pairs in data-model.md
}
@immutable final class DsContrastReport {
  final String brandId; final List<DsContrastViolation> violations; bool get passed;
}
@immutable final class DsContrastViolation {
  final Brightness brightness; final DsColorRole foreground; final DsColorRole background;
  final double ratio; final double requiredRatio;
}
```

## Catalog, QA, and guidelines

```dart
enum DsComponentLevel { atom, molecule, organism, template }
enum DsComponentStatus { experimental, stable, deprecated }
@immutable final class DsComponentInfo {
  const DsComponentInfo({required this.name, required this.level, required this.status, required this.since, this.removedIn});
}
abstract final class DsComponentCatalog { static const List<DsComponentInfo> all = [/* every component's info */]; }

abstract final class DsQaIdentifiers { static const bool enabled = bool.fromEnvironment('DS_QA_IDENTIFIERS'); }

final class DsGuidelines extends InheritedWidget {
  const DsGuidelines({super.key, required this.enabled, required super.child});
  static const bool buildEnabled = !kReleaseMode && bool.fromEnvironment('DS_GUIDELINES');
  static bool isEnabledOf(BuildContext context);   // always false in release builds
  final bool enabled;
}
```

## Components (0.2.0 changes)

All eight components keep their 0.1.0 parameters and behaviors (rules D1 to D9 of the Foundations contract) with these changes:

| Change | Applies to |
|--------|------------|
| Read every token from `context.ds*`. No literal visual values, and tap targets use `kMinInteractiveDimension`. | all |
| New `static const DsComponentInfo info`, and a `**Status**: stable.` documentation line | all |
| New optional `String? qaIdentifier` | all |
| Output wrapped in the internal `DsGuidelinesFrame(level: info.level)` | all |
| `padding` becomes `EdgeInsetsGeometry?`. Null means `context.dsSpacing.all(DsSpacingStep.md)`. | `DsCard` |
| `size` becomes `double?`. Null means `context.dsIconSizes.md`. | `DsLoadingIndicator` |
| Moved to `lib/src/components/<level>/`. The public names are unchanged. | all |

D8 now reads: changing the brand scale changes every component that uses `actionPrimary` or `focus`.

## Removed in 0.2.0 (listed in the Migration section)

| 0.1.0 element | Replacement |
|---------------|-------------|
| `DsBrand(primary:, fontFamily:)` | A brand file through `DsBrandFile`, or a full `DsBrand(...)` |
| `DsPalette` | `context.dsBrand.colors` (primitives). Components use `context.dsColors`. |
| `DsSpacing.md` (static) | `context.dsSpacing.md`, or `DsGap(DsSpacingStep.md)` |
| `DsRadius.md` (static) | `context.dsRadius.md` |
| `DsColors` and `DsTypography` as `ThemeExtension`s (`Theme.of(c).extension<DsColors>()`) | `context.dsColors`, `context.dsTypography` |
