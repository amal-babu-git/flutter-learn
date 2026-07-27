# Part 1 — Dart Crash Course

[← Orientation](00-orientation.md) · [Index](README.md) · [Next: Flutter Fundamentals →](02-flutter-fundamentals.md)

---

## 1.1 The 5-minute orientation

Dart is a garbage-collected, class-based, statically typed language with sound null safety. It compiles two ways: **JIT** during development (enabling hot reload) and **AOT** to native machine code for release.

Read this and you know 60% of Dart:

```dart
// A top-level function. `main` is the entrypoint.
void main() {
  // Type inference: `name` is String.
  final name = 'Amal';

  // Explicit type. Non-nullable by default.
  int count = 3;

  // Nullable requires `?`.
  String? maybe;

  // String interpolation.
  print('Hello $name, count=$count, maybe=${maybe ?? "none"}');

  // Collections.
  final users = <String>['amal', 'dev'];
  final ages = <String, int>{'amal': 30};

  // Cascade: call several methods on the same object.
  final buffer = StringBuffer()
    ..write('a')
    ..write('b');

  // Arrow function for single expressions.
  int double_(int x) => x * 2;

  // Async is await/async, like Python and JS.
  Future<void> load() async {
    final result = await fetch();
    print(result);
  }
}

Future<String> fetch() async => 'data';
```

Notable differences from TypeScript/Python that bite early:

- **No truthiness.** `if (someString)` is a compile error. Conditions must be `bool`. This is a feature.
- **`final` vs `const`:** `final` = assigned once at runtime. `const` = known at compile time. `const` is load-bearing in Flutter ([§2.6](02-flutter-fundamentals.md#26-the-widgets-you-will-actually-use)).
- **Everything is an object**, including `int`, `null`, and functions. `int` is a real class with methods.
- **No `undefined`.** Only `null`.
- **Private is `_`-prefixed and library-scoped**, not class-scoped. `_foo` is visible to everything in the same file. There is no `private`/`public` keyword.
- **No function overloading.** Use optional/named parameters.

---

## 1.2 Type system and sound null safety

Dart 3 is **100% soundly null safe**. "Sound" means the compiler can *prove* a non-nullable variable is never null, so the AOT compiler removes null checks entirely. This is stronger than TypeScript's `strictNullChecks`, which can be defeated by `any` and by untyped runtime data.

```dart
String a = 'hi';      // can never be null
String? b;            // may be null, defaults to null

// Compile error: b may be null.
// print(b.length);

// Options, in increasing order of "don't":
print(b?.length);          // null-aware access -> int?
print(b?.length ?? 0);     // null-coalescing -> int
print(b!.length);          // bang: assert non-null. Throws if wrong.

// Flow analysis (promotion) — the idiomatic way:
if (b != null) {
  print(b.length);         // b promoted to String here
}
```

**Promotion has one big gotcha:** it does not apply to non-`final` *instance fields*, because another isolate/closure could theoretically mutate them between the check and the use.

```dart
class Order {
  String? note;

  void bad() {
    if (note != null) {
      // Compile error: `note` is not promoted (mutable field).
      // print(note.length);
    }
  }

  void good() {
    final n = note;         // copy to a local
    if (n != null) {
      print(n.length);      // local promotes fine
    }
  }
}
```

The `final n = x;` copy-to-local idiom is everywhere in real Dart. Get comfortable with it.

**`late`** defers initialization while keeping the type non-nullable:

```dart
class ApiClient {
  late final Dio _dio;              // set in constructor body / init
  late final String baseUrl = _computeBaseUrl();  // lazy: runs on first read
}
```

`late` without an initializer throws `LateInitializationError` if read before assignment. Use it for dependency-injected fields and for `State` fields initialized in `initState`. Do not use it to dodge thinking about nullability.

**Type hierarchy in one paragraph:** `Object?` is the top type. `Object` is everything non-null. `Never` is the bottom type (a function returning `Never` never returns — useful for `throw` helpers and exhaustive switches). `dynamic` disables static checking — treat it like TypeScript's `any` and confine it to JSON boundaries.

---

## 1.3 Variables, finality, immutability

```dart
var mutable = 1;          // inferred int, reassignable
final once = 1;           // assigned once, runtime value
const compileTime = 1;    // compile-time constant

final list = [1, 2];
list.add(3);              // OK — `final` freezes the binding, not the object

const frozen = [1, 2];
// frozen.add(3);         // runtime error — const collections are deeply immutable
```

**Style rule adopted by the whole ecosystem:** prefer `final` by default; use `var` only when you genuinely reassign. The linter rule `prefer_final_locals` enforces it ([§3.3](03-project-setup.md#33-strict-analysis-options)).

`const` constructors create **canonicalized** objects: two `const Text('hi')` expressions with the same arguments are the *same object* in memory. Flutter exploits this to skip rebuilding entire subtrees. Sprinkling `const` is the cheapest performance win in Flutter, and the linter will tell you where.

---

## 1.4 Functions

```dart
// Positional required.
int add(int a, int b) => a + b;

// Positional optional, with default.
String greet(String name, [String greeting = 'Hello']) => '$greeting, $name';

// Named — the Flutter default style. `required` is a keyword.
Widget card({
  required String title,
  String? subtitle,
  int elevation = 1,
}) { /* ... */ }

// Calling named:
card(title: 'Invoice', elevation: 2);

// Functions are first-class; typedef for readability.
typedef OrderPredicate = bool Function(Order order);

List<Order> where(List<Order> src, OrderPredicate test) =>
    src.where(test).toList();

// Closures capture lexically, like JS.
Function counter() {
  var n = 0;
  return () => ++n;
}
```

**Why Flutter uses named parameters almost exclusively:** widget constructors have many optional parameters and call sites are deeply nested. Named parameters make the tree readable and make adding a parameter non-breaking.

**Dot shorthands (Dart 3.9+)** let you omit the enum/class name where the type is known — you will see this in newer code:

```dart
// Instead of MainAxisAlignment.center:
Row(mainAxisAlignment: .center, children: [...]);
```

---

## 1.5 Collections

```dart
final list = <int>[1, 2, 3];
final set = <String>{'a', 'b'};
final map = <String, int>{'a': 1};

// Spread and null-aware spread.
final more = [0, ...list, ...?maybeNullList];

// Collection-if and collection-for — used constantly in widget trees.
final children = [
  const Header(),
  if (isAdmin) const AdminPanel(),
  for (final o in orders) OrderTile(order: o),
];

// Functional pipeline. Note: these are lazy `Iterable`s — call .toList().
final totals = orders
    .where((o) => o.status == OrderStatus.paid)
    .map((o) => o.total)
    .fold<double>(0, (sum, t) => sum + t);
```

`collection-if` and `collection-for` are the reason Flutter doesn't need a template language. Conditional and repeated UI is just a list literal.

For heavier collection work, the first-party `package:collection` gives you `groupBy`, `firstWhereOrNull`, `sortedBy`, and deep equality (`ListEquality`, `MapEquality`).

---

## 1.6 Classes and constructors

```dart
class Money {
  final int cents;
  final String currency;

  // Initializing-formal shorthand: `this.x` assigns the field directly.
  const Money(this.cents, {this.currency = 'INR'});

  // Named constructor.
  const Money.zero() : cents = 0, currency = 'INR';

  // Factory: can return a cached/subtype instance, or run logic first.
  factory Money.parse(String raw) {
    final v = (double.parse(raw) * 100).round();
    return Money(v);
  }

  // Getter — no parentheses at call site.
  double get amount => cents / 100;

  // Operator overloading.
  Money operator +(Money other) {
    assert(currency == other.currency);
    return Money(cents + other.cents, currency: currency);
  }

  // Value equality is manual in plain Dart. This is why we use `freezed`.
  @override
  bool operator ==(Object other) =>
      other is Money && other.cents == cents && other.currency == currency;

  @override
  int get hashCode => Object.hash(cents, currency);

  @override
  String toString() => '$currency ${amount.toStringAsFixed(2)}';
}
```

**Key thing to internalize:** Dart classes have *reference* equality by default. Two identical-looking objects are not `==`. This matters enormously for Flutter, because state comparison drives rebuilds. In practice you never hand-write `==`/`hashCode` — you use `freezed` ([§6.4](06-networking-auth-data.md#64-dtos-models-and-serialization)), which generates them along with `copyWith` and JSON.

Inheritance is single, with `extends`; interfaces are implicit (`implements` any class); `mixin ... on` gives composable behaviour ([§1.10](#110-mixins-extensions-generics)).

---

## 1.7 Class modifiers and sealed hierarchies

Dart 3 added modifiers that let you express intent the compiler enforces:

| Modifier | Meaning |
|---|---|
| `abstract` | Cannot be instantiated. |
| `base` | Can be extended, cannot be implemented outside the library. Guarantees subtypes inherit your implementation. |
| `interface` | Can be implemented, cannot be extended outside the library. |
| `final` | Cannot be extended or implemented outside the library. Closes the hierarchy. |
| `sealed` | Implicitly `abstract` + `final`, and the compiler **knows all direct subtypes** — enabling exhaustiveness checking. |
| `mixin` | Declares a mixin. |

`sealed` is the important one. It gives you algebraic data types:

```dart
sealed class SyncState {
  const SyncState();
}

final class SyncIdle extends SyncState {
  const SyncIdle();
}

final class SyncRunning extends SyncState {
  const SyncRunning(this.progress);
  final double progress;
}

final class SyncFailed extends SyncState {
  const SyncFailed(this.error);
  final Object error;
}

// The compiler verifies exhaustiveness. Add a subtype -> this stops compiling.
String describe(SyncState s) => switch (s) {
      SyncIdle() => 'Idle',
      SyncRunning(:final progress) => 'Syncing ${(progress * 100).toInt()}%',
      SyncFailed(:final error) => 'Failed: $error',
    };
```

This is the Dart equivalent of a Rust `enum` + `match`, or a discriminated union in TypeScript — except exhaustiveness is checked without you writing a `never` assertion. **Use sealed classes for every finite state machine in your app**: request state, auth state, sync state, form state. It eliminates an entire category of "I forgot to handle the error case" bug at compile time.

Dart also has plain `enum`s, which since Dart 2.17 are enhanced (fields, methods, constructors):

```dart
enum OrderStatus {
  draft('Draft', false),
  submitted('Submitted', true),
  paid('Paid', true),
  cancelled('Cancelled', false);

  const OrderStatus(this.label, this.isFinalized);
  final String label;
  final bool isFinalized;

  static OrderStatus fromApi(String v) =>
      values.firstWhere((e) => e.name == v, orElse: () => draft);
}
```

Use `enum` for a fixed set of *tags* with no payload; use `sealed` when variants carry different data.

---

## 1.8 Records

Records are anonymous, immutable, structurally typed tuples. They solve "I need to return two things and a class is overkill."

```dart
// Positional record.
(int, String) parse() => (200, 'OK');
final (code, message) = parse();

// Named fields — far more readable, use these.
({int total, int page}) pageInfo() => (total: 120, page: 3);
final info = pageInfo();
print(info.total);

// Records have structural value equality for free.
print((1, 'a') == (1, 'a')); // true

// Great as compound map keys.
final cache = <({String tenant, int id}), Order>{};
cache[(tenant: 'acme', id: 7)] = order;
```

Records are **not** a replacement for domain models. Rule: if the shape crosses a layer boundary or has a name in your domain language (`Order`, `Invoice`), make it a class. If it's a local, ephemeral grouping (a function returning `(value, hasMore)`), use a record.

---

## 1.9 Patterns and destructuring

Patterns appear in six places: variable declarations, assignments, `switch` statements, `switch` expressions, `if-case`, and `for-in`.

```dart
// Destructuring declaration.
final Order(:id, :total, customer: Customer(name: customerName)) = order;

// Switch expression with guards.
final tier = switch (total) {
  < 1000 => 'bronze',
  >= 1000 && < 10000 => 'silver',
  _ => 'gold',
};

// if-case: pattern match a single case without a full switch.
if (response case {'data': final List items, 'next': final String? next}) {
  // items and next are typed and bound only inside this block
}

// List patterns with rest.
switch (segments) {
  case ['orders', final id]:         handleOrder(id);
  case ['orders', final id, ...final rest]: handleSub(id, rest);
  case []:                            handleRoot();
}

// Destructuring in for-in over map entries.
for (final MapEntry(:key, :value) in headers.entries) {
  print('$key: $value');
}
```

**Where this pays off in Flutter:** parsing untyped JSON safely, exhaustively rendering sealed state, and pulling fields out of records in one line. The `if (x case ...)` form in particular replaces a lot of defensive `is` checks and casts.

---

## 1.10 Mixins, extensions, generics

**Mixins** — reusable behaviour without inheritance. Flutter uses them heavily (`SingleTickerProviderStateMixin`, `WidgetsBindingObserver`).

```dart
mixin Loggable {
  String get logTag => runtimeType.toString();
  void log(String msg) => print('[$logTag] $msg');
}

// `on` constrains where the mixin can be applied.
mixin Retryable on ApiService {
  Future<T> retry<T>(Future<T> Function() op, {int attempts = 3}) async {
    for (var i = 0; i < attempts; i++) {
      try {
        return await op();
      } catch (_) {
        if (i == attempts - 1) rethrow;
        await Future<void>.delayed(Duration(milliseconds: 200 * (1 << i)));
      }
    }
    throw StateError('unreachable');
  }
}

class OrderService extends ApiService with Loggable, Retryable {}
```

**Extensions** — add methods to types you don't own. This is C#-style extension methods, resolved statically.

```dart
extension StringX on String {
  bool get isBlank => trim().isEmpty;
  String get titleCase =>
      isEmpty ? this : this[0].toUpperCase() + substring(1);
}

extension BuildContextX on BuildContext {
  ThemeData get theme => Theme.of(this);
  ColorScheme get colors => Theme.of(this).colorScheme;
  Size get screenSize => MediaQuery.sizeOf(this);
}

// Usage: context.colors.primary
```

A small `context` extension file is one of the highest-value 20 lines in any Flutter codebase.

**Generics** work as expected, with `extends` for bounds:

```dart
abstract interface class Repository<T, ID> {
  Future<T?> findById(ID id);
  Future<List<T>> findAll({int page, int size});
  Future<T> save(T entity);
  Future<void> delete(ID id);
}

class Paginated<T> {
  const Paginated({required this.items, required this.total, required this.page});
  final List<T> items;
  final int total;
  final int page;

  Paginated<R> map<R>(R Function(T) f) =>
      Paginated(items: items.map(f).toList(), total: total, page: page);
}
```

Note `abstract interface class` — that's the Dart 3 way to declare a pure interface that cannot be extended, only implemented. Exactly what you want for repository contracts.

---

## 1.11 Async: Future, Stream, isolates

**Dart is single-threaded per isolate with an event loop** — the same concurrency model as Python's asyncio or Node. `async`/`await` is syntactic sugar over `Future`. If you're comfortable with FastAPI's async, you already know this.

```dart
Future<Order> fetchOrder(String id) async {
  final res = await dio.get<Map<String, dynamic>>('/orders/$id');
  return Order.fromJson(res.data!);
}

// Parallel — like asyncio.gather.
final (order, customer) = await (
  fetchOrder(id),
  fetchCustomer(cid),
).wait;

// List parallel.
final orders = await Future.wait(ids.map(fetchOrder));

// Timeout.
final r = await fetchOrder(id).timeout(const Duration(seconds: 10));
```

The record-based `.wait` above is Dart 3's tuple-typed gather — it preserves individual types, unlike `Future.wait` on a heterogeneous list. Use it.

**Streams** are async iterables — the `AsyncIterator` / observable analogue.

```dart
Stream<int> ticks() async* {
  for (var i = 0; ; i++) {
    await Future<void>.delayed(const Duration(seconds: 1));
    yield i;
  }
}

await for (final t in ticks().take(3)) {
  print(t);
}

// Transformations
final debounced = searchInput
    .debounceTime(const Duration(milliseconds: 300))  // via rxdart
    .distinct()
    .asyncMap(searchApi);
```

Two flavours matter: **single-subscription** (default; one listener, e.g. an HTTP body) and **broadcast** (many listeners, e.g. auth state changes). Calling `.listen` twice on a single-subscription stream throws. `StreamController.broadcast()` when you need fan-out.

**Always cancel subscriptions.** In a `State`, store the `StreamSubscription` and cancel it in `dispose()`. This is the #1 memory-leak source in Flutter apps.

**Isolates** are Dart's answer to true parallelism. Each isolate has its own heap and its own event loop; they communicate by message passing (no shared memory, so no locks — same philosophy as Erlang processes or Python's multiprocessing).

```dart
// One-shot heavy computation off the UI thread.
final parsed = await Isolate.run(() => expensiveParse(bigJsonString));

// In Flutter specifically, `compute` is the ergonomic wrapper:
final result = await compute(parseOrders, jsonString);

List<Order> parseOrders(String raw) =>
    (jsonDecode(raw) as List).map((e) => Order.fromJson(e)).toList();
```

Use an isolate when a synchronous operation would exceed ~8 ms (one frame budget at 120 Hz is 8.3 ms; at 60 Hz, 16.6 ms). Typical candidates: parsing a large JSON payload, image processing, crypto, big list sorting. Network I/O does **not** need an isolate — it's already non-blocking.

---

## 1.12 Error handling and the `Result` type

Dart's exceptions are **unchecked** — nothing in the type system tells you a function can throw. Coming from FastAPI, where you'd raise `HTTPException` and have an exception handler, this feels loose. In a layered client app it's genuinely dangerous: a `DioException` thrown in the data layer will happily propagate into a widget's `build` and crash the frame.

```dart
try {
  await repo.save(order);
} on DioException catch (e, st) {
  logger.e('save failed', error: e, stackTrace: st);
} on FormatException catch (e) {
  // ...
} catch (e) {
  rethrow;     // preserves the original stack trace — never `throw e;`
} finally {
  setBusy(false);
}
```

**The recommended pattern** (this is in Flutter's official design-patterns docs) is to make failure explicit in the return type using a sealed `Result`:

```dart
// lib/core/result.dart
sealed class Result<T> {
  const Result();

  const factory Result.ok(T value) = Ok<T>;
  const factory Result.error(Object error, [StackTrace? stackTrace]) = Err<T>;

  /// Transform the success value, leaving errors untouched.
  Result<R> map<R>(R Function(T) f) => switch (this) {
        Ok<T>(:final value) => Result.ok(f(value)),
        Err<T>(:final error, :final stackTrace) => Result.error(error, stackTrace),
      };

  T? get valueOrNull => this is Ok<T> ? (this as Ok<T>).value : null;
}

final class Ok<T> extends Result<T> {
  const Ok(this.value);
  final T value;
}

final class Err<T> extends Result<T> {
  const Err(this.error, [this.stackTrace]);
  final Object error;
  final StackTrace? stackTrace;
}
```

Usage at the repository boundary:

```dart
Future<Result<List<Order>>> getOrders() async {
  try {
    final res = await _api.get<List<dynamic>>('/orders');
    return Result.ok(res.data!.map(Order.fromJson).toList());
  } on DioException catch (e, st) {
    return Result.error(AppException.fromDio(e), st);
  }
}
```

And in the view model:

```dart
switch (await repo.getOrders()) {
  case Ok(:final value):
    _orders = value;
  case Err(:final error):
    _error = error;
}
notifyListeners();
```

**Where to draw the line:** throw across *unexpected* boundaries (programming errors, invariant violations — let them crash in debug). Return `Result` for *expected* failures that the UI must render (network down, 422 validation, 403). This is the same discipline as distinguishing 500s from 4xx in your FastAPI handlers.

---

## 1.13 Dart idioms cheat sheet

| Task | Idiom |
|---|---|
| Null-safe read | `obj?.field ?? fallback` |
| Copy-to-local for promotion | `final x = field; if (x != null) { ... }` |
| Assign if null | `x ??= compute();` |
| Chain calls on one object | `..cascade()` |
| Conditional widget in a list | `if (cond) Widget()` inside `[...]` |
| Loop into a list | `for (final x in xs) Widget(x)` inside `[...]` |
| Exhaustive state render | `switch (state) { Case() => ..., }` |
| Multiple return values | Named record `({int a, String b})` |
| Parallel awaits, typed | `final (a, b) = await (fa, fb).wait;` |
| Heavy sync work | `await Isolate.run(() => ...)` or `compute(fn, arg)` |
| Preserve stack trace on rethrow | `rethrow`, never `throw e` |
| Deep-ish equality on models | `freezed`, not hand-written `==` |
| Debounce a stream | `rxdart`'s `debounceTime`, or a `Timer` |

---

## Exercises

**Do these before moving on.** Open [dartpad.dev](https://dartpad.dev) and implement:

1. A `sealed class ApiState<T>` with `Loading`, `Data(T)`, `Failure(Object)`, plus a `when`-style exhaustive `switch`.
2. A `Paginated<T>` generic that maps its items.
3. A function returning `({List<Order> items, bool hasMore})`, consumed with destructuring.
4. An extension `on List<Order>` adding `groupByStatus()` returning `Map<OrderStatus, List<Order>>`.

If those four feel comfortable, your Dart is sufficient for everything in the rest of this tutorial.

---

[← Orientation](00-orientation.md) · [Index](README.md) · [Next: Flutter Fundamentals →](02-flutter-fundamentals.md)
