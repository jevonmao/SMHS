# Case Study: Cross-Surface Search

## Scope
Explores the Search tab that unifies schedule days, campus news, and quick links, and how it reuses shared state to avoid redundant fetches.

## Architecture and Data Flow
- **Search shell:** `SearchView` owns the search query and swaps between static information cards and dynamic results while keeping the search bar active within a navigation view for deep links.【F:Sources/Shared/Views/SearchView/SearchView.swift†L9-L38】
- **Result composition:** `SearchResultView` builds three sections (schedule, campus news, quick information) in a grouped list, each filtered by the bound `searchText` to provide consistent query semantics.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L9-L52】
- **Schedule reuse:** A local `SharedScheduleInformation` instance powers the schedule section, flattening `scheduleWeeks` and filtering titles client-side instead of issuing new network calls for every query, keeping searches offline-friendly.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L53-L83】
- **News/state reuse:** `SearchResultView` consumes `NewsViewViewModel` via `@EnvironmentObject`, allowing the news list to stay in sync with the News tab without re-fetching, while selection uses bindings so updates propagate back to the shared array.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L14-L52】
- **External resources:** Quick information entries launch a full-screen Safari sheet bound to the selected `InformationCard`, giving native navigation to external URLs without leaving the app shell.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L85-L99】

## Strengths
- **Unified filtering:** All sections respect the same query string, providing predictable behavior across heterogeneous content types.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L12-L83】
- **Offline-aware schedule search:** Because schedule data is loaded once and filtered locally, users can find classes even without connectivity after the initial sync.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L53-L83】
- **Low coupling to UI:** Section builders are small computed properties, making it easy to add or reorder sections without changing the search bar or parent navigation shell.【F:Sources/Shared/Views/SearchView/SearchView.swift†L9-L38】【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L36-L83】

## Weaknesses and Risks
- **Local-only filtering:** Without server-side search, results scale linearly with local arrays; large data sets could degrade performance or memory use.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L53-L83】
- **Isolated schedule store:** The search view owns its own `SharedScheduleInformation`; it does not share the instance from `ContentView`, so schedule search can diverge from the Today/Schedule tabs or trigger redundant fetches.【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L14-L60】
- **No debounce:** Search input is not debounced before filtering, which could cause unnecessary work on rapid typing for large collections.【F:Sources/Shared/Views/SearchView/SearchView.swift†L9-L38】【F:Sources/Shared/Views/SearchView/Components/SearchResultView.swift†L12-L83】

## Future Optimization Ideas
- **Share schedule state:** Pass the shared `SharedScheduleInformation` from `ContentView` into `SearchResultView` to avoid duplicate loads and guarantee consistency.
- **Introduce debounced queries:** Wrap the search text in a debounced publisher to reduce redundant filtering work and enable future server-backed search without API spam.
- **Server-assisted search:** For news and resources, consider backend filtering or pagination to improve performance as content grows.
- **Result scoring:** Add lightweight ranking (e.g., fuzziness, recency weighting) to return more relevant matches for partial queries.
