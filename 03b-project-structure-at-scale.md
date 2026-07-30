# Part 3b — Project Structure at Scale

[← Project Setup](03-project-setup.md) · [Index](README.md) · [Next: Architecture →](04-architecture.md)

---

> **Read this when** the single-package layout from [§3.5](03-project-setup.md#35-folder-structure-the-official-layout) starts to strain — not before. Premature modularization costs more than it saves.

## 3b.1 What folders can't do

The official structure is a **single-package** layout, and the docs scope it that way deliberately. Past roughly 15–20 features, the binding constraint stops being "where does this file go" and becomes:

**Nothing enforces the boundaries.** Any file in `lib/` can import any other file in `lib/`. `flutter analyze` will not stop a widget from importing `AppDatabase` directly. Your layering rules live entirely in code review, and code review erodes.

Symptoms that you've hit the wall:

- A view model imports a service because "it was faster than adding a repository method"
- Full test suite runs on every PR because there's no way to know what's affected
- Two teams keep colliding in `data/repositories/`
- A second app (admin console, driver app) needs 60% of the same code and you're copy-pasting
- Cold build times are hurting the inner loop

If none of those are true, stay single-package. Seriously.

---

## 3b.2 The answer: multiple packages, one repo

Since **Dart 3.6**, monorepos are a first-class Dart feature via [pub workspaces](https://dart.dev/tools/pub/workspaces) — multiple packages sharing a single dependency resolution and one lockfile.

```
erp_monorepo/
├── pubspec.yaml                 # the workspace root
├── melos.yaml                   # script orchestration
├── analysis_options.yaml        # shared lint config
│
├── apps/
│   ├── erp_mobile/              # thin shell: entry points, DI wiring, router
│   │   ├── lib/
│   │   │   ├── main.dart
│   │   │   ├── main_development.dart
│   │   │   ├── config/dependencies.dart
│   │   │   └── routing/router.dart
│   │   └── pubspec.yaml
│   └── erp_admin/               # second app, same feature packages
│
└── packages/
    ├── core_domain/             # models, Result, shared contracts
    ├── core_network/            # Dio, interceptors, ApiException
    ├── core_storage/            # Drift, secure storage
    ├── core_ui/                 # design system, theme, shared widgets
    │
    ├── feature_auth/
    ├── feature_orders/
    ├── feature_inventory/
    └── feature_reports/
```

**Each feature package keeps the official internal structure** — `ui/`, `domain/`, `data/`, `utils/`. You haven't thrown away §3.5; you've drawn a hard line around each copy of it.

Root workspace declaration:

```yaml
# pubspec.yaml (repo root)
name: erp_workspace
publish_to: none
environment:
  sdk: ^3.12.0
workspace:
  - apps/erp_mobile
  - apps/erp_admin
  - packages/core_domain
  - packages/core_network
  - packages/core_storage
  - packages/core_ui
  - packages/feature_auth
  - packages/feature_orders
  - packages/feature_inventory
  - packages/feature_reports
```

Each member opts in:

```yaml
# packages/feature_orders/pubspec.yaml
name: feature_orders
publish_to: none
resolution: workspace          # ← required for workspace membership

environment:
  sdk: ^3.12.0

dependencies:
  flutter: {sdk: flutter}
  core_domain: {path: ../core_domain}
  core_network: {path: ../core_network}
  core_ui: {path: ../core_ui}
  flutter_riverpod: ^3.0.0
```

One `dart pub get` at the root resolves everything into a single `.dart_tool/package_config.json`. No more `pubspec_overrides.yaml` juggling.

---

## 3b.3 What this buys you

| | Folders | Packages |
|---|---|---|
| Boundary enforcement | Convention + code review | **Compile error** — an undeclared dependency won't resolve |
| Dependency graph | Implicit, drifts silently | Explicit in each `pubspec.yaml`, reviewable in a PR diff |
| CI scope | Test everything, every time | Test only changed packages and their dependents |
| Build/analysis time | Whole app | Incremental per package |
| Reuse across apps | Copy-paste | `import` |
| Team ownership | `CODEOWNERS` on paths | `CODEOWNERS` on packages, with a real API surface |

That first row is the whole point. `feature_orders` physically cannot import `feature_inventory` unless someone adds it to a `pubspec.yaml` — which is a line in a diff a reviewer will see, rather than an import buried in a 400-line file.

---

## 3b.4 The dependency rule

**Feature packages depend on `core_*`. Feature packages never depend on each other.**

```
        apps/erp_mobile
        ┌──────┼──────┬─────────────┐
        ▼      ▼      ▼             ▼
  feature_  feature_  feature_   feature_
   auth     orders   inventory   reports
        └──────┴──────┴─────────────┘
                    │
        ┌───────────┼───────────┬──────────┐
        ▼           ▼           ▼          ▼
  core_domain  core_network  core_storage  core_ui
```

When orders genuinely needs something from inventory, you have three options in increasing order of cost:

1. **Hoist the shared type into `core_domain`.** Usually correct — if two features need `StockLevel`, it isn't an inventory-specific concept.
2. **Declare an interface in `core_domain`, implement it in the owning feature, wire it in the app shell.** Dependency inversion. Same move as a FastAPI service depending on a protocol.
3. **Merge the two features.** If they're this entangled, they may be one module wearing two names.

What you must not do is add `feature_inventory` to `feature_orders`'s pubspec. Do it twice and you have a cycle and a de facto monolith with extra ceremony.

**The app shell is where composition happens.** `apps/erp_mobile` depends on everything and wires it together — routes, DI container, entry points. It should contain almost no logic.

```dart
// apps/erp_mobile/lib/config/dependencies.dart
List<Override> providerOverrides(AppConfig config) => [
      apiClientProvider.overrideWithValue(ApiClient(config.apiBaseUrl)),
      ordersRepositoryProvider.overrideWith(
        (ref) => OrdersRepositoryRemote(ref.watch(ordersApiClientProvider)),
      ),
      // Swap one line for a fully faked app:
      // ordersRepositoryProvider.overrideWithValue(OrdersRepositoryDev()),
    ];
```

---

## 3b.5 Melos on top

Pub workspaces solve *resolution*. [Melos](https://pub.dev/packages/melos) solves *orchestration* — running commands across packages. Since workspaces landed, Melos delegates resolution to pub instead of generating override files.

```yaml
# melos.yaml
name: erp

scripts:
  analyze:
    run: dart analyze .
    exec: {concurrency: 5}

  test:
    run: flutter test
    packageFilters:
      dirExists: test

  gen:
    run: dart run build_runner build --delete-conflicting-outputs
    packageFilters:
      dependsOn: build_runner

  ci:
    run: melos run analyze && melos run test
```

```bash
melos bootstrap              # link everything
melos run gen                # codegen across all packages that need it
melos run test --diff=main   # only packages changed vs main — the CI win
melos version               # conventional-commit driven versioning
```

`--diff` is what turns a 12-minute CI run into a 90-second one.

---

## 3b.6 Enforcing more than pubspec can

Package boundaries stop cross-package imports. They don't stop a widget inside `feature_orders` from importing `data/services/` directly. For that, two options:

**Export barrels** — each package exposes only what it means to:

```dart
// packages/feature_orders/lib/feature_orders.dart
library feature_orders;

export 'src/ui/orders/widgets/orders_screen.dart';
export 'src/ui/orders/view_models/orders_viewmodel.dart';
export 'src/domain/models/order.dart';
// data/ and services deliberately NOT exported
```

Put everything else under `lib/src/`. Dart already warns on importing another package's `src/`, so this is enforced across package lines for free.

**Custom lints** — `custom_lint` lets you write project-specific rules (e.g. "no file under `ui/` may import `data/services/`"). Worth it above ~5 engineers; overkill below.

---

## 3b.7 A migration path that doesn't stall

Don't big-bang this. The order matters — extract leaves first, so nothing you move has outbound dependencies into the app.

1. **`core_ui`** — theme, design-system widgets. Zero business logic, so it's a safe first move and immediately proves the tooling.
2. **`core_domain`** — models, `Result`, shared contracts.
3. **`core_network` / `core_storage`** — infrastructure.
4. **One feature** — pick the most isolated one. This is where you discover the leaks; expect to spend most of the effort here.
5. **Remaining features**, one per PR.
6. **App shell shrinks** to entry points, DI, and routing.

Each step should leave `main` green and shippable. If step 4 reveals a feature that can't be extracted without dragging three others with it, that's information: those modules aren't actually separate, and you should fix the coupling before continuing.

---

## 3b.8 Related: deferred loading

Package modularization is a *source* concern. If your goal is **binary size** — shipping a large app where most users touch a fraction of it — that's [deferred components](https://docs.flutter.dev/perf/deferred-components), which split Dart code into dynamically-downloaded modules (Android Play Feature Delivery; on web, deferred imports work broadly).

```dart
import 'package:feature_reports/feature_reports.dart' deferred as reports;

Future<void> openReports() async {
  await reports.loadLibrary();
  // ...
}
```

Different problem, different tool. Packages give you *maintainability*; deferred components give you *download size*. They compose well but neither substitutes for the other.

---

## 3b.9 The honest summary

| Team / app size | Structure |
|---|---|
| Solo, < 15 features | Official single-package layout ([§3.5](03-project-setup.md#35-folder-structure-the-official-layout)). Nothing more. |
| Small team, one app, growing | Same, plus strict lints and `lib/src/` discipline |
| Multiple apps, or 3+ teams | Pub workspaces + feature packages + Melos |
| Huge app, size-constrained | The above, plus deferred components |

The layer discipline from [Part 4](04-architecture.md) is what makes the jump cheap whenever you make it. Get that right and moving folders into packages is mechanical; get it wrong and no directory structure will save you.

---

[← Project Setup](03-project-setup.md) · [Index](README.md) · [Next: Architecture →](04-architecture.md)
