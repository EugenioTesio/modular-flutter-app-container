# Contract: Component Agent, Conventions, and Check Script

Covers FR-052 to FR-056, User Story 7, and SC-014. The rationale is in [research R16](../research.md#r16-component-agent-and-repository-conventions).

## Files

| File | Purpose |
|------|---------|
| `.claude/agents/ds-component-builder.md` | The Claude Code subagent definition |
| `CLAUDE.md` | Repository conventions that every assistant session and contributor follows |
| `doc/component-checklist.md` | **The** checklist, shared by the agent and human reviewers |
| `tool/check.sh` | The single command that verifies the repository |

## Agent definition

```markdown
---
name: ds-component-builder
description: Builds a new Design System component, or reviews an existing one, against doc/component-checklist.md. Use when asked to create, extend, or audit a Ds* component.
tools: Read, Write, Edit, Grep, Glob, Bash
---
<workflow and rules; summarized below>
```

**Input**: a component request that gives the name (with the `Ds` prefix), the atomic level, the purpose, the variants and states, the parameters, and optionally a design reference. A review request gives the name of an existing component instead.

**Workflow**:
1. Read `CLAUDE.md`, `doc/component-checklist.md`, and one existing component of the same level, together with its tests and use cases.
2. Load the `flutter:widget-composition` skill, which is a house convention: never write a `Widget`-returning method.
3. Produce every artifact in the next section.
4. Run `tool/check.sh`, and fix the failures it can fix.
5. Report a table with each checklist item marked pass or fail, the `tool/check.sh` results, and any open questions.

**Refusals** (FR-054): the agent declines the following and proposes the compliant alternative instead.

| Shortcut asked for | Compliant alternative |
|--------------------|-----------------------|
| A brand or token constructor parameter | Read from `context.ds*` |
| A literal color, size, spacing, radius, opacity, or border width | Use the matching token |
| A primitive color inside a component | Use a semantic role |
| A new primitive token | Draft a brand-schema change request: token path, type, reason, and white-label value. Never add the token silently. |
| A new semantic role | Draft a change to the semantic mapping and the contrast-gate pair list for review |

## What a new component must ship with (`doc/component-checklist.md`)

| # | Item | Checked by |
|---|------|-----------|
| C1 | The class is in `lib/src/components/<level>/ds_<name>.dart`, is `final`, and is exported from `lib/design_system.dart` | architecture test |
| C2 | `static const DsComponentInfo info` with `status: experimental` and `since:` the next version, and the component is added to `DsComponentCatalog.all` | architecture test |
| C3 | Reads tokens only from `context.ds*`, and only semantic color roles | lint rules (0 reports) |
| C4 | No brand or token constructor parameters | architecture test (it scans constructor parameter types) |
| C5 | `String? qaIdentifier`, applied through `DsQaSemantics` on the component's own semantics node | `qa`-tagged test |
| C6 | Output wrapped in `DsGuidelinesFrame(level: info.level)` | guidelines test |
| C7 | House-style documentation: role summary, "Use this…", "This is **not**…", "Built-in:", rules, "Usage from a micro app:", a `**Status**:` line, and constructor `Parameters:` lists | `dart doc --dry-run`, `public_member_api_docs`, architecture test (status line) |
| C8 | Behavior tests in Gherkin (`// Feature:`, `group('Given …')`, `test('When …, Then …')`) | review and agent self-check |
| C9 | Accessibility harness cases for every state, run over every brand and mode | `flutter test` |
| C10 | Golden images for every state × brand × mode | `flutter test --tags golden` |
| C11 | A Widgetbook component with a use case for every variant and state, and knobs for its parameters | Widgetbook coverage test |
| C12 | A CHANGELOG entry under `Unreleased` → `Added` | review and agent self-check |
| C13 | No `Widget`-returning methods: subwidgets are private `StatelessWidget` classes | `widget-composition` skill check |

A **review** reports C1–C13 for an existing component, and names the file and line of each failure (FR-055).

## `tool/check.sh`

The script runs these steps in order, stops at the first failure, and prints which step failed:

1. `flutter analyze` at the root. This includes the plugin, and must report 0 issues.
2. `flutter analyze` in `widgetbook/`.
3. `dart analyze` in `lints/`.
4. `flutter test` at the root, with goldens and the accessibility harness.
5. `flutter test --dart-define=DS_QA_IDENTIFIERS=true --tags qa` at the root.
6. `dart test` in `lints/`.
7. `flutter test` in `widgetbook/`, which includes the coverage check.
8. `dart doc --dry-run` at the root, which must report 0 warnings.

## `CLAUDE.md` (Design System repository)

It contains this container's conventions verbatim (public API documentation, naming, Gherkin tests), plus:

- **Tokens**:
  - Brands come from the tree, never from constructor parameters.
  - Use semantic colors only.
  - No literal visual values. `kMinInteractiveDimension` is used for tap targets.
- **Exceptions**: `…Exception` types live in `lib/src/exceptions/`, and the parser returns a `DsBrandParseResult`.
- **Lifecycle**: new components start `experimental`. A deprecation names its replacement and its removal version, and lasts at least one minor release.
- **Changelog**: every change goes in the CHANGELOG, and a breaking change also gets a Migration note.
- **Before finishing**: run `tool/check.sh`.
- **Delegation**: delegate component work to `ds-component-builder`.
