# Flutter for Full-Stack Engineers

**A beginner-to-advanced Flutter tutorial written for engineers who already ship production software.**

Written against **Flutter 3.44 / Dart 3.12** (stable as of May 2026).

---

## Who this is for

You build scalable backends (FastAPI, PostgreSQL, Redis) and React/Next.js frontends. You've architected ERP systems. You care about layering, testability, and stable project structure. You want theory *and* practice — not a "build a todo app in 10 minutes" video.

This tutorial assumes:

- You know what a closure, a generic, and a discriminated union are — so Dart is taught by **contrast**, not from zero
- Your backend is the source of truth; the client is a presentation layer
- You'd rather read one page on *why* the repository pattern is structured this way than five pages of widget screenshots

---

## Contents

| # | Part | What's in it |
|---|---|---|
| 0 | **[Orientation](00-orientation.md)** | Why Flutter honestly · React/Next.js → Flutter mental model map · what changed in 3.44 |
| 1 | **[Dart Crash Course](01-dart-crash-course.md)** | Null safety · records · patterns · sealed classes · async & isolates · the `Result` type |
| 2 | **[Flutter Fundamentals](02-flutter-fundamentals.md)** | Widget/Element/RenderObject · State lifecycle · BuildContext · the constraint algorithm · keys · theming |
| 3 | **[Project Setup & Tooling](03-project-setup.md)** | Strict `analysis_options` · dependency baseline · feature-first folder structure · flavors |
| 4 | **[Architecture](04-architecture.md)** | Official MVVM + repository · layer contracts · **mapping your FastAPI backend** · DI · the Command pattern |
| 5 | **[State Management](05-state-management.md)** | The ladder of state · Riverpod 3 in depth · Bloc and when to prefer it |
| 6 | **[Networking, Auth & Data](06-networking-auth-data.md)** | Dio setup · secure token storage · **single-flight JWT refresh** · freezed DTOs · error taxonomy · pagination |
| 7 | **[Navigation](07-navigation.md)** | go_router · auth guards via redirect · shell routes · type-safe routes & deep links |
| 8 | **[Persistence & Offline-First](08-persistence-offline.md)** | Drift (SQL) · reactive queries · the offline-first spectrum · optimistic state |
| 9 | **[Forms & Real UI Work](09-forms-and-ui.md)** | Forms with server-side 422 errors · lists & tables · responsive/adaptive layout · i18n |
| 10 | **[Testing](10-testing.md)** | The pyramid · unit · widget · golden · integration |
| 11 | **[Performance](11-performance.md)** | Impeller · rebuild discipline · isolates · the DevTools workflow |
| 12 | **[Ship It](12-shipping.md)** | Secrets · build & release · CI/CD · observability · security checklist |
| 13 | **[Capstone & Resources](13-capstone-and-resources.md)** | ERP field-ops capstone spec · 8-week path · pitfalls table · curated links |

---

## How to read this

**Don't skip [Part 1](01-dart-crash-course.md).** Dart's records, patterns, and sealed classes have no direct TypeScript equivalent, and they're what make Flutter's error handling and state modelling pleasant. There are four DartPad exercises at the end — if those feel comfortable, your Dart is sufficient for everything else here.

**[Parts 2](02-flutter-fundamentals.md)–[3](03-project-setup.md)** are foundations.
**[Parts 4](04-architecture.md)–[8](08-persistence-offline.md)** are the production shape — this is where your existing architecture instincts transfer almost one-to-one.
**[Parts 9](09-forms-and-ui.md)–[12](12-shipping.md)** are the ship-it layer.
**[Part 13](13-capstone-and-resources.md)** has a capstone that mirrors ERP work, plus an 8-week schedule.

Every code block is runnable-shaped, not pseudocode. Where a block is a fragment, it says so.

---

## The three things that matter most

If you read nothing else:

1. **Adopt the [official MVVM + repository architecture](04-architecture.md) on day one.** It costs almost nothing at the start and it's what makes testing, offline support, and team growth possible later.

2. **Your backend stays the source of truth.** Every business rule, permission check, and calculation of record lives in FastAPI. The Flutter app renders state, captures intent, and handles the network being awful.

3. **Learn the [widget/element/render distinction](02-flutter-fundamentals.md#22-three-trees) and the [constraint algorithm](02-flutter-fundamentals.md#25-layout-constraints-go-down-sizes-go-up) properly.** They're the two pieces of Flutter theory with no React analogue.

---

## Version note

Written for Flutter 3.44 / Dart 3.12 (May 2026). Notable recent changes reflected here:

- Material/Cupertino being unbundled into `material_ui` / `cupertino_ui`
- Impeller default on Android 10+ (Skia removed there)
- Swift Package Manager default on iOS/macOS
- Riverpod 3 (`StateNotifierProvider` and friends are now legacy)
- Flutter's official app-architecture guide (post-2024) — the biggest change in how the team recommends structuring apps

Verify version-specific details against [the official docs](https://docs.flutter.dev), which move fast.
