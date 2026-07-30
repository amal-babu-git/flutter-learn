# Part 5 — State Management

[← Architecture](04-architecture.md) · [Index](README.md) · [Next: Networking, Auth & Data →](06-networking-auth-data.md)

---

## 5.1 The ladder of state

Do not start with a state management library. Start at the bottom rung and climb only when you hit a real limit.

| Rung | Tool | Use when |
|---|---|---|
| 0 | Constructor parameters | Data flows one level down. Just pass it. |
| 1 | `setState` | State is ephemeral and used by exactly one widget: is a dropdown open, current tab index, animation flag. |
| 2 | `ValueNotifier` + `ValueListenableBuilder` | One value, a few listeners, no dependency graph. Zero dependencies, extremely cheap. |
| 3 | `ChangeNotifier` + `ListenableBuilder` | A small view model with several fields. This is what Flutter's own architecture docs use — it's fully sufficient for many apps. |
| 4 | `InheritedWidget` / `InheritedNotifier` | You need to expose something to a whole subtree without prop-drilling and don't want a package. |
| 5 | **Riverpod** | You need a dependency graph between pieces of state, async caching, auto-disposal, and DI. **The default for new apps.** |
| 6 | **Bloc** | You need an auditable event log, strict team conventions, or complex event-driven coordination. |

**The distinction that actually matters** (and Flutter's docs name it): *ephemeral state* vs *app state*. Ephemeral state lives and dies with a widget — keep it in `State`. App state is shared, outlives a screen, or must be tested independently — hoist it. Misclassifying app state as ephemeral is the root cause of most "why is my state resetting" bugs.

---

## 5.2 Riverpod, properly

Riverpod is a compile-safe, `BuildContext`-independent dependency graph with caching, auto-disposal, and built-in async state. Think Zustand + TanStack Query + a DI container. Version 3.x is current; `StateProvider`, `StateNotifierProvider`, and `ChangeNotifierProvider` are legacy — use `Notifier` / `AsyncNotifier`.

**Setup:**

```dart
void main() {
  runApp(const ProviderScope(child: App()));
}
```

**The four things you'll declare**, all via the `@riverpod` annotation + codegen:

```dart
// 1. A plain dependency (DI). Cached, disposed with the scope.
@riverpod
OrdersRepository ordersRepository(Ref ref) =>
    OrdersRepositoryRemote(ref.watch(ordersApiClientProvider), ref.watch(appDatabaseProvider));

// 2. Derived/computed state — recomputes when a watched provider changes.
@riverpod
int pendingCount(Ref ref) {
  final orders = ref.watch(ordersViewModelProvider()).valueOrNull;
  return orders?.items.where((o) => o.status == OrderStatus.submitted).length ?? 0;
}

// 3. Async read with caching. `family` parameters become function parameters.
@riverpod
Future<Order> orderById(Ref ref, String id) async {
  final res = await ref.watch(ordersRepositoryProvider).byId(id);
  return switch (res) {
    Ok(:final value) => value,
    Err(:final error) => throw error,
  };
}

// 4. Mutable state with methods — the view model.
@riverpod
class Cart extends _$Cart {
  @override
  List<CartLine> build() => const [];

  void add(CartLine line) => state = [...state, line];
  void remove(String sku) =>
      state = state.where((l) => l.sku != sku).toList();
}
```

Run `dart run build_runner watch -d` and the `*.g.dart` files appear with `ordersRepositoryProvider`, `orderByIdProvider`, `cartProvider`, all fully typed.

**Consuming:**

```dart
class OrderDetail extends ConsumerWidget {
  const OrderDetail({super.key, required this.id});
  final String id;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final order = ref.watch(orderByIdProvider(id));   // AsyncValue<Order>

    return order.when(
      loading: () => const Center(child: CircularProgressIndicator()),
      error: (e, st) => ErrorView(error: e),
      data: (o) => OrderView(order: o),
    );
  }
}
```

`ref` has three verbs, and confusing them is the #1 Riverpod mistake:

| Verb | Behaviour | Use in |
|---|---|---|
| `ref.watch(p)` | Subscribe. Rebuilds/recomputes on change. | `build()` methods only. |
| `ref.read(p)` | One-shot read, no subscription. | Callbacks (`onPressed`), event handlers. **Never in `build`.** |
| `ref.listen(p, cb)` | Subscribe with a side effect callback, no rebuild. | Showing snackbars/navigating in response to state changes. |

```dart
// Side effects belong in ref.listen, never in build().
ref.listen(authControllerProvider, (prev, next) {
  if (next is AsyncError) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(next.error.toString())),
    );
  }
});
```

### `AsyncValue` is the thing to master

It's a sealed union of `AsyncData` / `AsyncLoading` / `AsyncError`, and it carries the *previous* value during a refresh, so you can show stale data with a loading indicator rather than flashing a spinner:

```dart
switch (orders) {
  // Refreshing with data still present:
  case AsyncValue(:final value?, isLoading: true):
    return Stack(children: [OrderList(value), const LinearProgressIndicator()]);
  case AsyncValue(:final value?):
    return OrderList(value);
  case AsyncValue(:final error?):
    return ErrorView(error: error);
  default:
    return const CircularProgressIndicator();
}
```

### Lifecycle and caching

```dart
@Riverpod(keepAlive: true)   // survives having no listeners
Future<Session> session(Ref ref) async { ... }

@riverpod
Future<List<Order>> orders(Ref ref) async {
  // Auto-dispose by default. Keep alive for 30s after last listener:
  final link = ref.keepAlive();
  final timer = Timer(const Duration(seconds: 30), link.close);
  ref.onDispose(timer.cancel);

  // Cancel in-flight request when disposed:
  final cancelToken = CancelToken();
  ref.onDispose(cancelToken.cancel);

  return ref.watch(ordersRepositoryProvider).list(cancelToken: cancelToken);
}
```

Invalidation:

```dart
ref.invalidate(ordersProvider);        // discard cache, refetch when next watched
ref.refresh(ordersProvider);           // invalidate + immediately read
ref.invalidateSelf();                  // inside a notifier
```

### Riverpod 3 changes worth knowing

- Providers now use `==` to filter updates — so immutable models with real equality (i.e. `freezed`) matter more than before.
- Provider failures are rethrown wrapped in `ProviderException`.
- Failing providers **auto-retry with backoff by default**. Configurable, and worth disabling for non-idempotent operations.
- `Ref` is unified (no more `Ref<T>` generic).
- Experimental offline persistence and a "mutations" API have landed.

### Testing Riverpod

```dart
test('orders view model loads', () async {
  final container = ProviderContainer(overrides: [
    ordersRepositoryProvider.overrideWithValue(FakeOrdersRepository()),
  ]);
  addTearDown(container.dispose);

  final result = await container.read(ordersViewModelProvider().future);
  expect(result.items, hasLength(3));
});
```

No widgets, no `pumpWidget`, no mocking of HTTP. That's the payoff.

---

## 5.3 Bloc, and when to prefer it

Bloc models a feature as `Events → Bloc → States`, with an explicit, serializable transition log.

```dart
sealed class OrdersEvent {}
final class OrdersRequested extends OrdersEvent {}
final class OrdersRefreshed extends OrdersEvent {}
final class OrderSubmitted extends OrdersEvent {
  OrderSubmitted(this.id);
  final String id;
}

@freezed
sealed class OrdersState with _$OrdersState {
  const factory OrdersState.initial() = _Initial;
  const factory OrdersState.loading() = _Loading;
  const factory OrdersState.loaded(List<Order> orders) = _Loaded;
  const factory OrdersState.failure(Object error) = _Failure;
}

class OrdersBloc extends Bloc<OrdersEvent, OrdersState> {
  OrdersBloc(this._repo) : super(const OrdersState.initial()) {
    on<OrdersRequested>(_onRequested, transformer: droppable());
    on<OrderSubmitted>(_onSubmitted, transformer: sequential());
  }

  final OrdersRepository _repo;

  Future<void> _onRequested(OrdersRequested e, Emitter<OrdersState> emit) async {
    emit(const OrdersState.loading());
    final res = await _repo.list();
    emit(switch (res) {
      Ok(:final value) => OrdersState.loaded(value.items),
      Err(:final error) => OrdersState.failure(error),
    });
  }
}
```

**What Bloc buys you that Riverpod doesn't:**

- `BlocObserver` — a single global hook that sees *every* event and state transition in the app. Wire it to your logging/Sentry and you get a complete, replayable user-action trail. For a regulated or audited ERP, this is a genuine compliance feature.
- **Event transformers** (`droppable`, `restartable`, `sequential`, `concurrent` from `bloc_concurrency`) — declarative concurrency control per event type. Debouncing search, dropping duplicate submits, serializing writes: one line each.
- **Enforced uniformity.** With 10+ engineers, "every feature is a Bloc with events and states" removes an entire category of architecture argument from code review.
- `HydratedBloc` — automatic state persistence with almost no code.

**Costs:** more boilerplate per feature (an event class per action), and cross-bloc coordination is clunkier than a provider graph (`BlocListener` chains vs `ref.watch`).

---

## 5.4 Choosing

**Default to Riverpod** for a new app, especially a solo or small-team build. Less boilerplate, no `BuildContext` coupling, providers double as your DI container, `AsyncValue` handles the data+loading+error triad that dominates a backend-driven app, and testing is override-based and clean.

**Choose Bloc** if: the team is large and needs enforced convention; the domain requires an audit trail of every state transition; or you have genuinely complex event choreography where transformers do real work.

**Do not** mix them for the same feature, and don't adopt GetX — it optimizes for typing less and against everything you value about layering and testability.

**Migration note:** Provider→Riverpod can be done incrementally (both can coexist). Bloc↔Riverpod is a real rewrite — pick once.

---

[← Architecture](04-architecture.md) · [Index](README.md) · [Next: Networking, Auth & Data →](06-networking-auth-data.md)
