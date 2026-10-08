# iOS Interview Prep - Index

> **TL;DR:** Topic-focused study notes split out from the original monolithic [plan.md](plan.md). Read in order for an end-to-end refresher, or jump straight to the topic you need.

`plan.md` is the original checklist roadmap and is kept here untouched for reference. The numbered files below expand each section with code samples, interview Q&A, and pitfalls.

## Study Order

| #  | File | Focus | Time |
| -- | ---- | ----- | ---- |
| 01 | [01-swift-essentials.md](01-swift-essentials.md) | Optionals, value/reference types, generics, closures, errors | 2-3 h |
| 02 | [02-swiftui-basics.md](02-swiftui-basics.md) | Declarative UI, property wrappers, layout, navigation | 2-3 h |
| 03 | [03-uikit-basics.md](03-uikit-basics.md) | VC lifecycle, Auto Layout, tables/collections | 2 h |
| 04 | [04-mvvm-and-architecture.md](04-mvvm-and-architecture.md) | MVVM, Coordinator, Clean Architecture, DI | 2 h |
| 05 | [05-concurrency-async-await.md](05-concurrency-async-await.md) | GCD, async/await, actors, MainActor, Sendable | 3 h |
| 06 | [06-combine-framework.md](06-combine-framework.md) | Publishers/Subscribers, operators, @Published vs Rx | 2 h |
| 07 | [07-core-data-and-swiftdata.md](07-core-data-and-swiftdata.md) | Core Data stack, SwiftData @Model, migrations | 2 h |
| 08 | [08-networking-urlsession.md](08-networking-urlsession.md) | URLSession, Codable, error handling, retries | 2 h |
| 09 | [09-persistence-options.md](09-persistence-options.md) | UserDefaults, Keychain, files, Realm vs Core Data | 1 h |
| 10 | [10-testing-ios.md](10-testing-ios.md) | XCTest, XCUITest, mocks via protocols | 2 h |
| 11 | [11-app-lifecycle-and-state.md](11-app-lifecycle-and-state.md) | AppDelegate, SceneDelegate, state restoration | 1 h |
| 12 | [12-performance-and-instruments.md](12-performance-and-instruments.md) | Instruments, leaks, image perf, launch time | 2 h |
| 13 | [13-build-system-and-spm.md](13-build-system-and-spm.md) | Xcode build, SPM, CocoaPods legacy, schemes | 1 h |
| 14 | [14-distribution-and-app-store.md](14-distribution-and-app-store.md) | TestFlight, signing, App Review | 1 h |
| 15 | [15-accessibility-ios.md](15-accessibility-ios.md) | VoiceOver, Dynamic Type, traits, rotors | 1 h |
| 16 | [16-modern-ios-apis.md](16-modern-ios-apis.md) | Widgets, App Intents, App Clips, APNs, BG tasks | 2 h |

## Suggested 1-Week Crash Plan

- **Day 1:** 01, 02
- **Day 2:** 03, 11
- **Day 3:** 04, 05
- **Day 4:** 06, 08
- **Day 5:** 07, 09
- **Day 6:** 10, 12
- **Day 7:** 13, 14, 15, 16

## Cross-Links

- [../serious-prep/](../serious-prep/) - language-agnostic DSA, system design, behavioral prep
- [../android/](../android/) - Android counterparts (Kotlin, Jetpack Compose, lifecycle, etc.)

## How to Use These Notes

1. **Skim TL;DR + Interview Questions first** to gauge what you know.
2. **Re-derive code samples by hand** in Xcode rather than just reading.
3. **Verbalize answers** aloud - interviews reward fluency, not silent recall.
4. **Use Common Pitfalls as a checklist** the night before an interview.
