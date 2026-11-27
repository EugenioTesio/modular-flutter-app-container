# Contract: `design_system_lints` (analyzer plugin)

Covers FR-049 to FR-051 and User Story 6 (scenarios 5–7). The rationale is in [research R11](../research.md#r11-token-lint-rules).

## Package

| Item | Value |
|------|-------|
| Location | `lints/` in the Design System repository |
| Name | `design_system_lints`, `publish_to: none` |
| Dependencies | `analysis_server_plugin: 0.3.26`, `analyzer: 14.7.0` (exact pins, which move together with the Dart SDK) |
| Dev dependencies | `analyzer_testing: 0.4.5`, `test_reflective_loader: 0.6.0`, `test` |
| Entry point | `lib/main.dart`, with a top-level `final plugin = DsLintsPlugin();`. `register` calls `registry.registerWarningRule` for each rule. |
| Runtime impact | None. It is loaded only by the analysis server (FR-051). |

## Rules

All four are **warning rules**: they are on as soon as the plugin is enabled, and each can be switched off under `diagnostics:`. Each rule's `LintCode` is a `static const`, so `// ignore: design_system_lints/<rule>` works. Each rule reports only when `context.isInLibDir`, so `test/` is never reported.

| Rule | Scope | Reports | Correction message |
|------|-------|---------|--------------------|
| `ds_avoid_literal_color` | any `lib/` file | `Color(...)`, `Color.fromARGB(...)`, `Color.fromRGBO(...)`, `Color.from(...)`, and any `Colors.<x>` or `CupertinoColors.<x>` except `Colors.transparent` | "Use a semantic color: `context.dsColors.<role>`." |
| `ds_avoid_literal_spacing` | any `lib/` file | A non-zero number literal passed to `EdgeInsets.all/symmetric/only/fromLTRB`, to `EdgeInsetsDirectional.*`, to `SizedBox(width:/height:)`, to `SizedBox.square(dimension:)`, to `Row`/`Column`/`Wrap` `spacing:`/`runSpacing:`, to `BorderRadius.circular`, or to `Radius.circular` | "Use `context.dsSpacing.<step>`, `DsGap(DsSpacingStep.<step>)`, or `context.dsRadius.<step>`." |
| `ds_avoid_literal_component_value` | `package:design_system/src/components/**` only | A number literal in `Icon(size:)`, `BorderSide(width:)`, `.withValues(alpha:)`, or `.withOpacity(...)` | "Use `context.dsIconSizes`, `context.dsBorderWidths`, or `context.dsOpacities`." |
| `ds_avoid_primitive_color` | `package:design_system/src/components/**` only | Any read through `DsBrand.colors`, which means the primitive scales | "Components use semantic roles: `context.dsColors.<role>`." |

- **Tap targets**: the 48 × 48 minimum is never reported. Components use the SDK constant `kMinInteractiveDimension` instead of a literal.
- **Debug colors**: the guidelines-overlay colors are the only allowed literal colors in the package. They live in one file with `// ignore_for_file: design_system_lints/ds_avoid_literal_color` and a comment explaining why.

## Opting in

**In the Design System's root `analysis_options.yaml`** (FR-050: 0 reports):

```yaml
plugins:
  design_system_lints:
    path: lints
```

**In consumer projects** (the App Shell, micro apps, Foundations' example), in `analysis_options.yaml`:

```yaml
plugins:
  design_system_lints:
    git:
      url: https://github.com/EugenioTesio/modular-flutter-app-design-system.git
      path: lints
      ref: v0.2.0
    # diagnostics:
    #   ds_avoid_literal_spacing: false   # per-rule opt-out
```

If quickstart V8 shows that `git:` isn't accepted for plugins, the documented fallback inside the container checkout is `path: ../design-system/lints`. The lint is dev-only, so it never changes which `design_system` version a consumer resolves (FR-051).

## Rule tests (`lints/test/`, `analyzer_testing`)

One `AnalysisRuleTest` per rule. Each one contains:
- one positive case for each reported form;
- negative cases: `Colors.transparent`, `0`, `kMinInteractiveDimension`, token reads, and the same code under `test/`;
- for the two component-scoped rules, a file outside `src/components/` that is not reported.

`tool/check.sh` runs `dart test` in `lints/` and `flutter analyze` at the root. The analyze run must report 0 plugin diagnostics.
