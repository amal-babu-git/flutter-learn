# Part 11 — Performance

[← Testing](10-testing.md) · [Index](README.md) · [Next: Ship It →](12-shipping.md)

---

## 11.1 Build modes and Impeller

| Mode | Compilation | Use |
|---|---|---|
| Debug | JIT + assertions + observatory | Development. **Never benchmark in debug** — it's 10–50× slower. |
| Profile | AOT + tracing | Performance measurement. `flutter run --profile` on a **real device**. |
| Release | AOT, tree-shaken, no assertions | Production. |

As of 3.44, **Impeller is default on Android 10+** (Skia removed there) and on iOS. Impeller precompiles shaders, which eliminated the "first-run jank on a new animation" problem that plagued Flutter for years. If you read old advice about `--purge-persistent-cache` or SkSL shader warm-up, it no longer applies.

**Frame budget:** 16.6 ms at 60 Hz, 8.3 ms at 120 Hz.

Two threads matter — the **UI thread** (your Dart, layout, build) and the **raster thread** (GPU work). DevTools tells you which one is over budget, and the fix differs completely:

- UI-thread jank → too much Dart work per frame
- Raster-thread jank → expensive painting (opacity layers, `saveLayer`, huge images, blurs)

---

## 11.2 Rebuild discipline

The three rules, in order of impact:

### 1. `const` everything you can

A `const` widget subtree is skipped entirely during rebuild. Free.

### 2. Scope rebuilds narrowly

Don't `setState` at the top of a screen for a change that affects one badge. Push state down to the smallest widget that needs it, or use a targeted builder.

```dart
// BAD: whole screen rebuilds when the counter changes.
class Screen extends StatefulWidget {}
class _ScreenState extends State<Screen> {
  int count = 0;
  @override
  Widget build(context) => Column(children: [
        const ExpensiveHeader(),
        Text('$count'),
        ExpensiveList(items: items),
      ]);
}

// GOOD: only the Text rebuilds.
final _count = ValueNotifier(0);
Column(children: [
  const ExpensiveHeader(),
  ValueListenableBuilder(valueListenable: _count, builder: (_, c, __) => Text('$c')),
  ExpensiveList(items: items),
]);
```

With Riverpod, the equivalent is `ref.watch(provider.select((s) => s.field))` — you only rebuild when that field changes.

### 3. Hoist non-reactive subtrees

Use the `child` parameter of `AnimatedBuilder` / `ValueListenableBuilder` / `ListenableBuilder`.

### Other high-value habits

- `ListView.builder`, never `ListView(children: [...200 items])`.
- Add `itemExtent` or `prototypeItem` to lists with uniform row heights — it lets the sliver skip measurement.
- Avoid `Opacity` in animations; use `AnimatedOpacity`, `FadeTransition`, or bake alpha into the color. `Opacity` forces a `saveLayer`.
- `RepaintBoundary` around a small, frequently-repainting widget (a progress ring, a live chart) isolates its repaints from the rest of the tree. Don't scatter them — each one is a texture.
- **Size images to their display size.** Use `cacheWidth`/`cacheHeight` on `Image` so a 4000 px photo isn't decoded at full resolution into a 100 px avatar. This is the single most common memory problem in real apps.

---

## 11.3 Isolates and heavy work

Anything synchronous over ~8 ms belongs off the UI isolate:

```dart
// One-shot.
final report = await Isolate.run(() => buildMonthlyReport(rows));

// Flutter's ergonomic wrapper (top-level or static function required).
final orders = await compute(parseOrders, jsonBody);
```

For a long-lived worker (continuous processing, a sync engine), spawn an isolate with a `ReceivePort`/`SendPort` pair, or use `package:worker_manager`.

> Objects sent between isolates are **copied** (except for a few transferable types), so passing a 50 MB list has real cost — send the raw string and parse on the other side, not the parsed objects back.

**Do not** isolate network I/O. It's already asynchronous and non-blocking; an isolate adds overhead with no benefit.

Drift can run its database on a background isolate (`driftWorkerIsolate`), which keeps heavy queries off the UI thread — worth doing for large local datasets.

---

## 11.4 DevTools workflow

```bash
flutter run --profile
# then open the DevTools URL printed in the console
```

The tabs, and what each is actually for:

- **Flutter Inspector** — see the widget/element tree, select-widget-mode to jump from pixel to code, and the **Layout Explorer** for flex/constraint debugging. Start here for any layout confusion.
- **Performance** — the frame chart. Bars over the budget line are jank. Click one, read the timeline, see whether UI or raster is the culprit. "Track widget builds" shows which widgets rebuild per frame.
- **CPU Profiler** — flame chart of Dart execution. Find the function eating your frame.
- **Memory** — heap snapshots and diffs. The workflow for leak-hunting: snapshot, navigate to a screen and back ten times, snapshot again, diff. If `_OrderDetailScreenState` instances accumulate, you have a leaked subscription or a retained closure.
- **Network** — every Dio request with timing and payloads. Your browser devtools network tab.
- **Logging** — structured `dart:developer` logs.

Two toggles to know: `debugPaintSizeEnabled` (draws every layout box) and "Highlight repaints" (flashes repainting layers — anything flashing that shouldn't be is a `RepaintBoundary` candidate).

> **Method:** measure first, on a real mid-range device in profile mode. A Pixel 8 Pro will hide problems that a ₹12,000 Android tablet on a warehouse floor will not — and that tablet is your actual target.

---

[← Testing](10-testing.md) · [Index](README.md) · [Next: Ship It →](12-shipping.md)
