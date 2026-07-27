# Part 0 — Orientation

[← Index](README.md) · [Next: Dart Crash Course →](01-dart-crash-course.md)

---

## 0.1 Why Flutter, honestly

You already know how to build UI. The question is whether Flutter's specific tradeoffs are worth learning a new language.

**What Flutter actually is:** a rendering engine plus a widget framework. Flutter does not wrap native UI components (unlike React Native, which bridges to `UIView`/`android.view.View`). Flutter ships its own renderer — **Impeller** — and draws every pixel itself onto a canvas. Text fields, scroll physics, ripples: all Flutter-drawn.

**The consequences of that choice:**

| Consequence | Why it matters to you |
|---|---|
| Pixel-identical across iOS/Android/web/desktop | Your ERP looks the same on the warehouse Android tablet and the manager's iPad. No per-platform CSS drift. |
| No JS bridge, AOT-compiled to native ARM | Predictable frame times. Animation jank is a *you* problem, not a bridge problem. |
| Own widget set | You get Material 3 and Cupertino out of the box, but you never get "free" OS-level redesigns. |
| One language top to bottom | Dart on client; Dart is also usable server-side, though you'll keep FastAPI. |
| Large binary floor (~5–8 MB before assets) | Irrelevant for enterprise apps, relevant for consumer utilities. |

**Honest downsides:** Flutter web is heavier than a Next.js SSR app and worse for SEO-facing content (use Flutter web for internal dashboards, not marketing sites). Native platform integration requires platform channels and occasionally Kotlin/Swift. The ecosystem is smaller than npm.

**Where Flutter is the right call for someone with your stack:** internal tools, field-ops apps, B2B clients on top of an existing REST/gRPC backend, anything where one team must ship iOS + Android + a desktop/web admin surface without three codebases. That is exactly the ERP-adjacent shape.

---

## 0.2 The mental model map

This table is the single highest-leverage thing in Part 0. Read it twice.

| React / Next.js | Flutter | Notes on the difference |
|---|---|---|
| Component | Widget | Widgets are **immutable configuration objects**, not instances that live. They are cheap and recreated constantly. |
| JSX | Dart expression tree | No template language. You write nested constructor calls. Verbose, but it's just Dart — you can `if`, `for`, extract methods. |
| `props` | Constructor parameters (all `final`) | Compile-time checked. No `PropTypes`, no runtime shape surprises. |
| `useState` | `setState` / `ValueNotifier` / Riverpod provider | `setState` marks the element dirty; the framework re-runs `build`. |
| `useEffect(fn, [])` | `initState()` | Runs once when the `State` is inserted into the tree. |
| `useEffect` cleanup | `dispose()` | Not optional. Controllers, subscriptions, and timers leak if you skip it. |
| `useMemo` / `useCallback` | `const` constructors, cached fields | `const` widgets are canonicalized at compile time — genuinely free. |
| Context API | `InheritedWidget` (and everything built on it) | `Theme.of(context)`, `MediaQuery.of(context)` are `InheritedWidget` lookups. O(1), not a tree walk. |
| Redux / Zustand / TanStack Query | Riverpod or Bloc | Riverpod ≈ Zustand + TanStack Query in one. Bloc ≈ Redux with per-feature stores. |
| React Router / Next App Router | `go_router` | Path-based, declarative, with a global `redirect` hook for auth. |
| `fetch` / axios | `dio` | Interceptors, cancel tokens, `FormData`. Axios's closest analogue. |
| `zod` / `pydantic` | `freezed` + `json_serializable` | Codegen, not runtime reflection. Dart has no runtime reflection in AOT. |
| `npm` / `package.json` | `pub` / `pubspec.yaml` | `flutter pub get`, lockfile is `pubspec.lock`. |
| `tsconfig` strict | `analysis_options.yaml` | Configure this on day one. See [§3.3](03-project-setup.md#33-strict-analysis-options). |
| Vite HMR | Hot reload | Sub-second, preserves state. Genuinely better than web HMR. |
| Jest + RTL | `flutter_test` | Widget tests run headless in ~ms. In the box, no config. |
| Playwright | `integration_test` | Runs on real device/emulator. |
| `next build` | `flutter build apk/ipa/web` | AOT compile per platform. |

**Two mental shifts that trip up React developers:**

1. **Widgets are not instances.** `MyButton(label: 'Save')` creates a throwaway description. Flutter diffs it against last frame's description and mutates a long-lived `Element`. So "rebuilding the whole subtree" is cheap by design — like calling a render function, not like re-mounting DOM.

2. **There is no CSS cascade.** No inheritance of `font-size`, no `z-index`, no media queries baked into styling. Layout is explicit and compositional: you wrap in `Padding`, `Center`, `Expanded`. This feels verbose at first and then feels honest — you always know exactly why something is 16 px from the edge.

**From your FastAPI side, one framing that helps:** a Flutter widget tree is closer to a *rendered response* than to a stateful DOM. Every frame, you serialize state → widget tree. Flutter's job is to make that serialization cheap and to diff it efficiently.

---

## 0.3 What changed in Flutter 3.44

Context so you don't follow outdated tutorials:

- **Material and Cupertino are being unbundled.** The libraries inside `flutter/flutter` are frozen; new work moves to standalone `material_ui` and `cupertino_ui` packages. Existing `import 'package:flutter/material.dart';` code keeps working — this is about release cadence, not API breakage. Expect design-system packages to version independently going forward.
- **Impeller is the default on Android 10+**; the Skia backend is being removed there. Practical effect: shader-compilation jank is largely gone, and old advice about "shader warm-up" is obsolete.
- **Swift Package Manager is the default on iOS/macOS.** CocoaPods is on the way out. New projects should not need a `Podfile`.
- **WebAssembly is the direction for Flutter web** — `flutter build web --wasm` gives materially better performance than the JS output.
- **AI-oriented tooling** (Dart/Flutter MCP server, GenUI SDK on the A2UI protocol) is new and experimental. Interesting; not foundational. Skip it while learning.
- **Dart 3.12** continues the post-Dart-3 line: sound null safety, records, patterns, class modifiers, dot shorthands.

**Rule of thumb for tutorials you find online:** if it uses `StateNotifierProvider`, `ChangeNotifierProvider` as a primary pattern, `.value`-style Provider plumbing, or pre-null-safety `?`-free code, it predates the current recommendations. Anything from before mid-2024 also predates Flutter's *official* architecture guide, which is the single biggest recent change in how the team recommends structuring apps.

---

## 0.4 How to use this repo

[Part 1](01-dart-crash-course.md) is dense on purpose — it assumes you know what a closure and a generic are, so it teaches Dart by *contrast* rather than from zero. Don't skip it; Dart's records/patterns/sealed classes are what make Flutter's error handling and state modelling pleasant, and they have no direct TypeScript equivalent.

[Parts 2](02-flutter-fundamentals.md)–[3](03-project-setup.md) are foundations. [Parts 4](04-architecture.md)–[8](08-persistence-offline.md) are the "production shape" and are where your existing architecture instincts transfer almost one-to-one. [Parts 9](09-forms-and-ui.md)–[12](12-shipping.md) are the ship-it layer. [Part 13](13-capstone-and-resources.md) gives you a capstone that mirrors your ERP work.

Every code block is runnable-shaped, not pseudocode. Where a block is a fragment, it says so.

---

[← Index](README.md) · [Next: Dart Crash Course →](01-dart-crash-course.md)
