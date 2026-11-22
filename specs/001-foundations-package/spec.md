# Feature Specification: Foundations Package

**Feature Branch**: `main` (no feature branch created; spec directory `specs/001-foundations-package`)

**Created**: 2026-11-19

**Status**: Draft

**Input**: User description: "build the flutter foundation package based on the information of this page https://eugenio-tesio-porfolio.web.app/blog/modular-flutter-architecture?lng=en"

## Clarifications

### Session 2026-11-19

- Q: Which extra capabilities should Foundations define in its first release, given that the original list had no networking? → A: An authenticated API client, feature flags, and a logger, all as application-owned contracts.
- Q: Where should permission names be defined, given that one module may need to check another module's permission? → A: Permissions checked by more than one module live in Foundations. Permissions used by a single module stay in that module.
- Q: Should cross-module navigation return a result when the opened screen closes? → A: Yes. The caller receives either a map of primitive values (the same rule as inter-module data) or an explicit "cancelled" outcome.
- Q: Should Foundations offer global access points for its contracts? → A: No. Contracts are injected only, and the Core package provides the initialization that supplies them to hosts and micro apps.
- Q: What should a provider return when it refuses a call because the user isn't signed in or lacks a permission? → A: Keep the four statuses. The provider returns 400 with the shared error code `unauthenticated` or `forbidden`.

### Session 2026-11-20

- Q: Which state-management package must Foundations depend on? → A: `switchable_state` (pub.dev). Foundations depends on it and delivers its contract to micro apps unchanged. The host chooses the concrete library (adapter).

### Session 2026-11-21

- Q: What should the micro app contract take from the user's earlier `flutter_enterprise_architecture` package (whose "broker" is inter-module communication)? → A: Its main idea, without tenants. Micro apps declare their routes (each marked as needing sign-in or not), their data and UI endpoint providers, their localization resources, an initialization step, and a teardown step. The host gathers these and registers every provider through inter-module communication. Also adopted: an "is this endpoint registered?" check, an optional performance trace name on calls, and change notifications for feature flags.
- Q: Should the earlier package's stored-keys idea be kept? → A: Yes, as `storedKeys`. Each micro app declares the storage keys it owns so that no micro app can overwrite another's. Duplicates stop startup, and each micro app can access only its declared keys.
- Q: Should a performance-tracing capability be added? → A: No. The capability set stays as it was. The optional trace name on inter-module calls remains as a hint that Core records in its own diagnostics.
- Q: Should micro apps register their internal dependencies in a container? → A: No. Each micro app builds its own dependencies in its initialization step and accepts overrides through its constructor.
- Q: How should routes be defined without micro apps depending on a router package? → A: Foundations provides a router-neutral route contract (names, URL paths with parameters, child routes, route groups, presentation, and the sign-in requirement). Core uses it to build the actual router.
- Q: The `design-system` repository is still empty. How should the Design System story proceed? → A: For now, build an internal Design System package with a minimum set of components inside the Foundations repository. It moves to its own repository later.

## User Scenarios & Testing *(mandatory)*

The "users" of the Foundations package are the developers who build on it: micro app teams (consumers and providers of capabilities), the Core team (who implement its contracts), and host developers (the App Shell and each micro app's standalone example).

### User Story 1 - Request data from another micro app without importing it (Priority: P1)

A developer on one micro app (for example Transfers) needs data that another micro app (for example Accounts) owns. Using only Foundations, they describe the call (an endpoint address and its parameters) and receive a standard response that says whether the call succeeded and, if not, why. The developer of the providing micro app uses the same contract to publish a versioned endpoint, declare which parameters are mandatory, and supply a handler that returns the result in the agreed public format.

**Why this priority**: The architecture exists so that isolated features can collaborate through explicit contracts. Without this contract, features import each other's implementations and the boundaries collapse. Core, every micro app, and every host depend on it, so it must exist first.

**Independent Test**: Write a provider registration and a consumer call that depend only on Foundations. Run them against any implementation that passes the Foundations conformance check. Confirm that the consumer handles every outcome category without referencing the provider's code.

**Acceptance Scenarios**:

1. **Given** a provider registered `/module-b/v1/details` with `id` as a required parameter, **When** a consumer requests it with `id = "123"`, **Then** the consumer receives status 200 and data containing only primitive values, lists, and maps that match the published schema.
2. **Given** the same endpoint, **When** a consumer requests it without `id`, or with `id` set to null, **Then** the consumer receives status 400, error code `missing_required_parameters`, and no data, and the provider's handler is never invoked.
3. **Given** the same endpoint, **When** a consumer sends `id` as an empty string, **Then** the value reaches the provider, and the provider can reject it with status 400 and its own error code (for example `invalid_id`).
4. **Given** no provider has registered `/module-b/v1/details` in this application, **When** a consumer requests it, **Then** the consumer receives status 404 with error code `endpoint_not_registered`, so it can choose a fallback.
5. **Given** a provider whose handler fails unexpectedly, **When** a consumer requests the endpoint, **Then** the consumer receives status 500, and no internal failure details are exposed to it.
6. **Given** any response, **When** the consumer checks whether it succeeded, **Then** the check is true exactly when the status is between 200 and 299.
7. **Given** no user is signed in, **When** Transfers requests account balances from Accounts, **Then** Transfers receives status 400 with error code `unauthenticated`. **Given** a signed-in user who lacks the needed permission, **When** the same request is made, **Then** Transfers receives status 400 with error code `forbidden`.
8. **Given** an application that may or may not include Module B, **When** Module A checks whether `/module-b/v1/details` is registered as a data endpoint, **Then** it gets a yes or no answer without the provider being invoked, so it can hide an entry point for a feature that is absent.

---

### User Story 2 - Consume platform capabilities through application-owned abstractions (Priority: P2)

A micro app developer needs platform capabilities: the application version, calls to the backend, analytics events, crash reporting, logging, feature flags, storage, navigation, the signed-in user's session, and that user's permissions. They use contracts written in the application's own vocabulary and never learn which vendor SDK provides them. The host supplies real implementations, standalone examples can supply console or no-op ones, and tests supply fakes.

**Why this priority**: Every micro app needs these capabilities. Keeping vendors behind application-owned contracts means a vendor can be replaced in one adapter without touching any feature.

**Independent Test**: Build a sample feature that uses every abstraction. Run its tests with hand-written fakes for each contract, with no device, network, or Core present. Confirm that the sample feature declares no dependency on any vendor package.

**Acceptance Scenarios**:

1. **Given** a micro app that shows the application version, **When** it runs inside the App Shell or its standalone example, **Then** it gets the version through the application-information contract and depends on no vendor package to do so.
2. **Given** a micro app unit test, **When** the test supplies fakes for the capabilities the feature uses, **Then** the feature runs fully without a device, platform services, network, or Core.
3. **Given** the host replaces its analytics vendor with an implementation that honors the same contract, **When** the application is rebuilt, **Then** no micro app needs a code change.
4. **Given** a standalone example configured with no-op analytics and crash reporting, **When** the feature records events and errors, **Then** it behaves exactly as it does with real implementations, apart from the side effects that are now absent.
5. **Given** no user is signed in, **When** a micro app reads the session context, **Then** it receives an explicit "unauthenticated" state, not an absent or undefined value.
6. **Given** a signed-in user, **When** the user signs out, **Then** micro apps that observe the session context are notified of the change.
7. **Given** a micro app needs to open a screen that another module owns, **When** it asks to navigate to that screen's route name, **Then** navigation happens without importing the other module. **When** the route name is unknown, or a required path parameter is missing, **Then** the caller receives an explicit failure outcome and the application does not crash.
8. **Given** a signed-in user, **When** Accounts calls its backend through the API client contract using a path relative to the backend's base address, **Then** the request reaches the base address for the current environment with the user's credentials attached, and Accounts never sees or handles the credentials.
9. **Given** the user's session has expired, **When** a micro app makes a backend call, **Then** it receives an explicit "unauthorized" failure category, not a vendor-specific error.
10. **Given** a feature flag that the flag provider does not know, or a flag provider that is unavailable, **When** a micro app reads the flag with a default value, **Then** it receives that default.
11. **Given** a standalone example configured with console logging, **When** a feature writes log entries at different levels, **Then** the entries appear on the console, and with a no-op logger the feature behaves the same way.
12. **Given** Transfers opens the Accounts account picker by its public route, **When** the user selects an account, **Then** Transfers receives a result map of primitive values that matches the picker's published schema. **When** the user leaves without selecting, **Then** Transfers receives an explicit "cancelled" outcome.
13. **Given** a consumer that gives its inter-module call a trace name, **When** the call completes, **Then** the communication implementation records the call's duration and status under that name in its own diagnostics. A call without a trace name is not traced. Micro apps get no performance-tracing capability of their own.
14. **Given** a micro app that listens for feature flag changes, **When** the flag provider refreshes its values, **Then** the micro app is notified and re-reads its flags.

---

### User Story 3 - Integrate any micro app into any host the same way (Priority: P2)

A host developer, working on the App Shell or on a micro app's standalone example, integrates micro apps through one uniform micro app contract. Through that contract each micro app declares:
- its identity;
- the routes it exposes, each marked as needing sign-in or not;
- its localization resources;
- the data and UI endpoint providers it offers to other modules;
- an initialization step and a teardown step.

The host gathers these declarations and registers every endpoint provider through inter-module communication in one step. The same micro app plugs into the App Shell and into its own standalone example in exactly the same way.

**Why this priority**: The App Shell and the standalone examples must assemble micro apps without writing custom glue for each one. Micro apps can depend only on Foundations, so a shared integration contract can live only in Foundations.

**Independent Test**: Create two minimal micro apps that implement the contract and a minimal host that knows only the contract. Confirm that the host can collect every route and localization resource and run each initialization step, with no code specific to either micro app.

**Acceptance Scenarios**:

1. **Given** a micro app that implements the micro app contract, **When** a host is given that micro app, **Then** the host can list its identifier, routes, and localization resources and run its initialization step without code specific to that micro app.
2. **Given** a host about to run micro app initialization, **When** any micro app's initialization step runs, **Then** the inter-module communication contract is already available to it.
3. **Given** two micro apps that declare the same route name or the same full path, **When** a host assembles them, **Then** the host detects the conflict and stops startup, because every micro app exposes its routes in the same router-neutral form.
4. **Given** every micro app has finished its initialization step, **When** the host registers all declared endpoint providers, **Then** every endpoint is available before the app runs. **When** two micro apps declare the same address for the same type, **Then** startup stops with an error that names the address.
5. **Given** a route marked as needing sign-in, **When** someone who is not signed in tries to open it (through navigation or a deep link), **Then** the host sends them to sign-in instead of showing the screen.
6. **Given** a running host, **When** it is torn down (for example between tests, or when a standalone example resets), **Then** it runs each micro app's teardown step in reverse initialization order, and the micro apps release their subscriptions and resources.
7. **Given** a data endpoint provider, **When** a unit test calls it directly with a set of parameters, **Then** the test can check the provider's response without Core or any communication implementation.
8. **Given** two micro apps that both declare the storage key `session_cache`, **When** a host assembles them, **Then** startup stops with an error naming the key and both micro apps.
9. **Given** Accounts declared only `accounts.last_viewed`, **When** Accounts tries to read or write `transfers.draft`, **Then** the storage contract returns an explicit "undeclared key" failure, and Transfers' value is untouched.
10. **Given** Accounts declares a route named `accounts.details` with the path `/accounts/details/:accountId`, **When** the app is opened from the link `/accounts/details/42?tab=movements`, **Then** the route is shown with the parameters `accountId = "42"` and `tab = "movements"`. **When** another module opens `accounts.details` by name with the same parameters, **Then** it gets the same screen.
11. **Given** a micro app declares a route group (a shared tab layout) that contains three routes, **When** the user moves between those routes, **Then** the shared layout stays in place, and any state scoped to the group stays alive until the user leaves the group.

---

### User Story 4 - Use the unified Design System and state-management contract through Foundations (Priority: P2)

A micro app developer builds screens with the shared components, themes, typography, spacing, and design tokens. They manage screen and shared state through one state-management contract (`switchable_state`). They get both through Foundations, the same single dependency that gives them every other shared contract. That way every micro app and host uses one unified version of each. Foundations delivers both packages as they are and does not wrap, rename, or abstract any of their elements. The concrete state-management library (Cubit, GetX, MobX, Provider, or Riverpod) is chosen once by the host. Micro apps never depend on it, so changing it changes no micro app.

**Why this priority**: A shared visual language and a shared state model only work when every feature uses the same version of them. Delivering both through Foundations keeps one version across the product and leaves micro apps with a single shared dependency, as the architecture intends. Keeping the concrete state library out of micro apps means the product can change libraries in one place. The interim Design System lets micro apps build real screens now instead of waiting for the separate repository.

**Independent Test**: Create a sample micro app that depends only on Foundations. Build a screen with Design System components and tokens whose state lives in stores from the state-management contract. Confirm three things:
- The app builds without a direct dependency on the Design System, on `switchable_state`, or on any state-management library.
- The components it uses are the Design System's own.
- Its widget tests pass unchanged under two different adapters.

**Acceptance Scenarios**:

1. **Given** a micro app that depends only on Foundations, **When** its developer uses a Design System component, theme, or token, **Then** that element is available through Foundations with the same name and behavior it has in the Design System package.
2. **Given** the App Shell and two micro apps, **When** the product is assembled, **Then** all of them resolve the same Design System version: the one Foundations pins.
3. **Given** a new Design System version, **When** Foundations adopts it, **Then** the Foundations release states which Design System version it delivers. **When** that Design System change is breaking, **Then** the Foundations version also signals a breaking change.
4. **Given** a proposal to wrap or abstract a Design System component inside Foundations, **When** it is reviewed, **Then** it is rejected, because Design System elements are delivered unchanged.
5. **Given** a micro app that depends only on Foundations, **When** its developer declares app-wide state, state shared by a group of routes, or state owned by one screen, **Then** the state-management contract's elements are available through Foundations with the same names and behavior they have in `switchable_state`.
6. **Given** the App Shell and two micro apps, **When** the product is assembled, **Then** all of them resolve the same `switchable_state` version: the one Foundations pins.
7. **Given** a product running on one state-management library, **When** the host changes the adapter it passes at startup to another one (for example from Riverpod to Cubit), **Then** no micro app changes, and the micro apps' widget tests still pass.
8. **Given** a proposal to add a concrete state-management library or a `switchable_state` adapter package to Foundations, **When** it is reviewed, **Then** it is rejected, because the adapter is the host's choice.
9. **Given** the interim Design System, **When** a micro app builds a sign-in form, an accounts list, and an empty or error state, **Then** it can do so entirely with the minimum components (button, text field, card, list tile, inline message, empty state, loading indicator, page template) in both light and dark themes.
10. **Given** the brand configuration changes its primary color, **When** the app is rebuilt, **Then** every component that uses the primary-action role shows the new color, with no change to any component or micro app.
11. **Given** a user who has enlarged text to 200% and uses a screen reader, **When** they use any Design System component, **Then** text is not clipped, every tappable element has a label, and every tap target is at least 48 × 48 logical pixels.
12. **Given** the Design System package is later moved to its own repository, **When** Foundations switches its dependency to that repository's release, **Then** no micro app changes. They keep importing Foundations and using the same names.

---

### User Story 5 - Embed UI that another micro app owns (Priority: P3)

A developer building a federated Home wants to show a banner that the Cards team owns and maintains. Cards registers a UI factory at `/cards/v1/home-banner` that needs a `cardId`. Home asks for that UI through Foundations, then decides where to place it and what to show if it is unavailable. Home never imports the banner's concrete component.

**Why this priority**: Cross-team UI composition is valuable but builds on the same contract as Story 1. A first release can ship with data requests alone.

**Independent Test**: Register a UI factory and request it from a separate consumer that depends only on Foundations. Confirm the success and failure outcomes, including the case where the consumer falls back when the provider is absent.

**Acceptance Scenarios**:

1. **Given** Cards registered the banner factory with `cardId` as a required parameter, **When** Home requests it with `cardId = "card-123"`, **Then** Home receives status 200 with a UI element it can place. The element is returned immediately, without waiting for the banner's own data to load.
2. **Given** the same registration, **When** Home requests the banner without `cardId`, **Then** Home receives status 400 with `missing_required_parameters` and no UI element.
3. **Given** the same registration, **When** Home sends `cardId` as an empty string, **Then** Cards' factory can reject it with status 400 and `invalid_card_id`.
4. **Given** Cards is not part of this application, **When** Home requests the banner, **Then** Home receives status 404 with `endpoint_not_registered` and can omit the banner or show its own fallback.
5. **Given** a data endpoint and a UI endpoint registered at the same address, **When** both are used, **Then** they do not conflict, because data registrations and UI registrations are kept separate.

---

### User Story 6 - Evolve the shared contracts safely (Priority: P3)

A Foundations maintainer changes or extends the package, knowing that every micro app, Core, and every host depends on it. The public surface must stay small and deliberate. Every change must be clearly classified as compatible or breaking, and feature-specific models must stay in their own modules.

**Why this priority**: These guardrails protect every other story over time. They matter most after the first release.

**Independent Test**: Review a release against the package's versioning, documentation, and inclusion rules. Confirm that a breaking change is labeled as one and comes with migration notes, and that a proposed feature-specific model is rejected.

**Acceptance Scenarios**:

1. **Given** a change that removes or alters a public contract, **When** it is released, **Then** the version number signals a breaking change, and the changelog lists the migration steps.
2. **Given** a new public element, **When** it is reviewed, **Then** it has reference documentation that describes its purpose, its usage, and its possible outcomes.
3. **Given** a proposal to add a feature-specific model (for example, an Accounts response), **When** it is reviewed against the inclusion rules, **Then** it is rejected, and the model stays in its owning micro app.
4. **Given** a permission that only Transfers checks, **When** Home also needs to check it, **Then** the permission moves into Foundations, and both modules reference the shared definition.

---

### Edge Cases

- **Null vs. empty required parameter**: a required parameter that is present but null counts as missing (400, `missing_required_parameters`). An empty string counts as present and goes to the provider for value validation.
- **Consumer tries to relax validation**: a consumer's call cannot override the provider's required-parameter declaration. Validation always uses what the provider declared when it registered.
- **Extra parameters**: parameters that the provider did not declare are passed to the provider unchanged. They never cause a dispatch failure.
- **Duplicate registration**: registering the same address twice for the same type (data or UI) is rejected with an error that names the address, so a host can fail fast during startup. The same address may be registered once as a data endpoint and once as a UI endpoint.
- **Contract not supplied**: there are no global access points, so a micro app can only use the contracts it is given. A contract that the micro app requires but the host did not supply is reported as a clear developer-facing error when the host initializes the micro app, not later when a feature first uses it.
- **Unexpected status values**: consumers treat any status outside 200–299 as a failure, including statuses the contract does not list.
- **Payload contains provider-owned objects**: data payloads are limited to primitive values, lists, and maps. Returning any other kind of value violates the contract, and the documentation says so.
- **Errors after a UI element is returned**: failures while the returned UI element renders or loads its data belong to the module that provides it, not to the communication contract.
- **Incompatible endpoint change**: an incompatible change is published under a new versioned address (for example `/cards/v2/home-banner`) while the earlier version keeps serving existing consumers.
- **No-op capability implementations**: calling analytics or crash reporting when the host configured a no-op implementation succeeds and has no observable effect.
- **Storage failures**: when a storage read or write fails, the caller receives an explicit failure outcome, not a vendor-specific error.
- **Backend call failures**: the API client reports each failure as one explicit category (no connectivity, timeout, unauthorized, client error, server error), so a micro app can react without inspecting vendor errors.
- **Feature flag of the wrong kind**: reading a flag as a different kind of value than the provider holds (for example as a number when it is on/off) returns the caller's default.
- **Micro app depends on the Design System directly**: micro apps take the Design System only through Foundations. A direct dependency that pins a different version breaks the single-version guarantee and must be caught when dependencies are resolved, not discovered at runtime.
- **Dependency cycle**: Foundations depends on the Design System, so the Design System must never depend on Foundations. This holds even while the Design System lives inside the Foundations repository.
- **Consumers that pull Foundations from git**: the interim Design System is resolved from the same Foundations commit, so a consumer can't end up with a mismatched Design System version.
- **Fonts**: the Design System bundles no fonts. When the brand configuration names a font family that the app doesn't provide, text falls back to the platform font instead of failing.
- **Micro app uses a state library directly**: a micro app that imports a concrete state-management library (or `switchable_state` itself) bypasses the host's adapter choice. That breaks the "change one line at startup" guarantee, and the dependency must be caught when dependencies are resolved or reviewed.
- **Adapter incompatible with the pinned version**: a host's adapter package must accept the `switchable_state` version that Foundations pins. A mismatch fails during dependency resolution, not at runtime.
- **Widget tests need an adapter**: the contract needs an adapter to run. A micro app's widget tests may add any adapter as a development-only dependency, and that never becomes a runtime dependency.
- **Library-specific timing**: some libraries deliver changes a frame later (for example Cubit), so micro app widget tests must let the UI settle after a write instead of assuming a synchronous rebuild.
- **Requests during initialization**: endpoint providers are registered only after every micro app has initialized, so a request made from inside an initialization step may get a 404 (`endpoint_not_registered`). Micro apps must not depend on other modules' endpoints while they initialize.
- **Deep link to a guarded route while signed out**: the host redirects to sign-in. Whether the user is returned to the original route after signing in is the host's decision.
- **Trace name on a call to an unregistered endpoint**: the call is still traced, and its outcome (404) is recorded with it.
- **Teardown failure**: if one micro app's teardown step fails, the host still runs the remaining teardown steps and reports the failure.
- **Legacy storage keys**: a key does not have to follow the `<microAppId>.<key>` convention. A key inherited from an older app version (for example `auth_token`) can be declared as it is. Declaring it is what reserves it.
- **Same key in general and secure storage**: a declared key belongs to its micro app in both stores, so another micro app can't claim it in the other store.
- **Deep link missing a path parameter**: a link that matches no route, or that lacks a required path parameter, shows the host's "not found" screen. Opening a route by name without a required path parameter returns an explicit "parameters missing" outcome.
- **Child of a route that needs sign-in**: a child route needs sign-in if it or any of its ancestors does.

## Requirements *(mandatory)*

### Functional Requirements

#### Package boundary

- **FR-001**: Foundations MUST be a single package that defines three areas of its own, (a) capability abstractions, (b) the inter-module communication contract, and (c) the micro app integration contract. It also delivers two first-party packages: the Design System (see FR-002) and the state-management contract (see FR-042).
- **FR-002**: Foundations MUST deliver the Design System (components, themes, typography, spacing, and design tokens) by re-exporting the public surface of the separate Design System package unchanged.
  - The Design System MUST be its own package, with its own manifest and tests.
  - Until it moves to its own repository, it lives *inside the Foundations repository* as that separate package (see FR-054 and FR-056).
  - Foundations' own code MUST NOT contain any Design System implementation, and MUST NOT wrap, rename, or abstract Design System elements.
- **FR-003**: Foundations MUST NOT depend on Core, the App Shell, any micro app, any third-party vendor SDK, or any concrete state-management library. Its runtime dependencies are limited to the UI framework SDK, that SDK's standard libraries, the Design System package, and `switchable_state`.
- **FR-004**: Foundations' public surface MUST NOT expose any vendor-specific type, concept, or configuration. The re-exported Design System and `switchable_state` are first-party packages, not vendor SDKs.
- **FR-005**: Foundations MUST NOT contain feature-specific business models (for example account details or card data) or any concrete implementation of a capability.
- **FR-006**: Foundations MUST expose its own contracts and the two re-exported packages through deliberate public entry points, and its internal helpers MUST NOT be reachable by consumers.

#### Inter-module communication contract

- **FR-007**: Foundations MUST define an **endpoint description** made of an address, call parameters (each a name and a value), and the set of required parameter names that the provider declares when it registers. By convention the address includes the owning module and a version: `/<module>/v<N>/<name>`.
- **FR-008**: Foundations MUST define a **response** that carries a status, optional result data, and an optional machine-readable error code. It MUST include a success check that is true exactly when the status is between 200 and 299.
- **FR-009**: Foundations MUST define two endpoint types. **Data endpoints** answer asynchronously with a map of primitive values, lists, and maps. **UI endpoints** answer synchronously with a ready-to-place UI element.
- **FR-010**: Foundations MUST define five operations: register a data handler, request data, register a UI factory, get a UI element, and check whether an address is registered for a given kind (data or UI). The check never invokes a provider.
- **FR-011**: Foundations MUST define and document the standard outcome meanings, which are application statuses and not network responses:
  - **200**: success. The data holds the result map or the UI element.
  - **400**: a required parameter is missing or null (`missing_required_parameters`, detected before the provider is called), or the provider rejects a value with its own error code, or the provider refuses access because the user is not signed in (`unauthenticated`) or lacks a permission (`forbidden`).
  - **404**: the endpoint is not registered (`endpoint_not_registered`), or the provider reports that the requested item does not exist. The error code tells the two cases apart.
  - **500**: the handler or factory failed unexpectedly. No internal details reach the consumer.
- **FR-012**: Foundations MUST publish the standard status values and the shared error codes (`missing_required_parameters`, `endpoint_not_registered`, `unauthenticated`, `forbidden`) as part of the contract. The contract has exactly four statuses (200, 400, 404, 500), and no others are added for access refusals. Providers, consumers, and implementations MUST all reference these same published values.
- **FR-013**: The contract MUST specify that required-parameter checks use the provider's registered declaration and never the consumer's. It MUST also specify that a null value counts as missing, while an empty string is passed to the provider for validation.
- **FR-014**: The contract MUST specify that data registrations and UI registrations are kept separate, and that registering the same address twice within one type is rejected with an error that names the address.
- **FR-015**: The contract MUST specify that a UI endpoint returns its UI element without waiting for that element's data, and that loading and error states after the element is returned belong to the providing module.
- **FR-016**: Foundations MUST NOT offer a global access point for any of its contracts and MUST NOT hold global state. Consumers receive every contract by injection. The initialization that supplies contracts to hosts and micro apps belongs to the Core package.
- **FR-017**: Foundations MUST provide a reusable conformance check. Any implementation of the communication contract, whether Core or a stub in a standalone example, can run it to prove that it produces the outcomes required by FR-011, FR-013, and FR-014, and that the registration check required by FR-010 is correct.
- **FR-018**: The contract documentation MUST define the endpoint versioning convention: an incompatible change is published under a new versioned address, and the previous version stays available until its consumers have migrated.

#### Capability abstractions

- **FR-019**: Foundations MUST define an **application information** contract that exposes the application name, version, build number, and runtime environment (development, staging, or production).
- **FR-020**: Foundations MUST define an **analytics** contract that records screen views and named events with properties.
- **FR-021**: Foundations MUST define a **crash and error reporting** contract that records non-fatal errors with context and associates reports with an anonymous or pseudonymous user identifier.
- **FR-022**: Foundations MUST define a **key-value storage** contract with a general-purpose variant and a separate secure variant for sensitive values. A micro app MUST be able to read and write only the keys it declares (FR-052).
- **FR-023**: Foundations MUST define a **navigation** contract with three operations, none of which imports the module that owns the route:
  - **Open**: open a route by its name with text parameters, on top of the current screen. This MUST be able to return a result when the opened screen closes: either a map of primitive values, lists, and maps that follows the route's published schema, or an explicit "cancelled" outcome when the screen closes without a result.
  - **Go**: go to a route by its name, replacing the current location.
  - **Close**: close the current screen.

  An unknown route name MUST produce an explicit "not found" outcome. A missing path parameter MUST produce an explicit "parameters missing" outcome.
- **FR-024**: Foundations MUST define a **session context** contract that exposes the current authentication state as explicit states (authenticated with a user identity summary, or unauthenticated) and lets consumers observe changes to that state.
- **FR-025**: Foundations MUST define a **permissions** contract that answers whether the current user holds a named permission. Permissions MUST be explicit, named values, not free-form text. Foundations MUST define the permissions that more than one module checks. A permission used by only one module MUST be defined in that module, through the same contract. When a module-owned permission starts being checked by another module, it MUST move into Foundations.
- **FR-039**: Foundations MUST define an **API client** contract that lets a micro app call backend services using paths relative to the base address of the current environment. Implementations attach the current session's credentials automatically, and micro apps MUST never receive or handle credentials. Every failure MUST be reported as exactly one category: no connectivity, timeout, unauthorized, client error, or server error.
- **FR-040**: Foundations MUST define a **feature flags** contract that reads named flags as on/off, text, or number values. Every read takes a caller-supplied default, which is returned when the flag is unknown, has a different kind of value, or the flag provider is unavailable.
- **FR-041**: Foundations MUST define a **logging** contract that writes log entries at the levels debug, info, warning, and error, with optional context. Its documentation MUST state that log entries never contain credentials or personal data.
- **FR-050**: A consumer MAY give an inter-module call an optional **trace name**. Implementations of inter-module communication MUST time each traced call and record its name, duration, and status in their own diagnostics. Foundations MUST NOT add a performance-tracing capability: the capability set stays as listed in FR-019 through FR-025 and FR-039 through FR-041.
- **FR-051**: The **feature flags** contract MUST notify consumers whenever flag values are refreshed, so micro apps can react to remote changes without restarting.
- **FR-026**: The analytics, crash reporting, and logging contracts MUST allow no-op and console implementations without any change in consumer behavior. The feature flags contract MUST allow an implementation that always returns the caller's defaults.
- **FR-027**: Every capability abstraction MUST be replaceable by a test double in consumer tests, without a device, platform services, network, or Core.
- **FR-028**: Capability operations that can fail (such as backend calls, storage reads and writes, or navigation to an unknown route) MUST express that failure explicitly in the contract and MUST NOT surface vendor-specific errors.

#### Micro app integration contract

- **FR-029**: Foundations MUST define a uniform **micro app contract** through which each micro app declares:
  - a unique identifier;
  - the storage keys it owns;
  - the routes it exposes, as declared by the route contract (FR-053);
  - its localization resources;
  - its data endpoint providers and its UI endpoint providers;
  - an initialization step and a teardown step.

  The initialization step receives the Foundations contracts that Core's initialization supplies for that micro app, so the micro app can build its internal dependencies and make the contracts available to its own UI.
- **FR-030**: The micro app contract MUST let the same micro app be integrated in exactly the same way by the App Shell and by its standalone example.
- **FR-031**: The micro app contract MUST document the startup order:
  1. Identifiers and storage keys are read and checked for duplicates.
  2. Contracts are supplied to every initialization step, with storage scoped to that micro app's keys.
  3. After all initialization steps finish, routes are checked and every declared endpoint provider is registered, before the app runs.
  4. Teardown steps run in reverse initialization order.
- **FR-046**: Foundations MUST define **endpoint providers**: a data provider declares its endpoint (address and required parameters) and answers requests, and a UI provider declares its endpoint and builds UI elements. A provider MUST be usable directly, without Core or any communication implementation, so its behavior can be unit tested.
- **FR-047**: Foundations MUST offer one operation that registers a set of data and UI endpoint providers through inter-module communication. Hosts use it to register every provider that the micro apps declare. Micro apps MUST NOT register endpoints any other way.
- **FR-048**: Each route MUST state whether it needs sign-in, and the default MUST be that it does. The contract MUST document that hosts do not show a route that needs sign-in while the session is unauthenticated.
- **FR-049**: The teardown step MUST let a micro app release subscriptions and resources it acquired during initialization. Hosts call it when they are torn down.
- **FR-052**: Each micro app MUST declare every storage key it uses, in either general or secure storage, as its **stored keys**.
  - A key MAY be declared by only one micro app. Hosts MUST stop startup when two micro apps declare the same key, naming the key and both micro apps.
  - The storage contracts given to a micro app MUST be limited to its declared keys. Any read, write, delete, or existence check of an undeclared key MUST fail with an explicit "undeclared key" failure and leave other micro apps' values untouched.
- **FR-053**: Foundations MUST define a router-neutral **route contract** that Core uses to build the application's router, so micro apps never depend on a router package. Each route declares:
  - a unique name (`<microAppId>.<screen>`), used for navigation;
  - a URL path pattern that may contain named path parameters. Top-level paths start with `/<microAppId>`, and child paths are relative to their parent.
  - a builder that receives the route's text parameters (path and query);
  - whether it needs sign-in (FR-048);
  - its presentation: a normal page or a full-screen dialog;
  - its child routes.

  A **route group** wraps a set of routes in a shared layout (for example tabs) that stays in place while the user moves between them, and it may hold state scoped to the group. All route parameters are text, so every route can be opened from a deep link. Hosts MUST stop startup when two routes share a name or a full path.

#### Quality and evolution

- **FR-032**: Every public element MUST have reference documentation that describes its purpose, its usage, and its possible outcomes.
- **FR-033**: Foundations MUST follow semantic versioning and keep a changelog. A breaking change MUST be marked as such and come with migration notes. A deprecation MUST precede the removal of a public element by at least one minor release.
- **FR-034**: Foundations MUST include automated tests for every behavior it defines itself, including the response success rule, the endpoint description semantics, and the conformance check.
- **FR-035**: Foundations MUST include a runnable usage example covering a data endpoint (register and request) and a UI endpoint (register and get), each showing every standard outcome.

#### Design System delivery

- **FR-036**: Foundations MUST pin exactly one Design System version, so every micro app and host that depends on Foundations resolves that same version.
- **FR-037**: Micro apps MUST be able to use the whole public Design System through Foundations alone, with no direct dependency on the Design System package.
- **FR-038**: Each Foundations release MUST state the Design System version it delivers. When Foundations adopts a breaking Design System change, that Foundations release MUST be marked as a breaking change.
- **FR-054**: Until the Design System moves to its own repository, an **interim Design System package** MUST provide at least the following:
  - **Primitive tokens**: a color palette (neutral, brand, success, warning, and error scales), a spacing scale, and a corner-radius scale.
  - **Semantic tokens**: color roles that express intent, such as background, surface, text, primary action, disabled, focus, and the information, success, warning, and error backgrounds and foregrounds.
  - **A typography scale**: display, headline, title, body, label, and caption styles.
  - **Light and dark themes**: both built from a single **brand configuration** (primary color and, optionally, font family), so a brand change happens in one place.
  - **Components**, grouped as atoms, molecules, and templates:
    - a button with primary, secondary, and tertiary variants and loading and disabled states;
    - a text field with label, helper text, and error text;
    - a loading indicator;
    - a card;
    - a list row with leading, title, subtitle, and trailing slots;
    - an inline message in information, success, warning, and error tones;
    - an empty or error state with an optional action;
    - a page template with a title, body, and optional primary action.
- **FR-055**: Every interim Design System component MUST meet these accessibility rules in both themes:
  - Tap targets are at least 48 × 48 logical pixels.
  - Body text contrast is at least 4.5:1, and large text and interface contrast is at least 3:1.
  - Every interactive element has an accessible label.
  - Text is not clipped or overflowing at 200% text scale.
- **FR-056**: The interim Design System package MUST be movable to its own repository without changing its package name, its public library, or any consumer code.
  - Its only runtime dependency is the UI framework SDK.
  - It MUST NOT reference Foundations or any other package of the product.
  - Moving it MUST require changing only the dependency source in Foundations' manifest.

#### State-management delivery

- **FR-042**: Foundations MUST deliver the shared state-management contract by re-exporting the public surface of the `switchable_state` package unchanged, at exactly one pinned version. It MUST NOT wrap, rename, or abstract any `switchable_state` element.
- **FR-043**: Foundations MUST NOT depend on any `switchable_state` adapter package (Cubit, GetX, MobX, Provider, Riverpod) or on any concrete state-management library. Choosing the adapter belongs to the host.
- **FR-044**: Micro apps MUST be able to declare and use app-wide, route-group, and screen-owned state through Foundations alone, with no direct runtime dependency on `switchable_state`, on an adapter package, or on a concrete state-management library. That way the host's adapter choice applies to every micro app.
- **FR-045**: Each Foundations release MUST state the `switchable_state` version it delivers. When Foundations adopts a breaking `switchable_state` change, that Foundations release MUST be marked as a breaking change.

### Key Entities

- **Inter-Module Endpoint**: Describes one call. It holds an address used as the in-process registry key, the call parameters (each a name and a primitive value, list, or map), and the set of required parameter names that the provider declares when it registers.
- **Inter-Module Response**: The result of any inter-module call. It holds a status, optional data (a result map or a UI element), and an optional error code, plus a success check.
- **Data Handler**: The provider-supplied behavior that answers a data endpoint asynchronously with an Inter-Module Response.
- **UI Factory**: The provider-supplied behavior that answers a UI endpoint synchronously with an Inter-Module Response that carries a UI element.
- **Inter-Module Communication**: The contract with five operations: register a data handler, request data, register a UI factory, get a UI element, and check whether an address is registered. Core implements it, and hosts make it available to micro apps. It plays the role of the "broker" in the earlier `flutter_enterprise_architecture` package.
- **Endpoint Provider**: A class that a micro app declares to offer one data endpoint (it answers requests) or one UI endpoint (it builds UI elements), together with its address and required parameters. Hosts register providers in bulk.- **Outcome Status and Error Code**: The published vocabulary of statuses (200, 400, 404, 500) and the shared error codes that give every outcome a single meaning.
- **Application Information**: The application's name, version, build number, and runtime environment.
- **Analytics Event**: A named event or screen view with optional properties.
- **Error Report**: A non-fatal error with its context and an optional pseudonymous user identifier.
- **Storage Entry**: A value stored under a key, in either general-purpose or secure storage.
- **API Request and Result**: A backend call described by a method, a relative path, and an optional body and query. Its result is either a successful response (status and body) or exactly one failure category.
- **Feature Flag**: A named value (on/off, text, or number) that is read with a caller-supplied default.
- **Log Entry**: A message with a level (debug, info, warning, or error) and optional context, containing no credentials or personal data.
- **Route**: A screen exposed by a micro app. It has a unique name used for navigation and a URL path pattern used for deep links, and it receives text parameters. It states whether it needs sign-in (by default, it does) and how it is presented (page or full-screen dialog). It may have child routes and may publish a result schema: when it closes, it returns either a result map or "cancelled".
- **Route Group**: A set of routes that share a surrounding layout (for example tabs) and, optionally, state scoped to the group.
- **Stored Key**: A storage key declared by exactly one micro app. Only that micro app can read or write it, in general or secure storage.
- **Session Context**: The current authentication state (authenticated with a user identity summary, or unauthenticated), which consumers can observe as it changes.
- **Permission**: A named entitlement that the current user either holds or does not. Shared permissions are defined in Foundations. A permission used by only one module is defined in that module.
- **Micro App**: The integration descriptor for one micro app: its identifier, stored keys, routes and route groups, localization resources, data and UI endpoint providers, initialization step, and teardown step.
- **Design System (delivered)**: The separate first-party package of components, themes, typography, spacing, and design tokens. Foundations re-exports it unchanged at one pinned version. For now it lives inside the Foundations repository as its own package, and it will move to its own repository later.
- **Design Token**: A named design value. *Primitive* tokens hold base values (a palette color, a spacing step). *Semantic* tokens express intent (the primary action color, the error background), and components use only semantic tokens.
- **Brand Configuration**: The single place that sets the primary color and, optionally, the font family. The light and dark themes are both derived from it.
- **State-Management Contract (delivered)**: The first-party `switchable_state` package. It provides one contract for app-wide, route-group, and screen-owned state, and runs on whichever adapter the host chooses. Foundations re-exports it unchanged at one pinned version.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A dependency audit of Foundations finds 0 dependencies on Core, the App Shell, any micro app, any vendor SDK, any `switchable_state` adapter, or any concrete state-management library. The Design System and `switchable_state` are its only first-party dependencies, and its public surface contains 0 vendor-specific types.
- **SC-002**: A sample micro app that uses every contract in Foundations and builds a screen with the Design System and the state-management contract compiles, and its tests pass, with Foundations as its only internal architecture dependency: no Core, no other micro app, and no direct Design System, `switchable_state`, or state-library dependency.
- **SC-003**: The conformance check covers 100% of the documented outcome scenarios: success, missing parameter, null parameter, invalid value, endpoint not registered, item not found, unexpected failure, duplicate registration, and the registration check. A correct implementation passes 100% of it.
- **SC-004**: A developer new to the package can publish a data endpoint in one micro app and consume it from another in under 30 minutes, using only the package documentation and its example.
- **SC-005**: Replacing the implementation behind any single capability (for example, switching analytics vendors) requires changes in 0 micro apps.
- **SC-006**: 100% of public elements have reference documentation, and static analysis of the package reports 0 issues.
- **SC-007**: 100% of micro app tests that use test doubles for Foundations contracts run in a headless continuous-integration environment, with no device, network, or Core.
- **SC-008**: In 100% of failure responses, consumers can tell "module absent", "item not found", "invalid input", and "access refused" apart using only the status and the error code.
- **SC-009**: In the assembled product, 100% of micro apps and hosts resolve the same Design System version.
- **SC-010**: 100% of the Design System's public elements are available through Foundations, with the same names and behavior as in the Design System package.
- **SC-011**: 0 micro apps handle credentials or backend base addresses directly. 100% of their backend calls go through the API client contract.
- **SC-012**: Foundations contains 0 global access points and 0 global state, so micro app tests that use test doubles can run in parallel without affecting each other.
- **SC-013**: Changing the host's state-management adapter needs a change in 0 micro apps. The sample micro app's widget tests pass under at least two different adapters, with no code changes.
- **SC-014**: 100% of the sample micro app's endpoint providers are covered by unit tests that call them directly, without Core or any communication implementation.
- **SC-015**: Changing Core's router implementation needs a change in 0 micro apps. 100% of declared routes can be opened both by name and from a deep link to their path.
- **SC-016**: In the assembled product, 0 storage keys are declared by more than one micro app. 100% of attempts to access an undeclared key fail with the "undeclared key" outcome.
- **SC-017**: 100% of the interim Design System's components pass the tap-target, contrast, and labeling checks in both themes and render without overflow at 200% text scale. Changing the brand's primary color in one place changes 100% of the components that use the primary-action role.
- **SC-018**: Moving the Design System to its own repository changes exactly one dependency entry in Foundations' manifest and 0 lines in any micro app.

## Assumptions

- **Source**: The requirements come from the article "Modular Flutter Architecture" (published November 12, 2026). Its code samples define the communication contract's shape: endpoint, response, handler, UI factory, and the article's four operations. The fifth operation, the registration check, comes from the earlier `flutter_enterprise_architecture` package.
- **Design System delivery**: The article places the Design System inside Foundations. In this project the Design System belongs in its own repository (`design-system`), as the user confirmed. Foundations depends on it and re-exports it unchanged, with no abstraction layer, so that micro apps receive one unified Design System version through their single Foundations dependency. This keeps the article's intent that micro apps depend only on Foundations.
  - The `design-system` repository is still empty, so the user asked for an **interim** Design System package with a minimum set of components (FR-054). It lives inside the Foundations repository as its own package and will later move to its own repository.
  - Building the minimum set is in scope for this feature. Growing it into a complete system is not, and belongs to the Design System's own feature once it has moved.
  - The Design System must never depend on Foundations.
- **State-management delivery**: The user requires Foundations to depend on `switchable_state`, their own package, published on pub.dev. Version 0.1.0 is the only release so far. Foundations re-exports it unchanged, the way it re-exports the Design System. Adapter packages (`switchable_state_bloc`, `_getx`, `_mobx`, `_provider`, `_riverpod`) wrap vendor libraries, so they belong to the host. Core's initialization configuration chooses one, and the host places the root scope. Because the micro apps share this one contract, the article's "each team chooses its state management" now applies only to how a micro app structures its state inside that contract.
- **Location**: The package is built in the existing `foundations` repository (a submodule of this container; it currently contains only an empty README).
- **Platform**: The package targets the current stable Flutter SDK. UI endpoints return UI elements, so the package cannot be platform-independent.
- **Core owns the implementation**: The following all belong to the Core package and are out of scope here:
  - registries, dispatch, and required-parameter checks;
  - converting unexpected failures into status 500, and timing traced calls;
  - internal diagnostics and vendor adapters;
  - the configurable bootstrap and the initialization that supplies contracts to hosts and micro apps;
  - the reusable host assembly that initializes micro apps, collects their routes, localization resources, and providers, registers the providers, and checks for duplicates. This is the counterpart of `AppShellContainer` in `flutter_enterprise_architecture`. Foundations only defines the contracts and the conformance check that those implementations must pass.
- **Prior art**: The micro app contract adapts the user's earlier `flutter_enterprise_architecture` package. That package's broker becomes inter-module communication, its microapp container becomes the micro app contract, its brokers become endpoint providers, and its stored-keys idea becomes `storedKeys`, which keeps micro apps from overwriting each other's keys. Its routes, which were typed by the router package, become a router-neutral route contract that Core implements. These parts were left out:
  - tenants, tenant lifecycle, and the multitenant broker, at the user's request;
  - the global broker instance and the global service-locator registration of dependencies, because they conflict with the no-globals clarification. Micro apps take overrides through their own constructors instead.
  - the performance trace logger as a capability; only the trace name on calls remains, as a hint for Core;
  - callback-based calls and text-only parameters, because futures and structured payloads already cover them;
  - provider-to-consumer streaming through an observer (deferred, because it would add a third call style beyond the article's two);
  - product-specific action parsers.
- **Initial capability set**: The capability set covers what the current micro apps (Home, Accounts, Transfers) need: application information, an authenticated API client, analytics, crash and error reporting, logging, feature flags, storage, navigation, session context, and permissions. Camera and other device capabilities are deferred until a micro app needs them.
- **Meaning of "permissions"**: "Permissions" means what the signed-in user is entitled to do. It does not mean operating-system device permissions.
- **Test doubles**: Except for the communication conformance check, Foundations does not ship ready-made fakes for every abstraction. Consumers write their own test doubles against the contracts.
- **In-process communication**: All inter-module communication runs inside one application process. There are no network calls, and no UI code is downloaded at runtime.
- **Selective boundaries**: Foundations does not wrap general UI framework components and does not reproduce the API of any state-management library. State management is covered by re-exporting `switchable_state`, not by abstractions of its own. Foundations abstracts only capabilities where a boundary protects something meaningful.
- **Shared dependency resolution**: Resolving dependencies across the workspace (one lock file for the shell and the examples) and coordinating native builds are concerns for the planning phase and for the hosts, not requirements on this package.
- **Constitution**: The project constitution has not been ratified yet (`.specify/memory/constitution.md` is still the template), so this spec applies no extra governance constraints.
