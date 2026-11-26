# Contract: Interim Design System (`design_system`)

This contract covers FR-002, FR-036–FR-038, and FR-054–FR-056. It is the public surface of the interim Design System package. The package lives at `foundations/packages/design_system/` until it moves to its own repository ([research R3](../research.md#r3-design-system-dependency-and-re-export), [R21](../research.md#r21-interim-design-system-scope)). Foundations re-exports `package:design_system/design_system.dart` unchanged. Micro apps never import `design_system` directly.

Every public name starts with `Ds`. This avoids clashes with Material widgets (`Card`, `ListTile`), with Foundations, and with `switchable_state`.

## Package rules

| Rule | Check |
|------|-------|
| Package name `design_system`, single public library `lib/design_system.dart` | These stay the same when the package moves (FR-056) |
| Runtime dependencies: `flutter` only. No `google_fonts`, no `intl`, and no reference to `foundations` | A dependency allow-list test inside the package |
| `publish_to: none`, its own `version` (starting at `0.1.0`), and its own `CHANGELOG.md` | Foundations' changelog states which version it delivers (FR-038) |
| Components use **semantic** tokens only, never primitive palette values | Code review, plus the brand-change test (SC-017) |

## Tokens

```dart
/// Primitive tokens: fixed base values. Components never read these directly.
abstract final class DsPalette {
  // neutral0, neutral50 … neutral900; success/warning/error scales (100, 500, 700)
  static const Color neutral0 = /* … */;
  // …
}

abstract final class DsSpacing {
  static const double xxs = 4, xs = 8, sm = 12, md = 16, lg = 24, xl = 32, xxl = 48;
}

abstract final class DsRadius {
  static const double sm = 4, md = 8, lg = 16, pill = 999;
}

/// Semantic color roles, resolved for the active theme and brand.
@immutable
final class DsColors extends ThemeExtension<DsColors> {
  const DsColors({
    required Color background, required Color surface, required Color surfaceVariant,
    required Color border,
    required Color textPrimary, required Color textSecondary, required Color textDisabled,
    required Color textOnAction,
    required Color actionPrimary, required Color actionPrimaryPressed, required Color actionDisabled,
    required Color focus,
    required Color infoBackground, required Color infoForeground,
    required Color successBackground, required Color successForeground,
    required Color warningBackground, required Color warningForeground,
    required Color errorBackground, required Color errorForeground,
  });
  // fields as above; copyWith; lerp
}

@immutable
final class DsTypography extends ThemeExtension<DsTypography> {
  const DsTypography({
    required TextStyle display, required TextStyle headline, required TextStyle title,
    required TextStyle body, required TextStyle bodyStrong, required TextStyle label,
    required TextStyle caption,
  });
  // fields as above; copyWith; lerp
}
```

## Brand and themes

```dart
/// The single brand configuration (FR-054). Both themes derive from it.
@immutable
final class DsBrand {
  const DsBrand({required Color primary, String? fontFamily});
  static const DsBrand standard = DsBrand(primary: /* default brand color */);
  final Color primary;
  final String? fontFamily;   // supplied by the host app's assets; falls back to the platform font
}

abstract final class DsTheme {
  /// ThemeData with DsColors and DsTypography installed as ThemeExtensions,
  /// and Material component themes aligned to the same roles.
  static ThemeData light({DsBrand brand = DsBrand.standard});
  static ThemeData dark({DsBrand brand = DsBrand.standard});
}

extension DsThemeContext on BuildContext {
  DsColors get dsColors;          // Theme.of(this).extension<DsColors>()!
  DsTypography get dsTypography;
}
```

Contrast rule (FR-055): in both themes, every text role on its intended background reaches at least 4.5:1, and `actionPrimary` against `background` (interface contrast) reaches at least 3:1. The themes derive `textOnAction` from the brand's `primary`, choosing light or dark text so it meets 4.5:1.

## Components

```dart
// Atoms
enum DsButtonVariant { primary, secondary, tertiary }

final class DsButton extends StatelessWidget {
  const DsButton({
    super.key,
    required String label,                 // also the accessible label
    required VoidCallback? onPressed,      // null means disabled
    DsButtonVariant variant = DsButtonVariant.primary,
    IconData? icon,
    bool isLoading = false,                // shows a progress indicator, ignores taps, keeps its size
    bool expand = false,                   // full available width
  });
}

final class DsTextField extends StatelessWidget {
  const DsTextField({
    super.key,
    required String label,
    TextEditingController? controller,
    String? hint,
    String? helperText,
    String? errorText,                     // non-null shows the error state and is announced
    bool obscureText = false,
    bool enabled = true,
    TextInputType? keyboardType,
    TextInputAction? textInputAction,
    ValueChanged<String>? onChanged,
    ValueChanged<String>? onSubmitted,
  });
}

final class DsLoadingIndicator extends StatelessWidget {
  const DsLoadingIndicator({super.key, required String semanticsLabel, double size = 24});
}

// Molecules
final class DsCard extends StatelessWidget {
  const DsCard({
    super.key,
    required Widget child,
    VoidCallback? onTap,                   // when non-null, the card is a single tappable element
    String? semanticsLabel,                // required in practice when onTap is set
    EdgeInsetsGeometry padding = const EdgeInsets.all(DsSpacing.md),
  });
}

final class DsListTile extends StatelessWidget {
  const DsListTile({
    super.key,
    required String title,
    String? subtitle,
    Widget? leading,
    Widget? trailing,
    VoidCallback? onTap,
  });
}

enum DsMessageTone { info, success, warning, error }

final class DsInlineMessage extends StatelessWidget {
  const DsInlineMessage({
    super.key,
    required String message,
    DsMessageTone tone = DsMessageTone.info,
    String? title,
    String? actionLabel,
    VoidCallback? onAction,
  });
}

final class DsEmptyState extends StatelessWidget {
  const DsEmptyState({
    super.key,
    required String title,
    String? message,
    IconData? icon,
    String? actionLabel,
    VoidCallback? onAction,
  });
}

// Templates
final class DsPageScaffold extends StatelessWidget {
  const DsPageScaffold({
    super.key,
    required String title,
    required Widget body,                  // padded with DsSpacing.md, inside SafeArea
    DsButton? primaryAction,               // pinned at the bottom, full width
    VoidCallback? onBack,                  // when null, the platform back behavior applies
  });
}
```

## Behavior every component must have (verified in the package's own tests)

| # | Rule | Applies to |
|---|------|------------|
| D1 | Tap targets are at least 48 × 48 (`androidTapTargetGuideline`, `iOSTapTargetGuideline`) | All interactive components |
| D2 | Every tappable element is labeled (`labeledTapTargetGuideline`) | All interactive components |
| D3 | Text contrast meets `textContrastGuideline` in the light and dark themes | All components |
| D4 | At `TextScaler.linear(2.0)`, there are no overflow errors on a 320-wide surface | All components |
| D5 | The disabled state ignores taps and uses the `actionDisabled` and `textDisabled` roles | `DsButton`, `DsTextField` |
| D6 | The loading state ignores taps, keeps the same size, and announces `semanticsLabel` | `DsButton(isLoading)`, `DsLoadingIndicator` |
| D7 | The error state shows `errorText` in `errorForeground` and exposes it to semantics | `DsTextField` |
| D8 | Changing `DsBrand.primary` changes the primary-action color of `DsButton(primary)`, the focus ring, and any other component using `actionPrimary`, with no component change | Themes, `DsButton` |
| D9 | Each `DsMessageTone` maps to its matching background and foreground roles | `DsInlineMessage` |
