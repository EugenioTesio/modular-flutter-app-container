# Feature Specification: Multi-Brand Design System in Its Own Repository

**Feature Branch**: `001-foundations-package` (no feature branch created; spec directory `specs/002-multi-brand-design-system`)

**Created**: 2026-11-26

**Status**: Draft

**Input**: User description: "I want to move the design system to it's own repository https://github.com/EugenioTesio/modular-flutter-app-design-system and perform the following changes: 1. It has to be able to switch brands, and the brands should be initialized on some module initialization. 2. The brand should be parsed from a JSON with all the fields required to fill the primitive tokens. 3. Then should be able to find the closest brand in the widget tree and should take it to render the components. I don't want every component having a parameter on its constructor with the brand. It should be automatically taken from the nearest brand in the widget tree. 4. It must have a widget book with all the components and with an add-on that allows me to switch brands. 5. Widget book should be in another internal Flutter project. 6. The white label brand should be hard-coded. 7. It should have an agent to help the developers to build components. 8. The tests should have golden tests with the image persisted on the repository. Then test should have a comparator because it's not the same generating the golden test on the Mac computer and performing the test on a Linux computer. should allow a 5% pixel difference. 9. Research about any other thing that can be useful in building a professional design system."

## Clarifications

### Session 2026-11-26

- Q: Which module places brands, given that the Design System can't depend on Foundations? → A: The App Shell places the top-level brand at startup. A micro app can wrap its screens in another brand component that carries a different set of primitive tokens, and every Design System component below takes that brand's look. Where and when the App Shell or a micro app fetches a brand file (bootstrap, initialization, network, or bundled assets) is out of scope.
- Q: Which format does a brand file use? → A: The W3C Design Tokens Community Group format, which has been stable since version 2025.10. Figma Variables and Tokens Studio export it directly.
- Q: Which practices from the research are in scope? → A: The following are in scope:
  - a brand contrast gate;
  - a token lint rule;
  - component lifecycle and versioning;
  - a device-frame addon in the Widgetbook, delivered as phone and tablet viewports, because Widgetbook deprecated its device-frame addon;
  - two features from the user's earlier `ueno_bank` design system: QA identifiers that a build switch turns on, and a developer guidelines overlay that outlines components by atomic level.

  A hosted Widgetbook and text-scale and locale/RTL addons are out of scope.
- Q: When the brand changes while the app is running, should components switch instantly or animate to the new look? → A: Always instant. Every component, at the top level and in every brand area, shows the new brand on the next frame. The App Shell turns off the app's theme animation, which also makes the light/dark switch instant.
- Q: What should the Widgetbook do with a valid brand file that fails the contrast gate? → A: Keep it in the brand switcher, marked with a warning, and show its contrast report, so a designer can see the components in that brand while fixing its colors.
- Q: Can the developer guidelines overlay turn on in release builds? → A: No. Release builds ignore the build switch, the App Shell's startup toggle, and the Widgetbook toggle, so end users never see the overlay. Testers use it on debug or profile builds.

## User Scenarios & Testing *(mandatory)*

The users of the Design System are the people who build and review the product's screens:

- **The Design System team** builds and releases components.
- **The App Shell team** brands the product.
- **Micro app teams** build screens with the components and may brand their own screens.
- **Designers** hand over brand files and review components.
- **QA engineers** automate tests against the screens.

### User Story 1 - Move the Design System to its own repository without breaking anyone (Priority: P1)

The Design System team moves the interim package from the Foundations repository to its own repository and keeps its history. The first release from the new repository is the same package: same name, same public library, same behavior. Foundations switches its dependency to that release, and every micro app keeps working without a change.

**Why this priority**: Every other story in this spec lands in the new repository. The Foundations spec promised that the move would cost consumers nothing. Moving first, before any behavior changes, keeps that promise and separates where the package lives from what it does.

**Independent Test**:
1. Publish the first release from the new repository and point Foundations at it.
2. Run Foundations' tests and its example host.
3. Confirm that no micro app or example file changed, and that the interim folder no longer exists in Foundations.

**Acceptance Scenarios**:

1. **Given** the interim package in the Foundations repository, **When** it moves, **Then** the new repository holds the package's full commit history. **And** its first release has the same package name, public library, public names, and behavior.
2. **Given** that first release, **When** Foundations switches its dependency source to it, **Then** Foundations' tests and its example host pass with no change to any micro app or example code.
3. **Given** Foundations after the move, **When** its contents are inspected, **Then** the interim folder is gone, and Foundations' dependency checks and documentation name the new repository as the Design System's source.
4. **Given** the container repository, **When** its submodules are inspected, **Then** the existing `design-system` submodule points to the commit of the release Foundations pins.
5. **Given** a consumer that resolves Foundations from git, **When** its dependencies resolve, **Then** exactly one Design System version is resolved: the one Foundations pins.

---

### User Story 2 - Brand the whole app from a brand file (Priority: P1)

The App Shell has a brand file that a designer exported from the design tool. At startup, it turns the file into a brand and places that brand at the top of the widget tree. Every Design System component on every screen then draws itself with that brand's colors, typography, spacing, and shapes.

No component takes a brand in its constructor. Each one finds the nearest brand above it.

When the product needs a different look, the App Shell replaces the top-level brand, and every component updates in place without an app restart. For example, a user might pick another brand in settings. When no brand is placed above a component, it uses the built-in white-label brand.

**Why this priority**: Brand switching is the reason this feature exists. Reading the brand from the tree, rather than from constructor parameters, is what lets one set of components serve every brand without any change to micro app code.

**Independent Test**:
1. Build a host that places a brand parsed from a sample brand file at the top of the tree, and show a screen with every component.
2. Confirm that every component uses that brand's values.
3. Replace the brand, and confirm that every component updates without losing what the user typed.
4. Remove the brand, and confirm that the components fall back to the white label.

**Acceptance Scenarios**:

1. **Given** a valid brand file, **When** the App Shell parses it and places the brand at the top of the tree, **Then** every Design System component below renders with that brand's tokens. **And** no component received a brand through its constructor.
2. **Given** a brand file that has any of the following problems, **When** it is parsed, **Then** parsing fails with one report that names every problem and where it is in the file, and no brand is produced:
   - missing values;
   - values of the wrong type;
   - references to tokens that don't exist;
   - circular references;
   - an unsupported schema version.
3. **Given** a brand file with extra tokens that the Design System doesn't use, **When** it is parsed, **Then** parsing succeeds and the extra tokens are ignored.
4. **Given** a running app with brand A at the top, **When** the App Shell replaces it with brand B, **Then** every visible component shows brand B's values on the next frame, with no animation between the two brands and without a restart. **And** the screen keeps its state, such as text typed in a field and the scroll position.
5. **Given** no brand anywhere above a component, **When** the component renders, **Then** it uses the white-label brand. **And** in a debug build, the developer gets one warning that no brand was placed.
6. **Given** the reference white-label brand file in the repository, **When** it is parsed, **Then** the result equals the built-in white-label brand.
7. **Given** a brand that names a font family the app doesn't provide, **When** text renders, **Then** it falls back to the platform font instead of failing.
8. **Given** the app is in light or dark mode, **When** a brand is placed, **Then** the components use that brand's light or dark semantic colors, which the Design System derives from the brand's primitive tokens.

---

### User Story 3 - Brand a micro app's own screens (Priority: P2)

A micro app needs its own look inside the product, such as a partner section with co-branded colors. It already has a brand file's contents; how and when it got them is up to the micro app. It parses them and wraps one of its screens, or a group of its screens, in the Design System's brand component. Every Design System component inside takes that brand's look, and the rest of the app keeps the App Shell's brand.

**Why this priority**: The product needs co-branding and sub-brands, and each micro app owns its own screens. Wrapping a screen is enough to rebrand it, so a micro app never has to touch component code or ask the App Shell to restyle its screens.

**Independent Test**:
1. In a host branded with brand A, show a micro app screen wrapped in brand B next to a host screen.
2. Confirm that components inside the wrapped screen use brand B and components outside it use brand A.
3. Open a dialog from the wrapped screen, and confirm that it uses brand B.

**Acceptance Scenarios**:

1. **Given** the App Shell's brand A and a micro app screen wrapped in brand B, **When** the screen renders, **Then** every Design System component inside the screen uses brand B, and every component outside it uses brand A.
2. **Given** brands nested three levels deep, **When** a component renders, **Then** it uses the nearest brand above it.
3. **Given** a screen wrapped in brand B, **When** it opens a dialog, bottom sheet, or menu through the platform's standard functions, **Then** the overlay uses brand B, not the top-level brand.
4. **Given** the app switches between light and dark mode, **When** a screen wrapped in brand B is visible, **Then** that screen follows the app's mode, using brand B's light or dark colors.
5. **Given** the App Shell replaces its top-level brand, **When** a screen wrapped in brand B is visible, **Then** that screen keeps brand B.
6. **Given** a micro app that depends only on Foundations, **When** it parses a brand file and wraps a screen in a brand, **Then** the brand component and the parser are available through Foundations, with no direct Design System dependency.

---

### User Story 4 - Review every component in the Widgetbook (Priority: P2)

Designers and developers open the Widgetbook, a separate internal Flutter project inside the Design System repository, to see every component in every variant and state. Addons let them switch the brand, the light or dark mode, the device viewport, and the guidelines overlay without changing code. A designer who exports a new brand file drops it into the Widgetbook's brands folder and sees every component in that brand.

**Why this priority**: The Widgetbook is where designers and developers agree on what a component looks like in each brand before it reaches a micro app. It is also the fastest way to check a new brand file.

**Independent Test**:
1. Run the Widgetbook and open each component's use cases.
2. Switch through every brand, both modes, and a phone and a tablet viewport, and confirm that each component reflects each choice.
3. Add a new brand file to the brands folder and restart the Widgetbook. Confirm that the brand appears in the switcher.

**Acceptance Scenarios**:

1. **Given** the Design System repository, **When** its layout is inspected, **Then** the Widgetbook is its own Flutter project, outside the published package, and nothing in the package depends on it.
2. **Given** the Widgetbook, **When** a reviewer browses it, **Then** every public component has use cases for each of its variants and states, grouped by atomic level, with inputs the reviewer can adjust.
3. **Given** the brand switcher addon, **When** the reviewer opens it, **Then** it lists the white-label brand and every brand file in the Widgetbook's brands folder. **And** choosing one re-renders the current use case in that brand.
4. **Given** an invalid brand file in the brands folder, **When** the Widgetbook starts, **Then** that brand is left out of the switcher and its problem report is shown. **And** the other brands still work.
5. **Given** the light/dark, device-viewport, and guidelines-overlay addons, **When** the reviewer changes them, **Then** the use case re-renders in the chosen mode and device viewport, with or without the guidelines overlay.
6. **Given** a public component with no use case, **When** the repository's checks run, **Then** they fail and name the component.
7. **Given** any use case, **When** it is shown, **Then** it displays the component's lifecycle status.
8. **Given** a valid brand file in the brands folder that fails the contrast gate, **When** the Widgetbook starts, **Then** the brand is in the switcher, marked with a warning, and the reviewer can select it. **And** the Widgetbook shows its contrast report, with each failing pair's mode, roles, measured ratio, and required ratio.

---

### User Story 5 - Catch visual regressions with golden images (Priority: P2)

Each component's tests compare what it draws with reference images (golden images) committed to the repository. A developer generates the images on a Mac. The same tests pass on a Linux machine because the comparison tolerates the small differences in how the two platforms draw text and edges: up to 5% of the pixels may differ. A real visual change fails the test and leaves images that show the difference.

**Why this priority**: When one set of components serves many brands, a change for one brand can break another without anyone noticing. Golden images catch that on any developer's machine, whatever its operating system.

**Independent Test**:
1. Generate the golden images on a Mac and commit them.
2. Run the same tests on a Linux machine, and confirm that they pass.
3. Change a component's primary-action color and run the tests again. Confirm that they fail and produce difference images.

**Acceptance Scenarios**:

1. **Given** golden images generated on a Mac and committed, **When** the tests run on a Linux machine with no visual change, **Then** they pass.
2. **Given** a comparison where 5% of the pixels or fewer differ, **When** the test runs, **Then** it passes and reports the measured difference.
3. **Given** a comparison where more than 5% of the pixels differ, **When** the test runs, **Then** it fails, names the image and the measured difference, and writes difference images to a location that the repository doesn't track.
4. **Given** a test whose golden image doesn't exist yet, **When** it runs, **Then** it fails and says how to generate the image.
5. **Given** a developer who changed a component on purpose, **When** they run the documented update command, **Then** the golden images are regenerated, and the change appears in the repository diff for review.
6. **Given** any component, **When** its golden images are listed, **Then** there is an image for each of its states, in the white-label brand and the sample brand, in light and dark mode.
7. **Given** any golden image, **When** a reviewer opens it, **Then** its text is readable, because it is drawn with a font that only the tests use.

---

### User Story 6 - Guardrails that keep brands accessible and components on tokens (Priority: P2)

Two checks keep the system honest as brands multiply.

- **The brand contrast gate** checks a brand in both light and dark mode and lists every semantic color pair that falls below its required contrast.
- **The token lint rule** flags code that bypasses tokens, such as a literal color in a component or a literal spacing value in a micro app screen.

**Why this priority**: Brands now come from files that designers edit, so a single color change can make text unreadable in one mode. Components and screens that use literal values silently ignore the brand. Both problems are cheap to catch automatically and expensive to find later.

**Independent Test**:
1. Run the gate on a brand with a low-contrast primary color, and confirm that it names the failing pairs.
2. Write a component with a literal color, and confirm that the lint flags it and suggests a token.

**Acceptance Scenarios**:

1. **Given** a brand, **When** the contrast gate runs, **Then** it checks the following pairs in both light and dark mode:
   - every text role on its intended background, at 4.5:1 or more;
   - every interface pair, such as the primary action against the background or the focus indicator against the surface, at 3:1 or more.

   **And** it reports each failing pair with its mode, its two roles, the measured ratio, and the required ratio.
2. **Given** the white-label brand and every brand file in the repository, **When** the repository's tests run, **Then** all of them pass the gate. **And** a brand that fails the gate fails the tests.
3. **Given** an App Shell or micro app with its own brand file, **When** its developers run the gate in their own tests, **Then** they get the same report, using the gate through Foundations.
4. **Given** a brand that fails the gate, placed in the tree, **When** the app runs in a debug build, **Then** the developer gets one warning that names the failing pairs. **And** the brand still renders exactly as its file specifies, with no automatic color correction.
5. **Given** code inside a Design System component that uses a literal color, size, spacing, or radius, or a primitive color instead of a semantic role, **When** the lint runs, **Then** it reports each occurrence and suggests the token to use.
6. **Given** a micro app or the App Shell that has opted into the lint rule, **When** a screen's code uses a literal color or spacing value, **Then** the lint reports it and points to the token or spacing helper to use.
7. **Given** the 48 × 48 minimum tap target, **When** the lint runs, **Then** that accessibility value is not reported.

---

### User Story 7 - Build components with the component agent (Priority: P3)

A developer asks the repository's AI agent for a new component. They give its name, atomic level, purpose, variants, and states, and optionally a design reference. The agent produces the component and everything the repository requires of it, following the repository's conventions. It can also review an existing component against the same checklist.

**Why this priority**: Every new component carries many rules: tokens from the nearest brand, semantic roles only, accessibility, documentation style, Given/When/Then tests, golden images for every brand and mode, Widgetbook use cases, QA identifiers, the atomic level, the lifecycle status, and the changelog. An agent that knows them all lets developers who are new to the repository contribute correctly from the first change. It comes after the system it helps build.

**Independent Test**: Ask the agent for a simple new atom. Confirm that the result passes every repository check with no manual fixes for conventions. The checks are analysis, documentation, tests, golden images, accessibility, contrast, lint, and Widgetbook coverage.

**Acceptance Scenarios**:

1. **Given** the repository, **When** a developer opens it with the AI assistant, **Then** the component agent is available from a definition committed in the repository, together with the repository's conventions file.
2. **Given** a component request, **When** the agent completes it, **Then** it has produced all of the following:
   - the component, reading tokens only from the nearest brand and using only semantic color and typography roles;
   - public documentation in the house style;
   - Given/When/Then tests;
   - accessibility tests;
   - golden images for each state, brand, and mode;
   - a Widgetbook use case for each variant and state;
   - support for a QA identifier;
   - its atomic level for the guidelines overlay;
   - the "experimental" status;
   - a changelog entry.
3. **Given** a request that would need a brand constructor parameter, a literal color, or a new primitive token, **When** the agent handles it, **Then** it declines the shortcut and proposes the compliant alternative: a semantic role, an existing token, or a request to extend the brand schema.
4. **Given** an existing component, **When** a developer asks the agent to review it, **Then** the agent lists every checklist item that the component fails.
5. **Given** the agent has finished a component, **When** it reports back, **Then** it has run the repository's checks and reports their results.

---

### User Story 8 - Use QA identifiers and the developer guidelines overlay (Priority: P3)

QA engineers need stable identifiers to find elements in automated UI tests, without changing what screen readers announce. Developers and designers need to see at a glance which parts of a screen are Design System components, at which atomic level, and which spacing comes from tokens.

**Why this priority**: Both features speed up QA and design review on real screens, and the user relied on both in an earlier design system. They don't change what end users see, so they come after the features that do.

**Independent Test**:
1. Build the same screen with the QA switch off and then on, and confirm that the identifiers appear only when the switch is on.
2. Turn on the overlay, and confirm that each component is outlined in its level's color and that spacing gaps are tinted, with no change in layout.

**Acceptance Scenarios**:

1. **Given** a component that has a QA identifier, **When** the app is built with the QA switch on, **Then** UI automation tools can find the component by that identifier.
2. **Given** the same component, **When** the app is built with the QA switch off (the default), **Then** no QA identifier is exposed. **And** the accessibility tree is the same as it would be without identifiers.
3. **Given** a QA identifier, **When** a screen reader reads the component, **Then** it announces the same accessible label it would without the identifier.
4. **Given** the guidelines overlay is on, **When** a screen renders, **Then** each Design System component is outlined in the color of its atomic level (atom, molecule, organism, or template). **And** spacing gaps that come from the Design System's spacing helpers are tinted. In debug and profile builds, the overlay can be turned on by a build switch, by the App Shell at startup, or by the Widgetbook toggle.
5. **Given** the overlay is on, **When** a screen renders, **Then** no element moves or changes size compared with the overlay off.
6. **Given** the overlay is off (the default), **When** a screen renders, **Then** components show no outline and no tint.
7. **Given** a release build, **When** the build switch, the App Shell, or the Widgetbook asks for the overlay, **Then** no outline or tint is shown.

---

### User Story 9 - Follow the component lifecycle and versioning (Priority: P3)

Consumers of the Design System need to know how far they can rely on each component, what changed in each release, and how to move off anything that is going away. Brand files need the same clarity, because a file made for one release of the Design System may not suit the next.

**Why this priority**: The Design System now has its own repository and release cycle, and Foundations adopts its releases. Clear statuses, semantic versions, changelogs, and deprecation rules let both repositories upgrade without surprises.

**Independent Test**:
1. Inspect a release: confirm its changelog entry, a migration note for each breaking change, and a status on every component.
2. Deprecate an element, and confirm that its use site shows a warning naming the replacement and the removal version.
3. Parse a brand file that targets an unsupported schema version, and confirm that the parser rejects it and names the supported versions.

**Acceptance Scenarios**:

1. **Given** any public component, **When** a developer reads its documentation or its Widgetbook use case, **Then** its status is shown: experimental, stable, or deprecated.
2. **Given** a release, **When** it is published, **Then** its version follows semantic versioning. **And** the changelog has an entry for it that lists what was added, changed, deprecated, and removed.
3. **Given** a breaking change, **When** it is released, **Then** the changelog includes a migration note that says what to change in consumer code. **And** the Foundations release that adopts it is marked as breaking.
4. **Given** a deprecated element, **When** consumer code uses it, **Then** a deprecation warning at the use site names its replacement and the version that removes it. **And** the element stays available for at least one minor release before it is removed.
5. **Given** a brand file, **When** it is parsed, **Then** it must state the brand-schema version it targets. **And** a file that targets a version the Design System doesn't support is rejected with a message that names the supported versions.
6. **Given** a release that changes which primitive tokens a brand must supply, **When** it is published, **Then** the brand-schema version is bumped. **And** the changelog lists the tokens that were added, renamed, or removed.

---

### Edge Cases

#### Brand files

- **The file isn't valid JSON**: parsing fails with one problem that says where the file stops being readable.
- **References**: a reference to a token that doesn't exist, or a chain of references that loops back on itself, is reported with the path of every token involved.
- **Composite values**: an incomplete composite value, such as a text style missing its line height, is reported for each missing part.
- **Unsupported content**: values the format doesn't define, such as math expressions some design tools add, and token types the Design System doesn't use for primitives, are reported with their location when they sit on a required token. Elsewhere they are ignored.
- **Color spaces**: colors in a color space the platform can't draw are rejected with their location. sRGB colors are always accepted.
- **The same brand twice**: two brands parsed from equivalent files are equal, so placing the same brand again changes nothing on screen.
- **No brand above a component**: the component uses the white-label brand, and debug builds warn once per app run.
- **Missing fonts**: a font family that the app (or the micro app) doesn't provide falls back to the platform font. The Design System bundles no fonts. The test-only font is not part of the package.
- **A token the schema doesn't have**: a micro app that needs one can't add it in its brand file. It asks the Design System team, who add it in a release that bumps the brand-schema version.

#### Brand placement

- **Switching during a page transition**: replacing the top-level brand while a page transition is running completes the transition and shows the new brand, without errors.
- **Overlays**: a dialog, sheet, or menu opened from inside a micro app's brand area uses that area's brand, even though overlays are drawn above the screen.
- **Grouped screens**: a brand that wraps a group of screens with a shared layout applies to every screen in the group, and to the shared layout.

#### Golden images

- **Diluted differences**: a 5% tolerance could hide a real change in a large image. Each golden image therefore shows one component, in one state, for one brand and mode, so a change to a color that fills a large part of the component, such as a button's fill, goes over the threshold. Thin details, such as a focus border or a text color, can stay under it, and the components' behavior tests check those colors instead.
- **Different screens**: golden images are drawn at a fixed size and pixel ratio, so the machine's screen density doesn't affect them.
- **Difference images**: images written on failure are never committed.

#### Guardrails and QA identifiers

- **Lint scope**: the lint covers the package's components and opted-in consumer code. Tests and Widgetbook use cases may use literal values for test data and are exempt.
- **Duplicate QA identifiers**: two components with the same QA identifier on one screen are the screen's responsibility. The Design System can't see the whole screen, so it doesn't enforce uniqueness.

#### Dependencies

- **Dependency cycle**: Foundations depends on the Design System, so the Design System, the Widgetbook, and the lint package never depend on Foundations.

## Requirements *(mandatory)*

### Functional Requirements

#### Repository and delivery

- **FR-001**: The Design System MUST live in its own repository, `modular-flutter-app-design-system`, moved from the Foundations repository with its commit history preserved.
- **FR-002**: The first release from the new repository MUST be the interim package unchanged: the same package name, public library, public names, and behavior. When Foundations adopts it, Foundations MUST change only three things: its dependency source, the removal of the interim folder, and its own dependency checks and documentation. Micro apps and Foundations' example MUST change no line.
- **FR-003**: The multi-brand changes in this spec MUST ship in a later release than the move. That release MUST be marked as breaking and come with migration notes (see FR-066).
- **FR-004**: The package's runtime dependencies MUST be limited to the UI framework SDK. The package MUST NOT depend on Foundations or on any other product package. The Widgetbook project, the lint package, the test-only font, and test tools MUST NOT become runtime dependencies of the package.
- **FR-005**: Foundations MUST keep delivering the Design System by re-exporting its public surface unchanged, at exactly one pinned version. The rules of the Foundations spec still apply. The re-exported surface includes the brand component, the brand file parser, the contrast gate, and the spacing helpers.
- **FR-006**: The container repository's `design-system` submodule MUST point to the new repository, at the commit of the release Foundations pins.
- **FR-007**: The new repository MUST contain:
  - the package;
  - the Widgetbook project;
  - the lint package;
  - the reference white-label brand file and at least one sample brand file;
  - the component agent's definition;
  - a conventions file for contributors and AI assistants, with the same conventions as this container: public API documentation, naming, and Given/When/Then tests.

#### Brand file and parsing

- **FR-008**: A brand MUST be created from a brand file in the W3C Design Tokens Community Group format (version 2025.10). The file MUST supply every primitive token listed in FR-019.
- **FR-009**: A brand file MUST carry the brand's identifier, its display name, and the brand-schema version it targets.
- **FR-010**: Parsing MUST be strict and complete. It either produces a brand, or fails with one report that lists every problem, each with its token path and its reason. The reasons are:
  - a missing token;
  - a wrong type;
  - an invalid value;
  - an unresolved reference;
  - a circular reference;
  - an unsupported color space or value format;
  - an unsupported schema version;
  - a malformed file.
- **FR-011**: Parsing MUST resolve the references between tokens (aliases) that the format defines.
- **FR-012**: Parsing MUST ignore tokens that the Design System doesn't use, without failing.
- **FR-013**: Parsing MUST take the file's contents, either as text or as already-decoded data. It MUST NOT read files, assets, or the network itself. Fetching, caching, and choosing brand files is the caller's job and is out of scope.
- **FR-014**: Parsing MUST return its failure report to the caller as a result value. It MUST NOT crash or throw for a bad file.
- **FR-015**: Two brands parsed from equivalent files MUST be equal.

#### White-label brand

- **FR-016**: The Design System MUST include a white-label brand defined in its own code, not loaded from a file. It is always available, with no setup.
- **FR-017**: The repository MUST include a reference brand file whose parsed result equals the white-label brand. A check MUST fail when the two drift apart. This file is the starting template for new brands.
- **FR-018**: The white-label brand MUST pass the contrast gate in both modes.

#### Tokens and components

- **FR-019**: The primitive tokens that a brand supplies MUST include:
  - **Color scales**: a brand (primary) scale, a neutral scale, and success, warning, error, and information scales.
  - **Typography**: font families, and a type scale (size, line height, weight, and letter spacing) for the display, headline, title, body, strong body, label, and caption styles.
  - **A spacing scale.**
  - **A corner-radius scale.**
  - **The visual values that components hard-code today**: border widths, state-layer opacities (pressed, hovered, and focused), and icon sizes.
- **FR-020**: The Design System MUST derive the semantic tokens for light and dark mode from the brand's primitive tokens, using one mapping that it owns. The semantic tokens are the color roles and typography roles the interim package defines today. Brands MUST NOT override semantic roles.
- **FR-021**: The Design System MUST render a brand exactly as its file specifies. It MUST NOT alter a primitive value, for example to reach a contrast ratio. The contrast gate reports such problems instead.
- **FR-022**: Components MUST read every token from the nearest brand above them in the widget tree.
  - Colors and text styles MUST come only from semantic roles, never from primitive colors.
  - Spacing, radius, border widths, opacities, and icon sizes MUST come from the nearest brand's scales.
- **FR-023**: No component constructor may take a brand or a set of tokens.
- **FR-024**: Components MUST NOT hard-code any visual value. The only exception is the 48 × 48 minimum tap target that accessibility requires.
- **FR-025**: The Design System MUST provide spacing helpers (gaps and insets) that read the nearest brand's spacing scale, so screens never need literal spacing values.
- **FR-026**: The eight current components MUST be carried over with their current behavior: the button, text field, loading indicator, card, list row, inline message, empty state, and page template.
  - Each MUST keep the accessibility rules of the Foundations spec, in both modes and in every repository brand.
  - The brand-change rule now reads: changing the brand's primary scale changes every component that uses the primary-action role.

#### Brand placement, switching, and nesting

- **FR-027**: The Design System MUST provide a brand component that places a brand over a part of the widget tree. Every Design System component below it MUST use the nearest such brand.
- **FR-028**: The App Shell MUST be able to place a top-level brand at startup and replace it at runtime. Replacing it MUST update every component below on the next frame, without an app restart and without losing widget state. The switch MUST be instant, with no animation between the old and new values. The same holds when a brand area's brand changes and when the app switches between light and dark mode. If the app animates theme changes, debug builds MUST warn once per app run that brand and mode switches aren't instant.
- **FR-029**: A micro app MUST be able to wrap one screen, or a group of screens, in its own brand using Foundations alone. When brands are nested, the nearest one applies.
- **FR-030**: Overlays opened from inside a brand area through the platform's standard functions, such as dialogs, sheets, and menus, MUST use that area's brand.
- **FR-031**: Light and dark mode MUST come from the app, not from the brand. Every brand area follows the app's current mode.
- **FR-032**: A component with no brand above it MUST use the white-label brand. Debug builds MUST warn once per app run, and release builds MUST NOT warn.
- **FR-033**: Inside a brand area, the app theme MUST match that brand, so that platform elements used inside it match the brand's colors and typography. Examples are the text cursor and the selection handles.

#### Widgetbook

- **FR-034**: The repository MUST contain a Widgetbook: a separate, internal, unpublished Flutter project that depends on the package and that nothing depends on. It replaces the interim package's example gallery.
- **FR-035**: Every public component MUST have Widgetbook use cases that:
  - cover each of its variants and states;
  - are grouped by atomic level;
  - have adjustable inputs for its parameters;
  - show its lifecycle status.
- **FR-036**: The Widgetbook MUST provide a brand switcher addon that lists the white-label brand and every brand file in the Widgetbook's brands folder. Adding a brand MUST take only a new file in that folder and a restart, with no code change.
- **FR-037**: An invalid brand file MUST be left out of the switcher, and its problem report MUST be shown, without affecting the other brands. A valid brand file that fails the contrast gate MUST stay in the switcher, marked with a warning, and its contrast report MUST be shown.
- **FR-038**: The Widgetbook MUST also provide these addons:
  - light/dark mode;
  - device viewports, with at least one phone size and one tablet size;
  - a guidelines-overlay toggle.
- **FR-039**: A repository check MUST fail when a public component has no Widgetbook use case.

#### Golden tests

- **FR-040**: Every component MUST have golden tests for each of its states, in the white-label brand and the sample brand, in light and dark mode. The golden images MUST be committed to the repository.
- **FR-041**: A comparison MUST pass when 5% of the image's pixels or fewer differ from the golden image, and MUST fail above that. The threshold MUST be set in one place.
- **FR-042**: A failure MUST name the image and the measured difference, and MUST write difference images to a location that the repository doesn't track.
- **FR-043**: Each golden image MUST show one component, in one state, for one brand and mode. That way, a change to a color that fills a large part of the component goes over the threshold. Thin details, such as a focus border or a text color, MUST be covered by the components' behavior tests.
- **FR-044**: Golden images MUST draw readable text with a font that only the tests use, at a fixed size and pixel ratio. They MUST NOT depend on the machine's fonts or screen.
- **FR-045**: Regenerating golden images MUST take one documented command. Golden images generated on macOS MUST pass on Linux.

#### Guardrails

- **FR-046**: The Design System MUST provide a contrast gate that checks a brand in both modes:
  - every text role on its intended background, at 4.5:1 or more;
  - every large-text and interface pair, at 3:1 or more.

  The gate reports every failing pair with its mode, its two roles, the measured ratio, and the required ratio. Hosts and micro apps can use it through Foundations.
- **FR-047**: The repository's tests MUST run the contrast gate on the white-label brand and on every brand file in the repository, and MUST fail on any failing pair.
- **FR-048**: In a debug build, placing a brand that fails the gate MUST produce one warning that names the failing pairs. Release builds MUST NOT run the gate automatically.
- **FR-049**: The repository MUST provide a token lint rule as a dev-only package.
  - In Design System components, it reports literal colors, sizes, spacing, and radius, and primitive colors used instead of semantic roles.
  - In consumer code that opts in, it reports literal colors and spacing values.
  - Each report suggests the token or helper to use.
  - The 48 × 48 tap target, tests, and Widgetbook use cases are exempt.
- **FR-050**: The Design System's own components MUST produce zero lint reports.
- **FR-051**: Consumers MUST opt into the lint through their analysis settings. The lint package MUST NOT add a runtime dependency, and MUST NOT affect the guarantee that only one Design System version is resolved.

#### Component agent

- **FR-052**: The repository MUST contain an AI assistant agent definition for building components, together with the conventions file it follows.
- **FR-053**: Given a component request, the agent MUST produce everything listed in User Story 7, acceptance scenario 2.
- **FR-054**: The agent MUST decline non-compliant shortcuts and propose the compliant alternative. Examples of shortcuts are a brand constructor parameter, a literal visual value, and an unplanned primitive token.
- **FR-055**: The agent MUST be able to review an existing component and list every checklist item it fails.
- **FR-056**: Before finishing, the agent MUST run the repository's checks and report their results.

#### QA identifiers and guidelines overlay

- **FR-057**: Every component MUST accept an optional QA identifier.
- **FR-058**: The QA identifier MUST be exposed to UI automation only when the app is built with the QA switch on. The switch is off by default, in every build mode.
- **FR-059**: QA identifiers MUST NOT change accessible labels, or anything else that screen readers announce.
- **FR-060**: Each component MUST declare its atomic level: atom, molecule, organism, or template.
- **FR-061**: The guidelines overlay MUST outline each component in its level's color and tint the spacing gaps that come from spacing helpers. In debug and profile builds, a build switch, the App Shell at startup, or the Widgetbook toggle turns it on. It is off by default. Release builds MUST ignore all three and never show the overlay.
- **FR-062**: The overlay MUST NOT change layout. When it is off, it MUST cause no visible change.

#### Lifecycle and versioning

- **FR-063**: Each public component MUST have a status: experimental, stable, or deprecated. The status MUST be shown in its documentation and in its Widgetbook use cases.
- **FR-064**: The eight carried-over components MUST start as stable. Components added after this feature MUST start as experimental.
- **FR-065**: Releases MUST follow semantic versioning, and every release MUST have a changelog entry that lists additions, changes, deprecations, and removals.
- **FR-066**: Every breaking change MUST come with a migration note. Foundations MUST mark the release that adopts a breaking Design System release as breaking.
- **FR-067**: A deprecated element MUST name its replacement and its removal version, MUST produce a warning at the use site, and MUST stay available for at least one minor release before removal.
- **FR-068**: Each Design System release MUST state the brand-schema versions it accepts. A change to the primitive tokens that a brand must supply MUST bump the brand-schema version, and the changelog MUST list the tokens that were added, renamed, or removed.

### Key Entities

- **Brand**: A complete set of primitive token values with an identifier and a display name. Two brands with the same values are equal.
- **Brand File**: A design-tokens document that describes one brand. It is exported from the design tool or written by hand, and it states the brand-schema version it targets.
- **White-Label Brand**: The brand built into the Design System's code. It is used wherever no other brand is placed, and it is mirrored by the reference brand file.
- **Primitive Token**: A base value that a brand supplies: a color scale step, a font family, a type-scale entry, a spacing step, a radius step, a border width, an opacity, or an icon size.
- **Semantic Token**: A role that expresses intent, such as the primary-action color or the body text style. The Design System derives semantic tokens from primitive tokens for light and dark mode.
- **Brand Area**: The part of the widget tree under one brand component. Components resolve to the nearest brand area above them.
- **Brand Problem Report**: The list of every problem found while parsing a brand file. Each problem has a token path and a reason.
- **Contrast Report**: The list of semantic color pairs that fail their required contrast. Each pair has its mode, its roles, the measured ratio, and the required ratio.
- **Golden Image**: A committed reference image of one component, in one state, for one brand and mode.
- **Widgetbook Use Case**: A named, adjustable example of one component variant or state in the Widgetbook.
- **Component Status**: Experimental, stable, or deprecated.
- **Component Level**: Atom, molecule, organism, or template.
- **QA Identifier**: An optional, stable name for a component that UI automation can see only when the QA switch is on.
- **Brand-Schema Version**: The version of the set of primitive tokens that brand files must supply.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Moving the Design System changes 0 lines in micro apps and in Foundations' example. In Foundations, only the dependency source, the removed interim folder, and Foundations' own dependency checks and documentation change.
- **SC-002**: 100% of the interim package's commits appear in the new repository's history.
- **SC-003**: 0 public component constructors accept a brand or a set of tokens, and 100% of components render with the nearest brand above them.
- **SC-004**: Replacing the top-level brand updates 100% of visible components on the next frame, and 0 frames show values between the two brands. On the test screen, 100% of the screen state (typed text and scroll position) survives the switch.
- **SC-005**: A brand file with N problems, of any mix of the reasons listed in FR-010, is rejected with exactly N problems reported, each with its token path.
- **SC-006**: Parsing and placing a complete brand adds less than 100 ms to app startup on a mid-range phone.
- **SC-007**: A designer can view a newly exported brand file across every component in the Widgetbook within 10 minutes, with no code change.
- **SC-008**: 100% of public components have Widgetbook use cases for every variant and state. They also have golden images for every state, in every repository test brand, in both modes.
- **SC-009**:
  - With no visual change, golden images generated on a Mac pass on a Linux machine in 100% of runs.
  - When the sample brand's primary scale changes, 100% of the golden images on the primary-color list fail, in both modes. The list holds the images where the primary-action color fills a large area: the filled primary button (enabled and loading) and the loading indicator.
- **SC-010**: 100% of the brands in the repository, the white label included, pass the contrast gate in both modes. A brand with K failing pairs is reported with all K pairs.
- **SC-011**:
  - The Design System's components produce 0 lint reports.
  - A consumer screen that has opted in and uses a literal color gets a lint report in 100% of cases.
- **SC-012**: With the QA switch on, 100% of components that are given a QA identifier expose it to UI automation. With the switch off, 0 do, and the accessibility tree is identical to that of a build without identifiers.
- **SC-013**: Turning on the guidelines overlay changes the position or size of 0 elements, and release builds show it in 0 cases.
- **SC-014**: A developer new to the repository adds a simple atom with the component agent in under 1 hour. The result passes every repository check, and review requests no convention fixes.
- **SC-015**: 100% of releases have a changelog entry, and 100% of breaking changes have a migration note. 0 deprecated elements are removed before at least one minor release has passed.
- **SC-016**: 100% of components pass the accessibility rules in both modes, in the white-label brand and the sample brand. The rules are the tap-target size, labels, contrast, and no clipping at 200% text scale.

## Assumptions

- **Delivery in two releases.** Spacing, radius, and the other visual values become brand-driven, so screens that read spacing as fixed constants today must read it from the nearest brand. That is a breaking change for consumers. The move therefore ships first as an unchanged release (0.1.x), which keeps the Foundations spec's promise that the move changes no consumer. The multi-brand release follows as a breaking release (0.2.0), with migration notes.
- **Brand files carry primitive tokens only.** The Design System owns the mapping from primitive tokens to semantic roles for light and dark mode, so every brand gets the same accessible structure. Overriding semantic roles per brand is out of scope.
- **No automatic color correction.** Today the interim package silently adjusts the brand color to reach contrast. Brands now render exactly as designed, and the contrast gate replaces that adjustment.
- **Light and dark come from one brand file.** The format's companion resolver for multiple contexts is not needed, because the Design System derives both modes from one set of primitive tokens.
- **Brand files are obtained elsewhere.** Fetching, caching, storing, and choosing brand files, and deciding which brand a user sees, belong to the App Shell and the micro apps. They are out of scope.
- **Metadata location.** The brand's identifier, display name, and brand-schema version sit in the format's vendor-extension area, so design tools keep the file valid.
- **Fonts are supplied by the app**, or by the micro app for its own brand. The Design System bundles no fonts, and a missing font falls back to the platform font.
- **The 5% tolerance is a fixed user requirement.** Keeping each golden image to one component in one state is how this spec stops the tolerance from hiding real changes.
- **No CI service is assumed.** Tests must pass on a developer's Mac and on a Linux machine.
- **The component agent targets Claude Code**, the AI assistant this project uses. It is committed as an agent definition plus a repository conventions file.
- **The Widgetbook is for local use and is not hosted.** Text-scale and locale/RTL addons are out of scope. Behavior at 200% text scale stays covered by the accessibility tests.
- **Atomic levels**: the levels are atoms, molecules, organisms, and templates. None of the current components is an organism. The level exists for future components.
- **Test brands**: besides the white label, the repository holds at least one sample brand. It is used for golden images, the Widgetbook, and the contrast gate tests.
- **Relation to the Foundations spec.** The Foundations spec's delivery rules still apply: one pinned version, an unchanged re-export, and no dependency from the Design System on Foundations. This feature completes the interim package's planned move and replaces its single-color brand configuration.
