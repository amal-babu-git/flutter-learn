# Part 2 — Flutter Fundamentals

[← Dart Crash Course](01-dart-crash-course.md) · [Index](README.md) · [Next: Project Setup →](03-project-setup.md)

---

## 2.1 The core thesis

Flutter's central claim: **`UI = f(state)`**, and `f` is cheap enough to run 60–120 times a second.

You never mutate the UI. You mutate state and let the framework recompute the description of the UI. This is React's thesis too, so it's familiar — but Flutter takes it further because it controls the renderer, so there's no DOM to reconcile against, only its own retained object graph.

```dart
import 'package:flutter/material.dart';

void main() => runApp(const App());

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'ERP Mobile',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0B5FFF)),
        useMaterial3: true,
      ),
      home: const HomeScreen(),
    );
  }
}
```

`runApp` attaches the widget to the root of the tree and starts the frame pipeline. `MaterialApp` bootstraps navigation, theming, localization, and the overlay stack — it's the closest thing to your Next.js `_app.tsx` / root layout.

---

## 2.2 Three trees

This is the concept that separates people who *use* Flutter from people who *understand* it. Flutter maintains three parallel trees:

```
Widget tree          Element tree            RenderObject tree
(immutable config)   (mutable, long-lived)   (layout + paint)
     |                      |                        |
  recreated              persists                 persists
  every build          across builds            across builds
```

- **Widget**: an immutable description. `Container(color: red)`. Cheap to allocate, thrown away every frame. Think: the *props*.
- **Element**: the instantiation of a widget at a location in the tree. Holds the `State` for stateful widgets. Persists across rebuilds as long as the widget's `runtimeType` and `key` match. Think: the *component instance / fiber*.
- **RenderObject**: does layout, painting, and hit testing. Expensive. Reused aggressively.

**The reconciliation rule** — memorize this:

> When rebuilding, for each position Flutter compares the new widget to the old one. If `runtimeType` **and** `key` both match, it **updates** the existing Element (and its State survives). Otherwise it **deactivates** the old Element (State is destroyed) and inflates a new one.

Two consequences that explain 90% of confusing Flutter behaviour:

1. **Calling `setState` on a big subtree is not expensive** *per se*. Rebuilding widgets is allocation of small immutable objects; the costly render objects are reused. This is why "just rebuild" is idiomatic.
2. **Reordering a list without keys destroys state.** Position determines identity by default. See [§2.7](#27-keys-when-identity-matters).

You can see all three trees in the Flutter Inspector ([§11.4](11-performance.md#114-devtools-workflow)). Spend twenty minutes there early — it makes the model concrete.

---

## 2.3 Stateless vs Stateful

```dart
class OrderTile extends StatelessWidget {
  const OrderTile({super.key, required this.order, this.onTap});

  final Order order;
  final VoidCallback? onTap;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(order.number),
      subtitle: Text(order.customerName),
      trailing: Text(order.total.toString()),
      onTap: onTap,
    );
  }
}
```

Stateless = pure function of its constructor arguments plus inherited context. **All fields must be `final`.** Note `const` constructor + `super.key` — both are standard boilerplate and both matter (const enables canonicalization; key enables identity control).

Stateful splits into two classes because the widget is immutable but the state must survive rebuilds:

```dart
class SearchField extends StatefulWidget {
  const SearchField({super.key, required this.onQuery});
  final ValueChanged<String> onQuery;

  @override
  State<SearchField> createState() => _SearchFieldState();
}

class _SearchFieldState extends State<SearchField> {
  late final TextEditingController _controller;
  Timer? _debounce;

  @override
  void initState() {
    super.initState();
    _controller = TextEditingController()..addListener(_onChanged);
  }

  @override
  void didUpdateWidget(covariant SearchField oldWidget) {
    super.didUpdateWidget(oldWidget);
    // Parent rebuilt with new config. React to changed props here.
  }

  void _onChanged() {
    _debounce?.cancel();
    _debounce = Timer(const Duration(milliseconds: 300),
        () => widget.onQuery(_controller.text));
  }

  @override
  void dispose() {
    _debounce?.cancel();
    _controller.dispose();     // ALWAYS
    super.dispose();
  }

  @override
  Widget build(BuildContext context) =>
      TextField(controller: _controller);
}
```

**The `State` lifecycle**, mapped to React:

| Flutter | React equivalent | Notes |
|---|---|---|
| `createState()` | — | Called once when the Element is created. |
| `initState()` | `useEffect(fn, [])` | `context` is available but *inherited widgets are not safe to use yet*. |
| `didChangeDependencies()` | — | Called after `initState` and whenever an inherited widget this State depends on changes. Safe place for `Theme.of` / provider reads that need to react. |
| `build()` | render | Must be pure and fast. No I/O, no `setState`. |
| `didUpdateWidget(old)` | `componentDidUpdate` | Parent rebuilt with new configuration. |
| `setState(fn)` | `setState` | Marks dirty; schedules a rebuild. Never call in `build` or after `dispose`. |
| `deactivate()` | — | Removed from tree, may be reinserted (e.g. `GlobalKey` reparenting). |
| `dispose()` | `useEffect` cleanup | Release controllers, subscriptions, timers, focus nodes. |

**Rules that will save you hours:**

- Access widget properties via `widget.foo`, never copy them into `State` fields (they go stale).
- Never `await` then `setState` without checking `if (!mounted) return;` — the widget may be gone.
- Anything with a `dispose()` method (`TextEditingController`, `AnimationController`, `ScrollController`, `FocusNode`, `StreamSubscription`) must be disposed.

**When to use Stateful at all:** only for genuinely *ephemeral, widget-local* state — animation controllers, text controllers, scroll position, whether a dropdown is open. Everything that outlives the widget or is shared belongs in a view model ([Part 4](04-architecture.md), [Part 5](05-state-management.md)).

---

## 2.4 BuildContext is not `this`

`BuildContext` **is** the `Element`. It is a handle to a *location in the tree*, and its main job is upward lookup:

```dart
Theme.of(context)          // nearest Theme ancestor
MediaQuery.sizeOf(context) // nearest MediaQuery (size-only, fewer rebuilds)
Navigator.of(context)      // nearest Navigator
ScaffoldMessenger.of(context).showSnackBar(...)
```

These are `InheritedWidget` lookups. The Element keeps a hash map of inherited widget types up the tree, so `.of(context)` is O(1) — not a walk. It also *registers a dependency*: when that inherited widget changes, this element rebuilds. That is the entire Context API mechanism, and it's what Provider/Riverpod are built on.

**The classic beginner error:**

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    body: ElevatedButton(
      // WRONG: `context` here is above the Scaffold, so this throws.
      onPressed: () => Scaffold.of(context).openDrawer(),
      child: const Text('Menu'),
    ),
  );
}
```

Fix with a `Builder`, which introduces a new context below the `Scaffold`:

```dart
body: Builder(
  builder: (innerContext) => ElevatedButton(
    onPressed: () => Scaffold.of(innerContext).openDrawer(),
    child: const Text('Menu'),
  ),
),
```

**Prefer the `sizeOf`/`platformBrightnessOf` style accessors** over `MediaQuery.of(context)`. The latter subscribes you to *every* MediaQuery change (including keyboard insets), causing needless rebuilds.

**Async gap rule:** after an `await`, a `BuildContext` may be defunct. The lint `use_build_context_synchronously` catches this. Guard with `if (!context.mounted) return;`.

---

## 2.5 Layout: constraints go down, sizes go up

Flutter's layout algorithm is a single pass, and it is genuinely simple once stated:

> **Constraints go down. Sizes go up. Parent sets position.**

Each parent passes a `BoxConstraints` (min/max width and height) to each child. Each child picks its own size *within* those constraints and reports it up. The parent then positions the child. A widget can't know its own position, and a parent can't dictate an exact size — only constrain.

This has one famous implication: **a widget cannot be "as big as its child wants" and "as big as the parent allows" at the same time**. Most layout errors come from a widget receiving *unbounded* constraints in one axis.

```dart
// THROWS: Column gives unbounded height to children; ListView wants infinite height.
Column(children: [ListView(children: [...])]);

// FIX 1: give it bounded height.
Column(children: [SizedBox(height: 300, child: ListView(...))]);

// FIX 2: let it take remaining space.
Column(children: [Expanded(child: ListView(...))]);

// FIX 3: let the inner list size to content (only for small lists).
Column(children: [
  ListView(shrinkWrap: true, physics: const NeverScrollableScrollPhysics(), children: [...]),
]);
```

**Flex layout** (`Row`/`Column`) is CSS flexbox with different names:

| CSS flex | Flutter |
|---|---|
| `flex-direction: row` | `Row` |
| `justify-content` | `mainAxisAlignment` |
| `align-items` | `crossAxisAlignment` |
| `flex: 1` | `Expanded(child: ...)` |
| `flex-shrink` behaviour | `Flexible(fit: FlexFit.loose)` |
| `gap` | `spacing:` on Row/Column (3.27+), or `Wrap`'s `spacing` |

Core layout widgets, ranked by how often you'll reach for them:

`Column` · `Row` · `Expanded` · `Padding` · `SizedBox` · `Center` · `Stack` + `Positioned` · `Container` · `Align` · `Wrap` · `SingleChildScrollView` · `ListView` · `GridView` · `AspectRatio` · `FittedBox` · `IntrinsicHeight` (avoid — expensive) · `LayoutBuilder` (gives you the incoming constraints; the closest thing to a container query) · `CustomMultiChildLayout` (escape hatch).

**Debugging layout:** `debugPaintSizeEnabled = true` draws every box. The Flutter Inspector's "Layout Explorer" shows constraints and flex factors visually and is the fastest way to understand a broken layout. When you see `RenderFlex overflowed by N pixels`, read it as "a child demanded more space than its constraint allowed" and go find which axis is unbounded.

---

## 2.6 The widgets you will actually use

You do not need to memorize the widget catalog. This is the working set for a business app:

**Structure:** `Scaffold` (app bar / body / FAB / drawer / bottom nav), `AppBar`, `SafeArea`, `Drawer`, `NavigationBar`, `NavigationRail`, `TabBar` + `TabBarView`.

**Display:** `Text`, `RichText`, `Icon`, `Image.network` / `Image.asset`, `CachedNetworkImage` (package), `CircleAvatar`, `Card`, `Divider`, `Chip`, `Badge`, `Tooltip`.

**Input:** `TextField` / `TextFormField`, `DropdownButtonFormField`, `Checkbox`, `Radio`, `Switch`, `Slider`, `DatePicker` (`showDatePicker`), `Form` + `GlobalKey<FormState>`, `Autocomplete`.

**Action:** `ElevatedButton`, `FilledButton`, `OutlinedButton`, `TextButton`, `IconButton`, `InkWell` (ripple + tap), `GestureDetector` (raw gestures, no ripple), `Dismissible`.

**Feedback:** `CircularProgressIndicator`, `LinearProgressIndicator`, `SnackBar` via `ScaffoldMessenger`, `AlertDialog` via `showDialog`, `showModalBottomSheet`, `Skeletonizer` (package) for loading skeletons.

**Scrolling:** `ListView.builder` (lazy — use this, not `ListView(children: [...])`, for anything over ~20 items), `ListView.separated`, `GridView.builder`, `CustomScrollView` + slivers (`SliverAppBar`, `SliverList`, `SliverToBoxAdapter`) when you need collapsing headers or mixed scroll content, `RefreshIndicator` for pull-to-refresh.

**Conditional/animated:** `AnimatedSwitcher`, `AnimatedContainer`, `AnimatedOpacity`, `Hero`, `TweenAnimationBuilder`. Implicit animations (`Animated*`) cover most needs without an `AnimationController`.

**A note on `Container`:** it's a convenience wrapper around `Padding` + `Align` + `DecoratedBox` + `ConstrainedBox` + `Transform`. It's fine, but if you only need padding, use `Padding` — it's clearer and marginally cheaper.

**Use `const` everywhere it compiles.** `const Text('Save')` is allocated once for the program's lifetime and short-circuits rebuilds. Enable the `prefer_const_constructors` lint and let the IDE add them.

---

## 2.7 Keys: when identity matters

Default identity is (position, runtimeType). Keys override that.

You need keys when **widgets of the same type are reordered, inserted, or removed from a list, and they hold state**.

```dart
// Without keys: deleting item 0 makes item 1's State attach to item 0's widget.
// Scroll offsets, animations, TextField contents all jump to the wrong row.
ListView(
  children: [for (final o in orders) OrderRow(key: ValueKey(o.id), order: o)],
);
```

Key types:

- `ValueKey(id)` — identity from a value. Your default.
- `ObjectKey(obj)` — identity from object reference.
- `UniqueKey()` — always different; forces a full rebuild. Useful to intentionally reset state (e.g. reset a form).
- `GlobalKey` — unique app-wide; lets you access a `State` from outside (`_formKey.currentState!.validate()`) and lets a widget move in the tree without losing state. **Expensive — use sparingly.** Common legitimate use: `GlobalKey<FormState>`, `GlobalKey<ScaffoldMessengerState>`.

If you're wondering "do I need a key here?", the answer is: only if state is being preserved incorrectly. Stateless rows in a static list don't need them.

---

## 2.8 Theming and design systems

Define your design system once, consume via `Theme.of(context)`. Never hardcode colors in widgets — same discipline as CSS custom properties or a Tailwind theme.

```dart
// lib/ui/core/themes/app_theme.dart
abstract final class AppTheme {
  static ThemeData light() => _base(Brightness.light);
  static ThemeData dark() => _base(Brightness.dark);

  static ThemeData _base(Brightness brightness) {
    final scheme = ColorScheme.fromSeed(
      seedColor: const Color(0xFF0B5FFF),
      brightness: brightness,
    );
    return ThemeData(
      useMaterial3: true,
      colorScheme: scheme,
      scaffoldBackgroundColor: scheme.surface,
      textTheme: const TextTheme(
        titleLarge: TextStyle(fontSize: 22, fontWeight: FontWeight.w600),
        bodyMedium: TextStyle(fontSize: 14, height: 1.4),
      ),
      inputDecorationTheme: InputDecorationTheme(
        border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
        filled: true,
      ),
      filledButtonTheme: FilledButtonThemeData(
        style: FilledButton.styleFrom(
          minimumSize: const Size.fromHeight(48),
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
        ),
      ),
    );
  }
}
```

For values Material doesn't model (spacing scale, custom brand colors), use a `ThemeExtension`:

```dart
@immutable
class AppSpacing extends ThemeExtension<AppSpacing> {
  const AppSpacing({this.xs = 4, this.sm = 8, this.md = 16, this.lg = 24});
  final double xs, sm, md, lg;

  @override
  AppSpacing copyWith({double? xs, double? sm, double? md, double? lg}) =>
      AppSpacing(xs: xs ?? this.xs, sm: sm ?? this.sm,
                 md: md ?? this.md, lg: lg ?? this.lg);

  @override
  AppSpacing lerp(AppSpacing? other, double t) => other ?? this;
}

// Register: ThemeData(extensions: [const AppSpacing()])
// Read:     Theme.of(context).extension<AppSpacing>()!.md
```

Given the 3.44 unbundling of Material/Cupertino, keeping all styling behind a single `AppTheme` + extensions layer also insulates you from the eventual `material_ui` package migration.

---

## 2.9 Async in the widget tree

Two built-in widgets bridge async data into the tree. They're fine for leaf-level, throwaway async — but for anything a repository owns, prefer a view model / provider ([Part 5](05-state-management.md)), because `FutureBuilder` re-triggers its future on every rebuild if you construct it inline.

```dart
// Acceptable: future created once, in initState.
class _OrderDetailState extends State<OrderDetail> {
  late final Future<Order> _future = repo.fetch(widget.id);

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<Order>(
      future: _future,
      builder: (context, snapshot) => switch (snapshot) {
        AsyncSnapshot(connectionState: ConnectionState.waiting) =>
          const Center(child: CircularProgressIndicator()),
        AsyncSnapshot(hasError: true, :final error) =>
          ErrorView(error: error!),
        AsyncSnapshot(:final data?) => OrderView(order: data),
        _ => const SizedBox.shrink(),
      },
    );
  }
}
```

```dart
// ANTI-PATTERN — new future on every rebuild, infinite spinner loops.
FutureBuilder(future: repo.fetch(id), builder: ...)
```

`StreamBuilder` has the same shape for streams. `ValueListenableBuilder` is the lightest-weight reactive primitive in the framework and is worth knowing:

```dart
final _busy = ValueNotifier<bool>(false);

ValueListenableBuilder<bool>(
  valueListenable: _busy,
  builder: (context, busy, child) => busy
      ? const CircularProgressIndicator()
      : child!,
  child: const SaveButton(),   // `child` is built once and passed through
);
```

That `child` parameter is the standard trick for hoisting an expensive, non-reactive subtree out of a rebuilding builder. You'll see it in `AnimatedBuilder` too.

---

[← Dart Crash Course](01-dart-crash-course.md) · [Index](README.md) · [Next: Project Setup →](03-project-setup.md)
