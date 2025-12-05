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
- Support for widgets/ extensions (e.g., SMHSWidgetExtension) should reuse shared data models where feasible.

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

