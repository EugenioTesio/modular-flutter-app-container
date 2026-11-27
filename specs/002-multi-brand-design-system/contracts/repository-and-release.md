# Contract: Repository, Move, Versioning, and Adoption

Covers FR-001 to FR-007 and FR-063 to FR-068, User Stories 1 and 9, and SC-001, SC-002, and SC-015. The rationale is in [research R1](../research.md#r1-repository-move-and-history), [R2](../research.md#r2-two-releases-an-unchanged-move-then-breaking-multi-brand), and [R15](../research.md#r15-versioning-changelog-and-brand-schema-versions).

## Repository rules

| Rule | Check |
|------|-------|
| The package `design_system` sits at the repository root with the single public library `lib/design_system.dart`, the same as 0.1.0 | Foundations' re-export test |
| Runtime dependencies are `{flutter}` only, and nothing in `lib/` imports `package:foundations` | `test/architecture/package_rules_test.dart` (kept from 0.1.0) |
| The `widgetbook/` and `lints/` projects never appear in the package's `dependencies` | architecture test (it reads `pubspec.yaml`) |
| `publish_to: none` for all three projects | review |
| `.gitignore` covers `build/`, `.dart_tool/`, `**/failures/`, and `pubspec_overrides.yaml` | review |

## Release 0.1.1: the move

These steps run in this order. Pushing and tagging are outward-facing, so they wait for the user's go-ahead.

1. In `foundations/`, run `git subtree split --prefix=packages/design_system -b design-system-split`.
2. In a clone of `modular-flutter-app-design-system`, fetch that branch and merge it into `main` with `--allow-unrelated-histories`. This keeps the README commit. No force-push.
3. Add `.gitignore`, a README that describes the package and states that consumers get it through Foundations, and a 0.1.1 CHANGELOG entry: "Moved to its own repository. No API or behavior change."
4. Bump the version to `0.1.1`, push `main`, and tag `v0.1.1`.
5. Foundations `0.1.1`:
   - the dependency becomes `git: {url: …design-system.git, ref: v0.1.1}`;
   - delete `packages/design_system/`;
   - update `tool/git_consumer_smoke.sh` and the README;
   - add a CHANGELOG entry: "Delivers design_system 0.1.1 from its own repository."
6. Container: `git submodule update --init design-system`, check out the `v0.1.1` commit, bump `foundations`, and commit both.

**Acceptance**:
- `git log` in the new repository contains every commit that touched `packages/design_system` in Foundations (SC-002).
- `git diff` between Foundations `0.1.0` and `0.1.1` touches only `pubspec.yaml`, `pubspec.lock`, `packages/design_system/**` (deleted), `tool/git_consumer_smoke.sh`, `README.md`, and `CHANGELOG.md` (SC-001).
- Foundations' tests and its example host pass.

## Release 0.2.0: multi-brand (breaking)

The new repository holds everything in [plan.md](../plan.md#source-code), and `CHANGELOG.md` has a `0.2.0` entry with:
- **Added**: brand files, `DsBrandScope`, `DsTokens`, the `context.ds*` getters, `DsGap`, the contrast gate, QA identifiers, guidelines, the catalog, the Widgetbook, goldens, the lints, and the component agent.
- **Changed**: tokens are brand-driven, brand and mode switches are instant (the App Shell passes `themeAnimationStyle: AnimationStyle.noAnimation`), `DsCard.padding` and `DsLoadingIndicator.size` are nullable, and the files moved into `components/<level>/`.
- **Deprecated**: `DsBrand.standard` (removed in 0.3.0).
- **Removed**: see the table in [design-system-api.md](design-system-api.md#removed-in-020-listed-in-the-migration-section).
- **Migration**: a before/after snippet for each removed or changed element.
- **Brand-schema versions accepted**: 1.

Then:
- Tag `v0.2.0`.
- Foundations `0.2.0`, **marked breaking**:
  - `ref: v0.2.0`;
  - migrate the example and tests (`DsSpacing.md` → `context.dsSpacing.md` or `DsGap`, `themeAnimationStyle: AnimationStyle.noAnimation` on the example's `MaterialApp`, and `ambiguous_exports_test.dart`);
  - a CHANGELOG entry: "Delivers design_system 0.2.0 (breaking: see its Migration notes)."
- Container: bump both submodules.

## Versioning rules

| Rule | Detail |
|------|--------|
| Semver | Before 1.0: a minor release may break, a patch never does. From 1.0: the usual major, minor, and patch. Tags are `vX.Y.Z`. |
| Changelog | Every release has an entry with Added / Changed / Deprecated / Removed / Fixed, and **Migration** whenever anything breaks (FR-065, FR-066). An `Unreleased` section collects the entries between releases. |
| Deprecation | `@Deprecated('Use <replacement>. Removed in <version>.')`, with the component's `info.status` set to deprecated and `info.removedIn` set. The element stays for at least one minor release (FR-067). |
| Experimental | An experimental component may change or be removed in any minor release, and the change is still listed (FR-064). |
| Brand schema | A change to the required tokens bumps `schemaVersion`. The CHANGELOG lists the token paths that changed and the `supportedSchemaVersions` (FR-068). |
| Foundations | Each Foundations release states the `design_system` version it delivers. Adopting a breaking Design System release makes the Foundations release breaking too (the Foundations spec, FR-038). |
