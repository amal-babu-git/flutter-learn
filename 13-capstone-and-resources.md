# Part 13 — Capstone and Beyond

[← Ship It](12-shipping.md) · [Index](README.md)

---

## 13.1 Capstone: ERP field-ops client

Build this and you will have exercised every concept in this tutorial. Scope it as a real internal tool against a FastAPI backend you control.

**Domain:** a field-operations client for an ERP — stock counts, order fulfilment, delivery confirmation.

### Requirements

1. **Auth** — email/password login against your FastAPI JWT endpoints, secure token storage, single-flight refresh interceptor, biometric unlock on relaunch, logout that clears everything.
2. **Orders list** — server-paginated infinite scroll, filter by status, pull-to-refresh, search with debounce, empty/error/loading states.
3. **Order detail** — line items, customer info, an action (`Submit` / `Fulfil`) that is optimistic with rollback and sends an idempotency key.
4. **Stock count** — offline-first. Scan a barcode (`mobile_scanner`), enter a quantity, save locally; an outbox syncs on reconnect with conflict handling.
5. **Attachments** — camera capture, local compression in an isolate, multipart upload with progress, retry on failure.
6. **Dashboard** — a few aggregate figures with charts (`fl_chart`), cached with a TTL.
7. **Settings** — theme mode, language, environment indicator, app version, sign out.
8. **Cross-cutting** — go_router with auth redirect + `StatefulShellRoute` tabs, deep link to `/orders/:id` from a push notification, i18n for two languages, adaptive layout for phone and tablet.

### Engineering bar

- Flutter's [official package structure](03-project-setup.md#35-folder-structure-the-official-layout), MVVM, abstract repositories with `_remote` and `_dev` implementations
- Riverpod for state + DI; `freezed` for every model; `Result<T>` at every repository boundary
- Drift for local persistence with a tested migration
- ≥70% coverage on view models and repositories; widget tests for every screen's four states; 3 integration tests
- CI: format, analyze (`--fatal-infos`), test, build
- Flavors for dev/staging/prod with distinct app ids
- Sentry wired with obfuscation symbols uploaded

### Build order

Build it in this order — each stage should be shippable:

`auth` → `orders list` → `order detail` → `offline stock count` → `attachments` → `dashboard` → `polish`

> Don't build the whole architecture first and features second; grow the architecture as the second and third features demand it.

---

## 13.2 A realistic 8-week learning path

Assumes ~10 focused hours/week alongside your day job. Your existing architecture instincts mean [Parts 4](04-architecture.md)–[8](08-persistence-offline.md) will feel like recognition rather than learning; budget your time toward Dart fluency and layout intuition, which are the genuinely new parts.

| Week | Focus | Deliverable |
|---|---|---|
| 1 | [Dart](01-dart-crash-course.md) — do the DartPad exercises at the end. Don't skim records/patterns/sealed. | Four exercises working; a `Result<T>` you wrote yourself |
| 2 | [Flutter fundamentals](02-flutter-fundamentals.md) — widget tree, layout, constraints. Build 5 static screens from designs you like. | 5 pixel-accurate static screens, no state |
| 3 | [State](05-state-management.md#51-the-ladder-of-state) + [navigation](07-navigation.md) — `setState` → `ValueNotifier` → Riverpod, go_router with auth redirect | Multi-screen app with login gating |
| 4 | [Networking](06-networking-auth-data.md) — Dio, `freezed` DTOs, repositories, the refresh interceptor, error taxonomy | List + detail against your real FastAPI backend |
| 5 | [Architecture](04-architecture.md) — refactor week. Impose MVVM, extract repositories, introduce `Result` and `Command`. | Same app, properly layered |
| 6 | [Persistence](08-persistence-offline.md) — Drift schema, reactive `watch()` queries, offline-first read cache, optimistic writes | Works on airplane mode |
| 7 | [Testing](10-testing.md) + [performance](11-performance.md) — write tests for what exists, then profile on a real mid-range device | 70% coverage; no frames over budget |
| 8 | [Ship](12-shipping.md) — flavors, signing, CI, Sentry, an internal release | An installed build on someone else's phone |

**Meta-advice:** the fastest way through weeks 2–3 is to rebuild a screen you've already built in React. You'll be comparing against a known-good mental model instead of learning UI and Flutter simultaneously.

**Where people stall:** layout (spend a full session in the Layout Explorer; it clicks and then never un-clicks) and over-engineering architecture before having two features to generalize from. Ship something ugly in week 3.

---

## 13.3 Pitfalls table

| Pitfall | Symptom | Fix |
|---|---|---|
| `FutureBuilder(future: fetch())` inline | Infinite spinner, request storm | Create the future in `initState`/a provider |
| Missing `dispose()` | Memory grows, listeners fire after unmount | Dispose every controller/subscription/timer |
| `setState` after `await` | `setState() called after dispose()` | `if (!mounted) return;` |
| `BuildContext` across an async gap | Crash or wrong-context navigation | `if (!context.mounted) return;` — the lint catches it |
| No keys on reorderable lists | State jumps between rows | `key: ValueKey(item.id)` |
| `ListView(children: [...])` for long lists | Jank, memory spike | `ListView.builder` |
| Missing `const` | Excess rebuilds | Enable `prefer_const_constructors` |
| `Scaffold.of(context)` in the same build | "No Scaffold found" | Wrap in `Builder` |
| Unbounded height for a `ListView` in a `Column` | "RenderFlex overflowed" / infinite constraint error | `Expanded` or `SizedBox(height:)` |
| Tokens in `SharedPreferences` | Trivially extractable | `flutter_secure_storage` |
| N parallel refresh calls on 401 | Random logouts | Single-flight `Completer` ([§6.3](06-networking-auth-data.md#63-the-jwt-refresh-interceptor)) |
| No `==` on models | Riverpod rebuilds constantly | `freezed` |
| Business logic in widgets | Untestable, duplicated | Move to view model |
| Widgets importing `Dio` | Layer violation | Repository interface |
| Benchmarking in debug mode | "Flutter is slow" | `--profile` on a real device |
| `pumpAndSettle` with a looping animation | Test hangs forever | `pump(Duration)` instead |
| Absolute sizes everywhere | Breaks on tablets/foldables | `LayoutBuilder`, `Expanded`, breakpoints |
| Ignoring `flutter analyze` warnings | Bugs ship | `--fatal-infos` in CI |
| Committing `--dart-define` secrets | Extractable from the APK | Move behind the backend |

---

## 13.4 Resources

### Official — read these first, they're unusually good

- [Flutter docs](https://docs.flutter.dev)
- **[App architecture guide](https://docs.flutter.dev/app-architecture/guide)** — the most important non-obvious page
- [Architecture case study](https://docs.flutter.dev/app-architecture/case-study)
- [Architecture recommendations](https://docs.flutter.dev/app-architecture/recommendations)
- [Design patterns](https://docs.flutter.dev/app-architecture/design-patterns) — Command, Result, offline-first, optimistic state
- [Dart language tour](https://dart.dev/language)
- [Effective Dart](https://dart.dev/effective-dart)
- [Understanding constraints](https://docs.flutter.dev/ui/layout/constraints)
- [Flutter for React Native devs](https://docs.flutter.dev/flutter-for/react-native-devs)
- [Performance best practices](https://docs.flutter.dev/perf/best-practices)
- [Release notes](https://docs.flutter.dev/release/release-notes)
- [API reference](https://api.flutter.dev)
- [DartPad](https://dartpad.dev) — browser playground, no install

### Packages

- [Riverpod](https://riverpod.dev) — read "What's new in 3.0" and the [migration guide](https://riverpod.dev/docs/3.0_migration)
- [go_router](https://pub.dev/packages/go_router)
- [Dio](https://pub.dev/packages/dio)
- [Drift](https://drift.simonbinder.eu)
- [freezed](https://pub.dev/packages/freezed)
- [Flutter Favorites](https://docs.flutter.dev/packages-and-plugins/favorites) — vetted packages

### Learning

- [Flutter's own learning pathway](https://docs.flutter.dev/learn/pathway)
- [Code with Andrea](https://codewithandrea.com) — best architecture writing in the ecosystem
- [Very Good Ventures blog](https://verygood.ventures/blog) — production practices, Bloc-flavoured
- Flutter YouTube channel — *Widget of the Week*, *Decoding Flutter*
- [Official samples](https://github.com/flutter/samples) — see `compass_app`, the architecture guide's reference implementation

### Community

- r/FlutterDev, Flutter Discord, `#flutter` on Stack Overflow
- Flutter GitHub issues — surprisingly good search for "why does this widget do that"

---

## Closing note

Three things are worth repeating because they're what separate a Flutter app that survives two years from one that gets rewritten:

1. **Adopt the official MVVM + repository architecture on day one.** It costs almost nothing at the start and it is what makes testing, offline support, and team growth possible later. You already believe this from backend work — the same instincts apply unchanged.

2. **Your backend stays the source of truth.** The client is a presentation layer with a cache. Every business rule, every permission check, every calculation of record lives in FastAPI. The Flutter app's job is to render state, capture intent, and handle the network being awful.

3. **Learn the widget/element/render distinction and the constraint algorithm properly.** They're the two pieces of Flutter theory with no React analogue, and understanding them converts most "why is Flutter doing this" moments into "oh, obviously."

Everything else in this tutorial is a detail you can look up.

---

[← Ship It](12-shipping.md) · [Index](README.md)
