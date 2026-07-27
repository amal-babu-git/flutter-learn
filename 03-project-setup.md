# Part 3 — Project Setup and Tooling

[← Flutter Fundamentals](02-flutter-fundamentals.md) · [Index](README.md) · [Next: Architecture →](04-architecture.md)

---

## 3.1 Install and verify

```bash
# macOS (recommended path)
brew install --cask flutter
# or download the SDK from docs.flutter.dev/install and add bin/ to PATH

flutter --version          # expect 3.44.x / Dart 3.12.x
flutter doctor -v          # resolve every ✗ before writing code
flutter upgrade
```

`flutter doctor` will want: Xcode + CocoaPods-or-SwiftPM for iOS, Android Studio + SDK + an accepted licence set (`flutter doctor --android-licenses`) for Android, and Chrome for web. Editor: VS Code with the Dart + Flutter extensions, or Android Studio. Both give hot reload, the Inspector, and refactorings.

Useful daily commands:

```bash
flutter run                       # debug, hot reload with `r`, hot restart with `R`
flutter run -d chrome             # pick a device
flutter devices
flutter analyze                   # static analysis — wire into CI
dart format .                     # formatter, non-negotiable, no config
flutter test                      # unit + widget tests
flutter test --coverage
dart run build_runner watch -d    # codegen loop (freezed/riverpod/json)
flutter clean && flutter pub get  # the "turn it off and on again"
```

---

## 3.2 Creating the project properly

```bash
flutter create \
  --org in.hybridinteractive \
  --project-name erp_mobile \
  --platforms=android,ios \
  --empty \
  erp_mobile
```

- `--org` sets the bundle/application id prefix. **Changing it later is painful** (signing, Firebase, deep links). Get it right now.
- `--empty` skips the counter demo boilerplate.
- List only the platforms you'll actually ship; you can add more later with `flutter create --platforms=web .`.

---

## 3.3 Strict analysis options

Do this before you write feature code. Coming from `tsconfig` strict + `ruff`/`mypy`, you'll want more than the defaults.

```yaml
# analysis_options.yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  language:
    strict-casts: true
    strict-inference: true
    strict-raw-types: true
  errors:
    invalid_annotation_target: ignore   # noisy with freezed
    missing_required_param: error
    missing_return: error
    todo: ignore
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
    - "build/**"

linter:
  rules:
    # correctness
    always_use_package_imports: true
    avoid_dynamic_calls: true
    avoid_slow_async_io: true
    cancel_subscriptions: true
    close_sinks: true
    only_throw_errors: true
    unawaited_futures: true
    use_build_context_synchronously: true

    # style / consistency
    prefer_const_constructors: true
    prefer_const_constructors_in_immutables: true
    prefer_const_declarations: true
    prefer_final_locals: true
    prefer_single_quotes: true
    require_trailing_commas: true
    sort_pub_dependencies: true
    directives_ordering: true

    # API design
    avoid_positional_boolean_parameters: true
    prefer_named_parameters: false   # too aggressive for small helpers
    public_member_api_docs: false    # enable for shared packages
```

Three of these carry real weight:

- `always_use_package_imports` — bans `../../..` relative imports. Makes moving files safe.
- `unawaited_futures` — catches fire-and-forget async, the source of silent failures. Mark intentional ones with `unawaited(...)` from `dart:async`.
- `require_trailing_commas` — makes `dart format` produce one-argument-per-line widget trees, which makes diffs readable. In a deeply nested widget tree this is the difference between a reviewable PR and a wall.

Consider adding `custom_lint` + `riverpod_lint` once you adopt Riverpod — it catches provider misuse statically.

---

## 3.4 Dependency baseline

This is a defensible starting set for a backend-driven business app. Every entry earns its place.

```yaml
# pubspec.yaml
name: erp_mobile
description: ERP mobile client
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.12.0

dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter

  # State management + DI
  flutter_riverpod: ^3.0.0
  riverpod_annotation: ^3.0.0

  # Routing
  go_router: ^16.0.0

  # Networking
  dio: ^5.7.0
  pretty_dio_logger: ^1.4.0

  # Models / serialization
  freezed_annotation: ^3.0.0
  json_annotation: ^4.9.0

  # Storage
  flutter_secure_storage: ^9.2.0
  shared_preferences: ^2.3.0
  drift: ^2.20.0
  sqlite3_flutter_libs: ^0.5.0
  path_provider: ^2.1.0

  # Utilities
  collection: ^1.19.0
  intl: ^0.20.0
  logger: ^2.4.0
  connectivity_plus: ^6.0.0
  package_info_plus: ^8.0.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter

  build_runner: ^2.4.0
  freezed: ^3.0.0
  json_serializable: ^6.8.0
  riverpod_generator: ^3.0.0
  drift_dev: ^2.20.0

  flutter_lints: ^5.0.0
  custom_lint: ^0.7.0
  riverpod_lint: ^3.0.0

  mocktail: ^1.0.0
  golden_toolkit: ^0.15.0

flutter:
  uses-material-design: true
  generate: true          # enables gen-l10n
  assets:
    - assets/images/
```

Pin ranges with `^` and commit `pubspec.lock` for apps (not for packages). Check package health on pub.dev — look at the **Flutter Favorite** badge, the maintenance score, and last-publish date. Prefer first-party (`flutter.dev` / `dart.dev` publisher) packages where they exist.

**Deliberately not in the list:** `get` / GetX (encourages skipping architecture and hides `context`), `provider` for new code (Riverpod supersedes it), `http` (use `dio` — you need interceptors), `hive` (Drift is a better answer for relational data and you already think in SQL).

---

## 3.5 Folder structure — the official layout

Flutter has an **officially documented package structure**, published in the [architecture case study](https://docs.flutter.dev/app-architecture/case-study#package-structure) and implemented in [`compass_app`](https://github.com/flutter/samples/tree/main/compass_app), the team's reference application. This tutorial uses it. Every file-path comment in the code examples that follow matches this tree.

```
lib/
├── main.dart                       # production entry point
├── main_development.dart           # dev entry point
├── main_staging.dart               # staging entry point
│
├── config/                         # compile-time config, DI wiring
│   ├── app_config.dart
│   └── dependencies.dart
│
├── routing/
│   ├── router.dart
│   └── routes.dart
│
├── ui/                             # ← organized by FEATURE
│   ├── core/
│   │   ├── ui/                     # shared widgets (branded button, error view)
│   │   │   ├── app_error_view.dart
│   │   │   ├── app_empty_state.dart
│   │   │   └── app_button.dart
│   │   └── themes/
│   │       ├── app_theme.dart
│   │       └── app_spacing.dart
│   │
│   ├── auth/
│   │   ├── view_models/
│   │   │   └── login_viewmodel.dart
│   │   └── widgets/
│   │       ├── login_screen.dart
│   │       └── login_form.dart
│   │
│   ├── orders/
│   │   ├── view_models/
│   │   │   ├── orders_viewmodel.dart
│   │   │   └── order_detail_viewmodel.dart
│   │   └── widgets/
│   │       ├── orders_screen.dart
│   │       ├── order_detail_screen.dart
│   │       └── order_tile.dart
│   │
│   └── inventory/
│       ├── view_models/
│       └── widgets/
│
├── domain/                         # ← shared by both layers
│   └── models/
│       ├── order.dart
│       ├── session.dart
│       └── money.dart
│
├── data/                           # ← organized by TYPE
│   ├── repositories/
│   │   ├── auth/
│   │   │   ├── auth_repository.dart          # abstract
│   │   │   ├── auth_repository_remote.dart
│   │   │   └── auth_repository_dev.dart      # fake, for local dev
│   │   └── orders/
│   │       ├── orders_repository.dart
│   │       └── orders_repository_remote.dart
│   ├── services/
│   │   ├── api/
│   │   │   ├── api_client.dart               # Dio setup
│   │   │   ├── auth_interceptor.dart
│   │   │   └── orders_api_client.dart
│   │   ├── secure_storage_service.dart
│   │   └── local/
│   │       └── app_database.dart             # Drift
│   └── model/                                # API/DTO classes
│       ├── order/
│       │   └── order_dto.dart
│       └── auth/
│           └── token_pair.dart
│
└── utils/
    ├── result.dart
    ├── command.dart
    └── extensions/
        └── build_context_x.dart

test/                               # mirrors lib/ exactly
├── data/
├── domain/
├── ui/
└── utils/

testing/                            # separate subpackage: fakes + mocks
├── fakes/
└── models/
```

### Why it's a hybrid, and why that's the right call

This is the part people get wrong when they skim it. The layout is **deliberately not uniform**:

- **`data/` is organized by type.** Repositories and services aren't owned by any one feature — `OrdersRepository` is consumed by the orders list, the dashboard, and the offline sync job. Filing it under a feature forces an arbitrary choice and then an import from another feature the first time it's shared.
- **`ui/` is organized by feature.** Each feature has exactly one view and one view model, so grouping is unambiguous and vertical slices stay together.
- **`domain/` sits between them** because both layers use those types.

The moment two features share data — constantly, in an ERP — pure feature-first forces you to answer "which feature owns this repository?" The official layout removes the question.

### The naming conventions that come with it

The docs are specific about this, and consistency here is worth more than it sounds:

| Component | Name pattern | Example |
|---|---|---|
| View | `<Feature>Screen` | `OrdersScreen` |
| ViewModel | `<Feature>ViewModel` | `OrdersViewModel` |
| Repository | `<Entity>Repository` | `OrdersRepository` |
| Service | `<Thing>Service` / `<Thing>ApiClient` | `SecureStorageService`, `OrdersApiClient` |

Two explicit rules from the docs:

1. **Don't name a directory `/widgets` for shared widgets** — it collides conceptually with the SDK. Use `ui/core/ui/`.
2. **Repository classes should be abstract**, with concrete implementations per environment (`_remote`, `_dev`, `_local`). This is what lets you run the whole app against fakes without touching a network — and it's rated *strongly recommend*.

### Three details worth stealing regardless

- **Three `main_*.dart` entry points** instead of runtime flavor branching. Each wires a different `dependencies.dart` configuration. Dead simple, and it means a dev build physically cannot point at prod.
- **`testing/` as a separate subpackage** of fakes and mock models — the docs describe it as "a version of your app that you don't ship." Other packages' tests can import it.
- **`test/` mirrors `lib/` exactly.** Finding the test for a file is mechanical.

### The feature-first alternative

You will see this everywhere in the community — `features/<name>/{data,domain,presentation}` — and it's what [`very_good_cli`](https://cli.vgv.dev/) and much of the Code with Andrea material use:

```
lib/features/orders/
├── data/          # orders_api.dart, orders_repository_impl.dart, dto/
├── domain/        # order.dart, orders_repository.dart (interface)
└── presentation/  # orders_screen.dart, orders_viewmodel.dart
```

| | Official hybrid | Pure feature-first |
|---|---|---|
| Shared repository | Natural — lives in `data/repositories/` | Awkward — must pick an owner or hoist to `core/` |
| Deleting a feature | Touch 2–3 directories | Delete one directory |
| Extracting to a package later | Requires regrouping | Nearly mechanical |
| Matches official docs/samples | Yes | No |
| Onboarding a Flutter dev | Recognizable immediately | Also common, but non-canonical |

**Recommendation:** use the official layout. It's what the docs, samples, and any Flutter engineer you hire will expect, and the shared-repository problem is real in your domain. Feature-first earns its keep mainly when you're heading for a multi-package split — and at that point you're doing something different anyway (see [Part 3b](03b-project-structure-at-scale.md)).

### Rules that make either layout hold up

1. **Dependency direction is one-way:** `ui → domain ← data`. The UI never imports from `data/services/`; it goes through a repository.
2. **View models depend on repository *interfaces*,** never concrete implementations. Same inversion you use when a FastAPI service depends on a protocol rather than a concrete SQLAlchemy class.
3. **No logic in widgets.** The docs allow exactly four exceptions: simple show/hide on a flag, animation logic needing the widget, layout logic from screen size, and simple routing.
4. **One public widget per file,** named the same as the file.

### On the `domain/` layer

Flutter rates a domain layer with use-cases as **conditional**, not recommended-by-default: "in very large apps, use-cases are useful, but in most apps they add unnecessary overhead."

Start with `domain/models/` only. Add `domain/use_cases/` when a second view model needs the same orchestration logic — not before. Separate API models and domain models is likewise rated conditional ("use in large apps"); for an ERP client, you're in that bucket, which is why this tutorial keeps DTOs in `data/model/` and domain types in `domain/models/`.

---

## 3.6 Flavors and environment config

You need at minimum `dev`, `staging`, `prod` pointing at different API base URLs, with different app ids so they can coexist on one device.

**Dart side — compile-time constants:**

```dart
// lib/core/config/app_config.dart
enum Flavor { dev, staging, prod }

final class AppConfig {
  const AppConfig({
    required this.flavor,
    required this.apiBaseUrl,
    required this.enableLogging,
  });

  final Flavor flavor;
  final String apiBaseUrl;
  final bool enableLogging;

  static const _flavorName = String.fromEnvironment('FLAVOR', defaultValue: 'dev');

  static AppConfig current() => switch (_flavorName) {
        'prod' => const AppConfig(
            flavor: Flavor.prod,
            apiBaseUrl: 'https://api.example.com',
            enableLogging: false,
          ),
        'staging' => const AppConfig(
            flavor: Flavor.staging,
            apiBaseUrl: 'https://staging-api.example.com',
            enableLogging: true,
          ),
        _ => const AppConfig(
            flavor: Flavor.dev,
            apiBaseUrl: 'http://10.0.2.2:8000',   // Android emulator -> host
            enableLogging: true,
          ),
      };
}
```

Run with:

```bash
flutter run --dart-define=FLAVOR=dev --flavor dev
flutter build apk --release --dart-define=FLAVOR=prod --flavor prod
```

Better: put the defines in a JSON file and use `--dart-define-from-file=config/dev.json`, then add the file to `.gitignore` if it contains anything sensitive.

**Android side** (`android/app/build.gradle.kts`):

```kotlin
android {
    flavorDimensions += "env"
    productFlavors {
        create("dev") {
            dimension = "env"
            applicationIdSuffix = ".dev"
            resValue("string", "app_name", "ERP Dev")
        }
        create("staging") {
            dimension = "env"
            applicationIdSuffix = ".staging"
            resValue("string", "app_name", "ERP Staging")
        }
        create("prod") {
            dimension = "env"
            resValue("string", "app_name", "ERP")
        }
    }
}
```

**iOS side:** create matching Xcode schemes and build configurations (`Debug-dev`, `Release-prod`, …) with per-configuration `PRODUCT_BUNDLE_IDENTIFIER`. Flutter's `flavors-ios` doc walks through it; it's fiddly once and then stable.

> **Critical security note:** `--dart-define` values are compiled into the binary and are **recoverable by anyone who unzips your APK**. They are for *configuration*, not secrets. Never ship an API secret, signing key, or third-party private key in a Flutter app. Same rule as `NEXT_PUBLIC_*`. Anything secret stays behind your FastAPI service. See [§12.1](12-shipping.md#121-secrets-and-configuration).

---

[← Flutter Fundamentals](02-flutter-fundamentals.md) · [Index](README.md) · [Next: Architecture →](04-architecture.md)
