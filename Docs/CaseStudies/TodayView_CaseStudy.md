# Case Study: Today View and Schedule Resilience

## Scope
This case study examines the Today tab on iOS, focusing on how `ContentView`, `TodayView`, `SharedScheduleInformation`, and `NetworkLoadViewModel` coordinate to present the current schedule, recover from network issues, and surface freshness metadata.

## Architecture and Data Flow
- **Composition and entry point:** The shared `ContentView` injects a single `SharedScheduleInformation` instance into the Today tab and pairs it with a `NetworkLoadViewModel` so refreshes share the same state across tabs.【F:Sources/Shared/Views/ContentView.swift†L9-L54】 
- **Rendering and lifecycle:** `TodayView` overlays `TodayHeroView` with network error handling and triggers `reloadData()` on appear to keep the view fresh without blocking the UI thread.【F:Sources/SMHS (iOS)/Views/TodayView/TodayView.swift†L10-L46】 
- **Network awareness:** `NetworkLoadViewModel` monitors connectivity via `NWPathMonitor`; when users tap reload, it executes the injected reloader (`SharedScheduleInformation.fetchData`) and toggles loading flags for UI overlays.【F:Sources/Shared/Models/NetworkLoadViewModel.swift†L8-L45】 
- **Dual-source fetch:** `SharedScheduleInformation.fetchData` attempts the AppServ XML API first (parsed via SwiftyXMLParser) and falls back to the ICS calendar feed when AppServ fails, updating `scheduleWeeks`, `todaySchedule`, and `scheduleLastUpdateTime` to drive the UI.【F:Sources/Shared/Models/SharedScheduleInformation.swift†L13-L118】

## Strengths
- **Shared state across tabs:** A single observable provides consistent schedule data for Today, Schedule, and Search screens, minimizing redundant fetches.【F:Sources/Shared/Views/ContentView.swift†L9-L54】
- **Resilience-first networking:** Built-in fallback to ICS plus connectivity monitoring lowers the chance of blank states on poor networks.【F:Sources/Shared/Models/SharedScheduleInformation.swift†L66-L118】【F:Sources/Shared/Models/NetworkLoadViewModel.swift†L8-L45】
- **User feedback:** Loading and error overlays are wired to published flags, keeping the user informed without manual wiring per view.【F:Sources/SMHS (iOS)/Views/TodayView/TodayView.swift†L17-L37】【F:Sources/Shared/Models/NetworkLoadViewModel.swift†L8-L36】

## Weaknesses and Risks
- **Hotspot observable:** `SharedScheduleInformation` owns persistence, parsing, and networking; heavy responsibilities in one class can complicate testing and concurrency tuning.【F:Sources/Shared/Models/SharedScheduleInformation.swift†L13-L118】
- **Fallback duplication:** The ICS fetch runs even when data is unchanged, relying on a raw string comparison; more granular diffing or ETag-style checks could reduce work.【F:Sources/Shared/Models/SharedScheduleInformation.swift†L85-L118】
- **Limited retry control:** `NetworkLoadViewModel` defers retry logic to the injected reloader; there is no backoff strategy when connectivity oscillates, risking rapid repeat calls.【F:Sources/Shared/Models/NetworkLoadViewModel.swift†L26-L45】

## Future Optimization Ideas
- **Split responsibilities:** Extract persistence and parsing helpers from `SharedScheduleInformation` to smaller collaborators (e.g., a `ScheduleRepository`), enabling deterministic tests and isolating thread handling.
- **Backoff-aware retries:** Teach `NetworkLoadViewModel` to throttle reloads after repeated failures and to surface retry cause for analytics.
- **Incremental ICS updates:** Parse ICS into structured hashes and skip append/replace when event sets are unchanged, or respect HTTP caching headers to avoid unnecessary work.
- **Async/await adoption:** Move the callback-based fetches to Swift concurrency to simplify control flow and align UI updates explicitly on the main actor.
