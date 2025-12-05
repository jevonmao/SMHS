# Case Study: Grades Authentication and Data Pipeline

## Scope
Deep dive into the Grades tab, covering credential handling, validation, network orchestration, and UI state management between `GradesView` and `GradesViewModel`.

## Architecture and Data Flow
- **Presentation shell:** `GradesView` switches between a login form and the course list based on whether `gradesResponse` is populated, while also wiring logout and loading overlays to observable state.【F:Sources/Shared/Views/GradesView/GradesView.swift†L10-L60】
- **Form validation:** `GradesViewModel` composes Combine publishers to validate email format and non-empty passwords, updating `emailErrorMsg`, `passwordErrorMsg`, and `isValid` reactively without manual checks in the view layer.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L7-L77】
- **Reload cadence:** `reloadData` enforces a configurable interval (`Constants.gradeReloadInterval`) using stored `lastReloadTime`, ensuring silent refreshes happen only when stale while allowing explicit reloads via the UI hook.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L80-L119】
- **Login + fetch pipeline:** `loginAndFetch` chains authentication with two grade endpoints using Alamofire publishers, merges summary data with supplemental details, and commits parsed course summaries back to `gradesResponse` on the main run loop, driving the list UI.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L121-L186】

## Strengths
- **Separation of concerns:** Networking details (endpoints, parsing) stay inside the view model, while the SwiftUI view simply reacts to published state changes and toggles between login and results.【F:Sources/Shared/Views/GradesView/GradesView.swift†L10-L60】【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L7-L186】
- **Resilient user experience:** Validation prevents wasted requests, and network errors are surfaced via titles/messages bound to the view, reducing silent failures.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L13-L77】【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L133-L172】
- **Secure defaults:** Credentials are stored using the `@Published(keychain:)` wrapper, minimizing accidental plaintext persistence while keeping refresh logic simple.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L19-L25】

## Weaknesses and Risks
- **Coupled parsing logic:** Grade merging and error messaging live in the same view model, making it harder to unit test parsing paths independently of networking or UI state.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L121-L186】
- **Limited token lifecycle:** The pipeline assumes a fresh login per fetch; there is no token refresh or reuse strategy, which could add latency and strain the endpoint.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L121-L172】
- **Blocking UI during fetch:** `isLoading` is set for the entire request chain; there is no intermediate progress or partial rendering for supplement data, so long responses block the whole tab.【F:Sources/Shared/Views/GradesView/ViewModel/GradesViewModel.swift†L133-L172】

## Future Optimization Ideas
- **Extract a GradesRepository:** Move endpoint orchestration and parsing into a repository protocol to allow mock injection and focused tests on merge logic.
- **Token/session reuse:** Cache and refresh session tokens when supported by the backend to reduce full-login frequency and improve perceived speed.
- **Progressive rendering:** Fetch supplement summaries separately and merge asynchronously so the main course list appears sooner, with badges updating in place.
- **Metrics and alerts:** Add analytics around validation failures and network error types to prioritize reliability fixes.
