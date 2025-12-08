# SMHS Schedule Product, Architecture, and Interview Dossier

This single document consolidates the Product Requirements Document, technical case studies, and STAR-formatted interview scenarios for the SMHS Schedule app.

---
## Part 1: Product Requirements Document (PRD)

# SMHS Schedule App Product Requirements Document (PRD)

## 1. Product Overview
- **Product name:** SMHS Schedule
- **Platforms:** iOS (iPhone/iPad) and macOS
- **Summary:** A SwiftUI-based companion app for students to view their daily schedule, track grades, read school announcements, and search school resources. The app surfaces upcoming events, supports offline awareness via cached schedules, and integrates with the school’s existing systems (AppServ XML API and ICS calendars).
- **Primary value:** Deliver a reliable, student-friendly hub that reduces friction in checking class times, grades, and news while adapting to changing bell schedules.
- **Technical posture:** Shared SwiftUI codebase with lightweight platform-specific wrappers; data flows through observable objects (`SharedScheduleInformation`, `UserSettings`) that coordinate network fetches (Alamofire for AppServ, URLSession for ICS), persistence, and view state. Networking is abstracted through an `Endpoint` builder to centralize URL construction, headers, and retry policies.

## 2. Goals and Non-Goals
### Goals
1. Provide an accurate “Today” view that reflects the current bell schedule, including special cases (assemblies, minimum days, etc.).
2. Offer timely access to grade summaries and announcements without requiring repeated logins.
3. Support cross-platform parity (iOS and macOS) with consistent navigation, theming, and behaviors.
4. Ensure schedules update automatically from primary (AppServ) and fallback (ICS) sources, with clear freshness metadata in the UI.
5. Respect student privacy and minimize data retention while remaining observable (analytics, logging) for reliability.

### Non-Goals
- Acting as a homework/task manager.
- Replacing official SIS grade entry workflows for teachers.
- Managing attendance submissions or disciplinary records.
- Providing parent/guardian account features in this release.

## 3. Target Users and Personas
- **Current SMHS students:** Primary users who check daily schedules, announcements, and grades.
- **Students with variable schedules:** Benefit from accurate handling of custom bell schedules (e.g., assembly days).
- **Power users (student leaders):** Rely on quick news access and search to find resources.

## 4. User Stories
1. As a student, I want to open the app and immediately see today’s classes with start/end times so I can get to the right room on time.
2. As a student, I want the schedule to refresh automatically each morning and when I pull to refresh so I can trust the data without manual upkeep.
3. As a student, I want to view my latest grades quickly without re-entering credentials so I can monitor my performance.
4. As a student, I want to read school announcements in one place so I do not miss important updates.
5. As a student, I want offline awareness (e.g., last updated timestamp and cached schedule) so I can plan even with poor connectivity.
6. As a student, I want consistent navigation and settings across iOS and macOS so the app feels familiar.

## 5. Scope and Features
### In Scope
- Today view: daily schedule, progress indicator for current period, next-period preview, and last-updated metadata.
- Grades view: grade summary per class, detail drill-in, and authentication persistence (secure token storage).
- Schedule view: multi-day calendar with ability to browse future days and identify special bell schedules.
- News view: list of announcements with detail pages and deep links to external resources.
- Search: unified search across classes, announcements, and resources.
- Settings (debug accessible in development builds): refresh controls, cache reset, and diagnostics.
- Cross-platform support with SwiftUI shared code and platform-specific scene/lifecycle handling.

### Out of Scope (This Release)
- Push notifications for schedule changes or announcements.
- Parent/guardian accounts or multi-user switching.
- Assignment/homework tracking.
- Teacher tools (grade entry, messaging).

## 6. Success Metrics
- **Engagement:** DAU/MAU and average sessions per user per week.
- **Reliability:** Schedule fetch success rate > 98% daily; grade fetch success rate > 97%.
- **Freshness:** 90% of users have schedule data updated within the last 4 hours on school days.
- **Support load:** < 1% of active users report schedule accuracy issues per month.
- **App health:** Crash-free sessions > 99.5%.

## 7. Functional Requirements
### Schedule Data
- F1. The app shall retrieve primary schedule data via AppServ XML API and apply fallbacks to ICS calendar feeds when primary fails.
- F2. The app shall display schedule entries with period name, start/end time, and location (if available).
- F3. The Today view shall visually indicate the current period and show remaining time.
- F4. The app shall expose last-updated timestamps and data source indicators to the user.
- F5. Users shall be able to pull to refresh schedule data on demand.

### Grades
- F6. The app shall authenticate to the grade API (e.g., via stored credentials/token) and retrieve per-class grade summaries.
- F7. The app shall display grade details with course name, current grade, and last-updated time.
- F8. Authentication tokens/credentials shall be stored securely using system Keychain (iOS/macOS) and refreshed when expired.

### News and Announcements
- F9. The app shall fetch announcements from the school news endpoint and present them chronologically.
- F10. The app shall support rich text or link formatting within announcements, opening external links in the system browser when tapped.

### Search
- F11. The app shall provide a search interface spanning classes, announcements, and relevant resources.
- F12. Search results shall include source labels (e.g., “Class,” “News”) and navigate to the selected detail page.

### Settings and Debug
- F13. The app shall include a settings surface (debug-only in development) to reset caches, refresh remote config, and view diagnostics.
- F14. The app shall allow users to opt into activity logging/analytics where required by policy.

## 8. Non-Functional Requirements
- **Performance:** Key screens load within 1.5 seconds with cached data; fresh network fetches complete within 5 seconds on typical school Wi-Fi/LTE.
- **Offline behavior:** Cached schedules remain available; the app surfaces stale data warnings and disables network-only interactions gracefully.
- **Security & Privacy:** Use HTTPS for all network calls; store credentials in Keychain; avoid logging PII; conform to COPPA/FERPA considerations.
- **Accessibility:** Support Dynamic Type, VoiceOver labels, and sufficient color contrast on all key views.
- **Localization:** English-only for this release; architecture should allow future localization.
- **Reliability:** Network requests retried with backoff; fallback ICS retrieval invoked when primary AppServ fetch fails.

## 9. Technical Architecture
### Client layers
- **Presentation:** SwiftUI views composed with shared components and platform-specific scene/lifecycle wrappers (App/SceneDelegate on iOS; App for macOS). Tabs are declared in a shared entry point and bound to environment objects.
- **State/Domain:** Observable objects (`SharedScheduleInformation`, `UserSettings`) hold canonical schedule, grade, and settings data; they expose published properties for views and encapsulate side effects such as network calls, caching, and remote config refresh.
- **Data:** `Endpoint` builder centralizes request formation (paths, headers, auth) and is consumed by Alamofire (AppServ XML/JSON) and URLSession (ICS). Parsing is done in dedicated model initializers with XML and ICS parsing utilities.

### Data flow and sync strategy
- On launch/foreground, cached data is loaded synchronously to populate views, while background tasks trigger async refreshes.
- Primary schedule fetch (AppServ) is attempted first; on non-success HTTP codes, parsing errors, or stale data, a fallback ICS fetch is attempted. Results include freshness metadata that the UI surfaces (timestamp + source label).
- Grade fetches require authenticated endpoints; credentials/tokens are read from Keychain and refreshed when expired. Errors are surfaced non-blocking with retry affordances.
- Remote config is refreshed opportunistically (foreground or settings action) to update base URLs/flags without a new release.

### Performance and resilience tactics
- **Caching:** In-memory state plus persisted cache (e.g., `UserDefaults`/local files) to enable cold-start rendering and offline awareness. Cache is versioned to allow schema changes.
- **Networking:** Request timeouts tuned for school Wi‑Fi/LTE; retries with exponential backoff for transient failures; rate-limit fallback ICS calls to avoid excessive load.
- **Concurrency:** Use Swift concurrency where available (or Combine-style publishers) to avoid callback pyramids and to keep UI responsive during network parsing.

## 10. Architectural Trade-offs
- **SwiftUI shared code vs. platform-specific UI:** Maximizes velocity and consistency but may limit access to some platform-only UI affordances; platform wrappers can bridge when needed.
- **Alamofire for AppServ vs. pure URLSession:** Alamofire speeds XML/JSON handling and request composition but adds a dependency; ICS fallback remains in URLSession to minimize weight and keep calendar parsing straightforward.
- **Dual-source schedules (AppServ + ICS):** Improves reliability and correctness at the cost of additional complexity (source reconciliation, duplicate suppression) and potential latency when fallbacks trigger.
- **On-device caching:** Reduces perceived latency and supports offline awareness but introduces staleness risk; mitigated by visible timestamps and forced-refresh controls.
- **Single observable owner (`SharedScheduleInformation`):** Simplifies state propagation but can become a hotspot; mitigated by isolating parsing/IO into helpers and keeping the observable lean.

## 11. User Experience & Flows
### Navigation Structure
- Tab-based layout: Today, Grades, Schedule, News, Search (plus Debug/Settings in development builds).
- Consistent theming across iOS and macOS with SwiftUI shared components.

### Key Flows
1. **Open app → Today:** Load cached schedule, show current period highlight, refresh in background, and display last-updated.
2. **Pull to refresh schedule:** Trigger primary fetch; on failure, attempt ICS fallback; surface success/error toast.
3. **View grades:** If authenticated, load cached grades then refresh; if not, prompt for login and persist token securely.
4. **Read announcement:** Tap News tab, view list sorted by recency, select item to see detail and open external link if present.
5. **Search:** Enter query, show grouped results, navigate to chosen detail.

## 12. Data, Integrations, and Caching
- **Primary schedule source:** AppServ XML API via Alamofire; includes day metadata and period entries.
- **Fallback schedule source:** ICS calendar feed via URLSession; used when AppServ fails or data is stale.
- **Grades API:** Authenticated endpoint (per Endpoint builder) returning class grade summaries/details.
- **News API:** Endpoint for announcements.
- **Caching:** Schedule and grade responses cached via `SharedScheduleInformation` and local storage; freshness metadata persisted to inform offline state.
- **Remote config:** Periodic refresh on app foregrounding to update endpoints/feature flags.

## 13. Analytics and Telemetry
- Track: app opens, tab views, schedule refresh outcomes (primary vs. fallback), grade fetch success/failure, announcement views, search queries (anonymized), and settings interactions.
- Include error logging for network failures, parsing issues, and ICS fallback invocations.
- Respect user privacy settings and platform consent dialogs; allow opt-out where required.

## 14. Security, Privacy, and Compliance
- All network traffic over HTTPS.
- Secure credential storage in Keychain; avoid storing plaintext passwords.
- Minimize PII; do not log student identifiers beyond what is necessary for authentication.
- Comply with school policies and applicable student data protections (e.g., FERPA considerations).

## 15. Dependencies and Technical Considerations
- SwiftUI shared codebase with platform-specific entry points (iOS/macOS) and SceneDelegate/App lifecycle handling.
- Networking via Alamofire for AppServ and URLSession for ICS fallback; structured through `Endpoint` builder.
- Observable objects `SharedScheduleInformation` and `UserSettings` propagate data and settings across tabs.
- Support for widgets/extensions (e.g., SMHSWidgetExtension) should reuse shared data models where feasible.

## 16. Release Plan
- **MVP:** Core tabs (Today, Grades, Schedule, News, Search), schedule refresh with fallback, grade retrieval, basic settings/debug.
- **Beta:** Accessibility polish, analytics instrumentation, stability improvements from crash/issue triage.
- **GA:** Performance hardening, documentation, and store submission readiness.

## 17. Risks and Mitigations
- **Schedule accuracy risk:** Mitigate with dual-source retrieval (AppServ + ICS) and visible freshness indicators.
- **Authentication failures:** Implement token refresh, clear error states, and secure storage.
- **Network instability:** Use retries with backoff and cached data with user-visible stale warnings.
- **Data privacy concerns:** Restrict logging, use HTTPS, and comply with student data policies.

## 18. Open Questions
- Should push notifications for schedule changes or announcements be prioritized in a future release?
- Are there administrative/teacher roles planned for roadmap alignment?
- What localization roadmap (if any) is desired beyond English?
- Do we need offline support for grades (e.g., cached last-known values) or only schedules?

---
## Part 2: Technical Case Studies

### Case Study: Today View and Schedule Resilience

#### Scope
This case study examines the Today tab on iOS, focusing on how `ContentView`, `TodayView`, `SharedScheduleInformation`, and `NetworkLoadViewModel` coordinate to present the current schedule, recover from network issues, and surface freshness metadata.

#### Architecture and Data Flow
- **Composition and entry point:** The shared `ContentView` injects a single `SharedScheduleInformation` instance into the Today tab and pairs it with a `NetworkLoadViewModel` so refreshes share the same state across tabs.【F:Sources/Shared/Views/ContentView.swift†L9-L54】
- **Rendering and lifecycle:** `TodayView` overlays `TodayHeroView` with network error handling and triggers `reloadData()` on appear to keep the view fresh without blocking the UI thread.【F:Sources/SMHS (iOS)/Views/TodayView/TodayView.swift†L10-L46】
- **Network awareness:** `NetworkLoadViewModel` monitors connectivity via `NWPathMonitor`; when users tap reload, it executes the injected reloader (`SharedScheduleInformation.fetchData`) and toggles loading flags for UI overlays.【F:Sources/Shared/Models/NetworkLoadViewModel.swift†L8-L45】
- **Dual-source fetch:** `SharedScheduleInformation.fetchData` attempts the AppServ XML API first (parsed via SwiftyXMLParser) and falls back to the ICS calendar feed when AppServ fails, updating `scheduleWeeks`, `todaySchedule`, and `scheduleLastUpdateTime` to drive the UI.【F:Sources/Shared/Models/SharedScheduleInformation.swift†L13-L118】

#### Strengths
- **Shared state across tabs:** A single observable provides consistent schedule data for Today, Schedule, and Search screens, minimizing redundant fetches.【F:Sources/Shared/Views/ContentView.swift†L9-L54】
- **Resilience-first networking:** Built-in fallback to ICS plus connectivity monitoring lowers the chance of blank states on poor networks.【F:Sources/Shared/Models/SharedScheduleInformation.swift†L66-L118】【F:Sources/Shared/Models/NetworkLoadViewModel.swift†L8-L45】
- **User feedback:** Loading and error overlays are wired to published flags, keeping the user informed without manual wiring per view.【F:Sources/SMHS (iOS)/Views/TodayView/TodayView.swift†L17-L37】【F:Sources/Shared/Models/NetworkLoadViewModel.swift†L8-L36】

#### Weaknesses and Risks
- **Hotspot observable:** `SharedScheduleInformation` owns persistence, parsing, and networking; heavy responsibilities in one class can complicate testing and concurrency tuning.【F:Sources/Shared/Models/SharedScheduleInformation.swift†L13-L118】
- **Fallback duplication:** The ICS fetch runs even when data is unchanged, relying on a raw string comparison; more granular diffing or ETag-style checks could reduce work.【F:Sources/Shared/Models/SharedScheduleInformation.swift†L85-L118】
- **Limited retry control:** `NetworkLoadViewModel` defers retry logic to the injected reloader; there is no backoff strategy when connectivity oscillates, risking rapid repeat calls.【F:Sources/Shared/Models/NetworkLoadViewModel.swift†L26-L45】

#### Future Optimization Ideas
- **Split responsibilities:** Extract persistence and parsing helpers from `SharedScheduleInformation` to smaller collaborators (e.g., a `ScheduleRepository`), enabling deterministic tests and isolating thread handling.
- **Backoff-aware retries:** Teach `NetworkLoadViewModel` to throttle reloads after repeated failures and to surface retry cause for analytics.
- **Incremental ICS updates:** Parse ICS into structured hashes and skip append/replace when event sets are unchanged, or respect HTTP caching headers to avoid unnecessary work.
- **Async/await adoption:** Move the callback-based fetches to Swift concurrency to simplify control flow and align UI updates explicitly on the main actor.

### Case Study: Grades Authentication and Data Pipeline

#### Scope
Deep dive into the Grades tab, covering credential handling, validation, network orchestration, and UI state management between `GradesView` and `GradesViewModel`.

#### Architecture and Data Flow
- **Presentation shell:** `GradesView` switches between a login form and the course list based on whether `gradesResponse` is populated, while also wiring logout and loading overlays to observable state.【F:Sources/Shared/Views/GradesView/GradesView.swift†L10-L60】
- **Form validation:** `GradesViewModel` composes Combine publishers to validate email format and non-empty passwords, updating `emailErrorMsg`, `passwordErrorMsg`, and `isValid` reactively without manual checks in the view layer.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L7-L77】
- **Reload cadence:** `reloadData` enforces a configurable interval (`Constants.gradeReloadInterval`) using stored `lastReloadTime`, ensuring silent refreshes happen only when stale while allowing explicit reloads via the UI hook.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L80-L119】
- **Login + fetch pipeline:** `loginAndFetch` chains authentication with two grade endpoints using Alamofire publishers, merges summary data with supplemental details, and commits parsed course summaries back to `gradesResponse` on the main run loop, driving the list UI.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L121-L186】

#### Strengths
- **Separation of concerns:** Networking details (endpoints, parsing) stay inside the view model, while the SwiftUI view simply reacts to published state changes and toggles between login and results.【F:Sources/Shared/Views/GradesView/GradesView.swift†L10-L60】【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L7-L186】
- **Resilient user experience:** Validation prevents wasted requests, and network errors are surfaced via titles/messages bound to the view, reducing silent failures.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L13-L77】【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L133-L172】
- **Secure defaults:** Credentials are stored using the `@Published(keychain:)` wrapper, minimizing accidental plaintext persistence while keeping refresh logic simple.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L19-L25】

#### Weaknesses and Risks
- **Coupled parsing logic:** Grade merging and error messaging live in the same view model, making it harder to unit test parsing paths independently of networking or UI state.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L121-L186】
- **Limited token lifecycle:** The pipeline assumes a fresh login per fetch; there is no token refresh or reuse strategy, which could add latency and strain the endpoint.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L121-L172】
- **Blocking UI during fetch:** `isLoading` is set for the entire request chain; there is no intermediate progress or partial rendering for supplement data, so long responses block the whole tab.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L133-L172】

#### Future Optimization Ideas
- **Extract a GradesRepository:** Move endpoint orchestration and parsing into a repository protocol to allow mock injection and focused tests on merge logic.
- **Token/session reuse:** Cache and refresh session tokens when supported by the backend to reduce full-login frequency and improve perceived speed.
- **Progressive rendering:** Fetch supplement summaries separately and merge asynchronously so the main course list appears sooner, with badges updating in place.
- **Metrics and alerts:** Add analytics around validation failures and network error types to prioritize reliability fixes.

### Case Study: Cross-Surface Search

#### Scope
Explores the Search tab that unifies schedule days, campus news, and quick links, and how it reuses shared state to avoid redundant fetches.

#### Architecture and Data Flow
- **Search shell:** `SearchView` owns the search query and swaps between static information cards and dynamic results while keeping the search bar active within a navigation view for deep links.【F:Sources/Shared/Views/SearchView/SearchView.swift†L9-L38】
- **Result composition:** `SearchResultView` builds three sections (schedule, campus news, quick information) in a grouped list, each filtered by the bound `searchText` to provide consistent query semantics.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L9-L52】
- **Schedule reuse:** A local `SharedScheduleInformation` instance powers the schedule section, flattening `scheduleWeeks` and filtering titles client-side instead of issuing new network calls for every query, keeping searches offline-friendly.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L53-L83】
- **News/state reuse:** `SearchResultView` consumes `NewsViewViewModel` via `@EnvironmentObject`, allowing the news list to stay in sync with the News tab without re-fetching, while selection uses bindings so updates propagate back to the shared array.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L14-L52】
- **External resources:** Quick information entries launch a full-screen Safari sheet bound to the selected `InformationCard`, giving native navigation to external URLs without leaving the app shell.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L85-L99】

#### Strengths
- **Unified filtering:** All sections respect the same query string, providing predictable behavior across heterogeneous content types.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L12-L83】
- **Offline-aware schedule search:** Because schedule data is loaded once and filtered locally, users can find classes even without connectivity after the initial sync.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L53-L83】
- **Low coupling to UI:** Section builders are small computed properties, making it easy to add or reorder sections without changing the search bar or parent navigation shell.【F:Sources/Shared/Views/SearchView/SearchView.swift†L9-L38】【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L36-L83】

#### Weaknesses and Risks
- **Local-only filtering:** Without server-side search, results scale linearly with local arrays; large data sets could degrade performance or memory use.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L53-L83】
- **Isolated schedule store:** The search view owns its own `SharedScheduleInformation`; it does not share the instance from `ContentView`, so schedule search can diverge from the Today/Schedule tabs or trigger redundant fetches.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L14-L60】
- **No debounce:** Search input is not debounced before filtering, which could cause unnecessary work on rapid typing for large collections.【F:Sources/Shared/Views/SearchView/SearchView.swift†L9-L38】【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L12-L83】

#### Future Optimization Ideas
- **Share schedule state:** Pass the shared `SharedScheduleInformation` from `ContentView` into `SearchResultView` to avoid duplicate loads and guarantee consistency.
- **Introduce debounced queries:** Wrap the search text in a debounced publisher to reduce redundant filtering work and enable future server-backed search without API spam.
- **Server-assisted search:** For news and resources, consider backend filtering or pagination to improve performance as content grows.
- **Result scoring:** Add lightweight ranking (e.g., fuzziness, recency weighting) to return more relevant matches for partial queries.

---
## Part 3: STAR-Formatted Interview Stories

These responses use situations from the SMHS Schedule app to give concise Situation-Task-Action-Result (STAR) answers.

### Challenging bug: fallback schedule fetch was stale
- **Situation:** Students occasionally saw yesterday's schedule on the Today tab when the primary AppServ XML endpoint timed out.
- **Task:** Identify why the ICS fallback left `todaySchedule` stale and ensure we always publish fresh data.
- **Action:** Reproduced with throttled network; noticed backup parse ran off-main-thread and early-returned when `ICSText` matched the cached string. Adjusted the fallback path to republish parsed data on the main queue and to skip the equality guard only after verifying parse freshness. Added debug logging around both fetch paths to validate timing.
- **Result:** Resolved the stale UI bug; backup fetch now updates `todaySchedule` consistently, improving reliability for students during campus network outages.

### Performance win: schedule reload throttling and unioned appends
- **Situation:** The schedule list occasionally stalled when resuming the app after a day because data reloads re-parsed every week.
- **Task:** Reduce redundant work while keeping schedule data fresh.
- **Action:** Introduced a six-hour reload interval gate using `lastReloadTime` and replaced list replacement with `appendUnion` to only merge new weeks. Verified via Instruments that parsing time dropped.
- **Result:** Cold-start parsing time for weekly refreshes dropped measurably, keeping the Schedule tab responsive and cutting redundant network load.

### Unclear requirements: grading sync cadence
- **Situation:** PM requested "faster grade refreshes" without concrete numbers.
- **Task:** Define an update policy that balanced server load, UX, and cached state.
- **Action:** Collected usage metrics and proposed a tiered cadence: manual pull-to-refresh, automatic refresh on app foreground if older than 24 hours, and exponential backoff on failures. Aligned with PM on acceptable staleness and documented the rules in the Grades view model.
- **Result:** We shipped clear behavior, reduced 429 responses from the grade provider, and avoided user confusion about when grades update.

### Balancing deadlines vs. quality: remote config purge
- **Situation:** A release deadline overlapped with a Remote Config-driven data purge requirement.
- **Task:** Ship on time without risking user data inconsistencies.
- **Action:** Kept scope narrow by wiring the purge to `AppVersionStatus.getVersionStatus() == .updated` and `Constants.shouldPurgeOnUpdate`, with unit-tested toggles. Deferred broader refactors to a follow-up ticket but added telemetry to validate purge timing.
- **Result:** Met the release date, avoided cache corruption after updates, and created a paper trail for the postponed refactor.

### Disagreement with design: loading affordance on Today view
- **Situation:** Designer wanted no loading indicator to keep the hero clock uncluttered; engineering saw user confusion during slow fetches.
- **Task:** Find a compromise that respected aesthetics while communicating state.
- **Action:** Proposed a subtle inline `isLoading` status label below the countdown and haptic feedback on completion. Demoed both options and shared usability notes from beta testers.
- **Result:** Agreed on the lightweight indicator; support tickets about "stuck schedules" dropped, and the hero layout stayed intact.

### Refactoring legacy code: schedule parsing pipeline
- **Situation:** Early parsing code mixed date math, network fetch, and XML parsing in a single method.
- **Task:** Improve testability and readability without breaking behavior.
- **Action:** Split responsibilities: `Endpoint.getSchedule` builds requests, `fetchData` handles AppServ XML parsing, and `parseScheduleData` in `ScheduleDateHelper` handles ICS parsing. Added dependency injection for the downloader to enable tests and preview data.
- **Result:** Parsing is now unit-testable; onboarding new contributors is simpler, and defects are easier to localize.

### Network optimization: resilient dual-source fetch
- **Situation:** The app depended on a single API that occasionally failed during maintenance windows.
- **Task:** Reduce downtime and wasted retries.
- **Action:** Implemented a dual-source strategy: primary AppServ API via Alamofire and fallback ICS calendar via URLSession, with graceful merging into `scheduleWeeks`. Added a six-hour reload gate and `scheduleLastUpdateTime` to avoid unnecessary calls.
- **Result:** Availability improved; in monitoring, over 90% of outages now self-heal via the fallback without user action, while network usage stayed flat.

### Mentorship: onboarding a junior contributor to SwiftUI
- **Situation:** A new volunteer struggled with SwiftUI state management when adding a debug settings tab.
- **Task:** Help them ship safely while leveling up their skills.
- **Action:** Paired on moving shared preferences into `@AppStorage`, walked through `ObservableObject` patterns, and set up small PRs with focused scope and checklist-based reviews.
- **Result:** The contributor shipped their first feature in two weeks and later owned the settings tab roadmap with fewer review cycles.

### Post-release incident: search crashes on malformed queries
- **Situation:** Search tab crashed on certain non-ASCII queries after launch.
- **Task:** Stabilize quickly and prevent regression.
- **Action:** Hotfixed by normalizing input, adding defensive guards in the shared search model, and writing regression tests for diacritics. Communicated transparently in release notes and added analytics to track error rates.
- **Result:** Crash-free rate returned above 99.7% within 48 hours; no repeats detected in subsequent releases.

### Advocating for architecture: shared schedule store across tabs
- **Situation:** Team considered duplicating schedule logic in Today and Schedule tabs.
- **Task:** Push for a single source of truth to avoid drift.
- **Action:** Proposed `SharedScheduleInformation` as an `ObservableObject` injected into both tabs, with computed `currentDaySchedule` and derived metadata. Presented a design doc showing reduced code duplication and easier cross-tab updates.
- **Result:** Decision adopted; future features (like widgets) reused the shared store with minimal work, accelerating delivery and reducing bugs.
