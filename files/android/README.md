# Android Interview Prep

This folder is structured as focused per-topic notes plus phase indexes for sequential study. Start with the [plan](plan.md) for the recommended timeline, then drill into individual topics as needed.

## Study Order (the 6-phase roadmap)
| Phase | Index | Focus |
|-------|-------|-------|
| 1 | [phase1-core-foundations.md](phase1-core-foundations.md) | Kotlin, components, lifecycle, UI basics |
| 2 | [phase2-modern-essentials.md](phase2-modern-essentials.md) | Compose, state, coroutines, Room, Hilt, Navigation, WorkManager |
| 3 | [phase3-architecture-best-practices.md](phase3-architecture-best-practices.md) | MVVM, Clean Architecture, repository pattern, testing |
| 4 | [phase4-advanced-development.md](phase4-advanced-development.md) | Performance, custom views, animations, gestures |
| 5 | [phase5-ecosystem-senior-skills.md](phase5-ecosystem-senior-skills.md) | Gradle, modularization, CI/CD, distribution, accessibility |
| 6 | [phase6-migration-comparative.md](phase6-migration-comparative.md) | iOS / Flutter comparison, KMP & Compose Multiplatform |

## Topic Files

### Foundations (Phase 1)
- [kotlin-essentials.md](kotlin-essentials.md) — null safety, sealed/data classes, scope functions, coroutine basics
- [android-components.md](android-components.md) — Activity, Service, BroadcastReceiver, ContentProvider, Intents
- [activity-fragment-lifecycle.md](activity-fragment-lifecycle.md) — lifecycle states, config changes, state saving
- [ui-basics.md](ui-basics.md) — Views, ViewGroups, ConstraintLayout, RecyclerView

### Modern Essentials (Phase 2)
- [jetpack-compose-basics.md](jetpack-compose-basics.md) — `@Composable`, layouts, modifiers, previews
- [compose-state-management.md](compose-state-management.md) — `remember`, `mutableStateOf`, hoisting, side effects
- [coroutines-and-flow.md](coroutines-and-flow.md) — suspend, dispatchers, Flow/StateFlow/SharedFlow
- [room-database.md](room-database.md) — entities, DAOs, type converters, migrations
- [dependency-injection-hilt.md](dependency-injection-hilt.md) — `@Inject`, modules, scopes, qualifiers
- [navigation-component.md](navigation-component.md) — Compose Navigation, deep links, nested graphs
- [workmanager.md](workmanager.md) — deferrable guaranteed work, constraints, chaining

### Architecture (Phase 3)
- [mvvm-architecture.md](mvvm-architecture.md) — modern MVVM with single UiState
- [clean-architecture.md](clean-architecture.md) — Presentation / Domain / Data layers
- [repository-pattern.md](repository-pattern.md) — single source of truth, caching, error boundaries
- [testing-android.md](testing-android.md) — unit, integration, Compose UI, Espresso

Pattern primers (existing): [mvc.md](mvc.md) | [mvp.md](mvp.md) | [mvvm.md](mvvm.md)

### Advanced (Phase 4)
- [android-performance.md](android-performance.md) — startup, jank, memory, Baseline Profiles, R8
- [custom-views.md](custom-views.md) — measure/layout/draw, custom attributes
- [animations.md](animations.md) — property animators and Compose animations
- [gestures-and-touch.md](gestures-and-touch.md) — touch handling in Views and Compose

### Senior & Ecosystem (Phase 5)
- [gradle-build-system.md](gradle-build-system.md) — KTS, build types/flavors, version catalogs
- [modularization.md](modularization.md) — feature/core modules, dynamic features
- [ci-cd-android.md](ci-cd-android.md) — GitHub Actions, signing, Fastlane / Play Publisher
- [app-distribution.md](app-distribution.md) — AAB, Play tracks, staged rollouts, in-app updates
- [accessibility-android.md](accessibility-android.md) — TalkBack, semantics, touch targets

### Comparative (Phase 6)
- [android-vs-ios.md](android-vs-ios.md) — primitives, language, lifecycle, distribution
- [android-vs-flutter.md](android-vs-flutter.md) — strengths, platform channels, when to pick what
- [kmp-and-kmm.md](kmp-and-kmm.md) — Kotlin Multiplatform, `expect`/`actual`, Compose Multiplatform

### Aggregated Indexes
- [java-xml-core-foundations.md](java-xml-core-foundations.md) — Java + XML reference index
- [kotlin-compose-core-foundations.md](kotlin-compose-core-foundations.md) — Kotlin + Compose reference index

### Reference Material
- [plan.md](plan.md) — original master plan with weekly milestones
- [revision.md](revision.md) — last-minute cheatsheet
- [android_cheat_sheet.pdf](android_cheat_sheet.pdf) — printable reference

## Cross-References
- [../serious-prep/](../serious-prep/) — broader system design and behavioral prep

## How to Use
- **Brand new to Android:** follow the phases in order; one phase ≈ 1–3 weeks.
- **Refreshing for an interview:** start with [revision.md](revision.md), then deep-dive on whichever sub-topic feels weak.
- **Targeting senior roles:** pay close attention to phases 3, 4, 5 — they're where senior interviews concentrate.
- **Comparative interviews:** phase 6 plus [kmp-and-kmm.md](kmp-and-kmm.md).
