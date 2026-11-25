# Validation Report: Foundations 0.1.0

This report records the outcome of every check in [quickstart.md](quickstart.md), V1–V10.

- **Run on**: 2026-11-25, macOS, Flutter 3.47.9 (stable), Dart 3.13.7. Rerun after `ModularAppContainer` (FR-057) was added, and the reference host and the example host moved onto it.
- **Foundations commit**: `b31d34e` on `001-foundations-package`, plus the uncommitted `ModularAppContainer` change.
- **Result**: every automated check passes. Three manual checks are still open: the example demo, the gallery, and the timed onboarding walkthrough. They are marked **Pending (manual)** below, with what already covers them.

## Summary

| Check | Covers | Result |
|-------|--------|--------|
| T093 | Analyzer and API docs in every package | Pass |
| V1 | Package boundary | Pass |
| V2 | Static quality and documentation | Pass |
| V3 | Conformance check proves itself | Pass |
| V4 | Every inter-module outcome in the example | Pass; the manual demo is pending |
| V5 | Capabilities and micro app integration with fakes | Pass |
| V6 | Interim Design System through Foundations | Pass; the manual gallery check is pending |
| V7 | Coverage | Pass (100%) |
| V8 | Developer onboarding | Pending (manual); a stand-in run passes |
| V9 | State-management contract | Pass |
| V10 | The route contract builds a real router | Pass |

## T093 — Analyzer and API docs (SC-006)

| Command | Outcome |
|---------|---------|
| `flutter analyze` in `foundations/` | No issues found |
| `flutter analyze` in `foundations/packages/design_system/` | No issues found |
| `flutter analyze` in `foundations/example/` | No issues found |
| `flutter analyze` in `foundations/packages/design_system/example/` | No issues found |
| `dart doc --dry-run` in `foundations/` | Found 0 warnings and 0 errors |
| `dart doc --dry-run` in `foundations/packages/design_system/` | Found 0 warnings and 0 errors |

No fixes were needed.

## V1 — Package boundary (SC-001, SC-012)

- `flutter pub deps --no-dev --style=compact`: the direct runtime dependencies are exactly `design_system 0.1.0`, `flutter`, and `switchable_state 0.1.0`. The transitive ones are Flutter's own (`characters`, `collection`, `material_color_utilities`, `meta`, `sky_engine`, `vector_math`). There are no vendor SDKs, adapters, state libraries, router packages, Core, or micro apps.
- `flutter test test/architecture`: 7 tests pass. They cover the dependency allow-list, no global state in `lib/`, and no `ambiguous_export`.

**Result**: Pass.

## V2 — Static quality and documentation (SC-006)

The same commands as T093: `flutter analyze` reports no issues with `public_member_api_docs` on, and `dart doc --dry-run` reports 0 warnings.

**Result**: Pass.

## V3 — Conformance check proves itself (US1, US5, SC-003)

`flutter test test/testing`: 30 tests pass. Cases C01–C17 pass against the reference implementation, and each broken variant fails exactly the cases that target its defect.

**Result**: Pass.

## V4 — Every inter-module outcome in a runnable example (US1, US5, FR-035, SC-008)

`flutter test` in `foundations/example/`: 61 tests pass.

- The example's in-memory communication passes C01–C17.
- The scenario tests cover every row of the V4 table: the eight Transfers scenarios against `/cards/v1/details`, the Cards banner (built synchronously, loading its own data, its own error state, and Home's fallback without Cards), and Transfers' home shortcut (returned, opens Transfers, left out without Transfers). Each consumer tells the outcomes apart by status and error code only.

The example host now navigates with go_router, as Core will. `test/example_navigator_test.dart` (13 tests) covers its stack, which is read from go_router's match list with no copy of its own, and the outcomes of `go`, `replace`, `pop`, and `popUntil`, including leaving a route group. The host tests cover the sign-in redirect for guarded routes and deep links, and the not-found screen.

**Manual demo** (`flutter run -d macos`): **Pending (manual)**. It wasn't run in this pass, because it needs someone to look at the app. Widget tests already drive the same flows through the full app: the "Cards installed" toggle swaps the banner and the fallback, and tapping Transfers' shortcut on Home opens Transfers with one row per scenario. Since the example now runs on go_router, the demo should also check opening and closing Transfers and the system back button.

**Result**: Pass. The manual demo is pending.

## V5 — Capabilities and micro app integration with fakes only (US2, US3, SC-002, SC-005, SC-007, SC-011, SC-014, SC-016)

- `flutter test test/capabilities test/micro_app`: 108 tests pass. They cover `ModularAppContainer` (declarations handed over as declared, combined and by micro app, sign-in and owner lookups, stored keys and endpoints by micro app, host paths, initialize and dispose failures reported, a second `bootstrap` refused), its route-tree checks, the reference host sequence built on it, duplicate ids and keys, per-micro-app stores and undeclared keys, `isRegistered` after startup, providers tested directly, feature-flag and session changes, flag defaults, every `ApiExceptionType`, and the missing-contract analyzer test.
- `flutter test --concurrency=8`: all 239 tests pass in parallel, so there is no shared global state.

**Result**: Pass.

## V6 — Interim Design System, delivered through Foundations (US4, SC-009, SC-010, SC-017, SC-018)

- **a. The package itself**: `flutter analyze` reports no issues, and `flutter test` passes 92 tests. They cover the tap-target, label, and contrast guidelines in both themes, 200% text at 320 wide, component states, token contrast, brand derivation, and the dependency rules (`flutter` only, no `foundations` import).
- **b. Delivery through Foundations**: in the example, `design_system 0.1.0` appears only as a transitive dependency of `foundations`. `test/design_system_reexport_test.dart` passes using only `package:foundations/foundations.dart`.
- **c. Git consumers** (`tool/git_consumer_smoke.sh`): pass. `design_system 0.1.0` resolves from git at `b31d34` in `packages/design_system`, in the same pub-cache checkout as `foundations`.
- **d. Gallery**: the gallery's 2 widget tests pass. The manual `flutter run -d macos` check of every state and the light/dark and brand toggles is **Pending (manual)**.

**Result**: Pass. The manual gallery check is pending.

## V7 — Coverage (FR-034)

`flutter test --coverage`: 239 tests pass, and `coverage/lcov.info` reports **100.00%** line coverage of `lib/` (715 of 715 lines). The minimum is 90%.

The project has no CI: the workflow was removed because GitHub Actions can't run on this account. Every check in this report runs locally.

**Result**: Pass.

## V8 — Developer onboarding (SC-004)

**Pending (manual).** SC-004 needs a developer who hasn't used Foundations before to finish the README's guide in under 30 minutes. That can only be measured with a person.

As a stand-in, the guide was followed mechanically in a new package outside the repository, depending on Foundations by path:

- The README's code (the `AccountsBalanceProvider` and `describeSource`) compiled unchanged.
- Registered with the example's communication implementation, the provider and consumer produced every status:
  - 200 for a known account;
  - 400 for an empty account id;
  - 404 for an unknown account;
  - 500 when the backend failed;
  - 404 `endpoint_not_registered` when Accounts wasn't installed.
- The trace name was recorded for each call.

**Result**: Pending. The stand-in run passes.

## V9 — State-management contract through Foundations (US4, SC-002, SC-013)

- `flutter test test/state_management`: 11 tests pass. They cover app-wide, grouped, and screen-owned state under both `RiverpodStateAdapter` and `CubitStateAdapter`, and initial-state overrides.
- `pubspec.yaml`: `switchable_state_bloc` and `switchable_state_riverpod` appear only under `dev_dependencies:`, and `flutter_riverpod` and `flutter_bloc` don't appear at all.

**Result**: Pass.

## V10 — The route contract builds a real router (US2, US3, SC-015)

`flutter test test/routing`: 27 tests pass. The route-tree checks moved to `test/micro_app/modular_app_route_checks_test.dart` (11 tests), where they run through `ModularAppContainer.bootstrap`, which hands the trees over unchanged. The routing tests cover deep links with parameters, `push` with a body, `RouteNotFoundException` for `push` and `popUntil`, `RouteCompleted` and `RouteCancelled`, `go` and `popUntil`, sign-in guards, full-screen dialogs, and route groups that keep their layout and state.

The reference router reads its stack from go_router's match list and keeps no copy. The four newest tests cover the outcomes contracts/routing.md now states: `go` and `replace` complete the openers of the screens they close with `RouteCancelled`, `popUntil` leaves a route group and cancels each opener, and replacing the only screen completes at once with `RouteCancelled`.

**Result**: Pass.

## Open items

1. Run the example demo (V4) and the gallery (V6d) on macOS, and check them visually.
2. Run the timed V8 walkthrough with a developer new to Foundations.
