# Part 4 — Architecture

[← Structure at Scale](03b-project-structure-at-scale.md) · [Index](README.md) · [Next: State Management →](05-state-management.md)

---

## 4.1 The official Flutter architecture

Since 2024, Flutter ships an official architecture guide, and the recommendation is **MVVM with a repository-backed data layer**. This is not a community opinion — it's [docs.flutter.dev/app-architecture](https://docs.flutter.dev/app-architecture/guide). Adopting it means your codebase matches the docs, the samples, and every new hire's expectations.

Two layers, four component types:

```
┌───────────────────────── UI Layer ─────────────────────────┐
│                                                            │
│   View  ──── user events ────►  ViewModel                  │
│    ▲                                │                      │
│    └────── UI state (immutable) ────┘                      │
│   (widgets only: layout + rendering)  (logic + state)      │
└────────────────────────────┬───────────────────────────────┘
                             │ calls, awaits
┌────────────────────────────▼──── Data Layer ───────────────┐
│                                                            │
│   Repository  ──►  Service  ──►  (REST API, SQLite,        │
│   (source of truth,   (raw I/O,    platform plugin)        │
│    caching, merging,   one per                             │
│    domain models)      data source)                        │
└────────────────────────────────────────────────────────────┘
```

If you squint, this is your FastAPI layering with the names changed:

| FastAPI | Flutter |
|---|---|
| Router / endpoint function | View (widget) |
| Service / use-case class | ViewModel |
| Repository (SQLAlchemy) | Repository |
| DB session / HTTP client | Service (Dio, Drift) |
| Pydantic schema | DTO (`freezed` + `json_serializable`) |
| Domain model | Domain model |
| `Depends()` | Riverpod provider / `get_it` |

The important architectural claim, and the one you already believe: **the repository is the source of truth for its model data.** Everything above it consumes an abstraction.

---

## 4.2 Layer-by-layer contract

**View — widgets only.**
- Contains layout and rendering. No business logic, no `if (response.statusCode == 401)`.
- Reads immutable state from exactly one view model.
- Forwards user events as method calls on the view model.
- Should be dumb enough that you could swap it for a different design without touching anything else.

**ViewModel — logic and state.**
- Holds the UI state for one screen (or one meaningful section of one).
- Depends on repositories, never on services or Dio directly.
- Transforms domain models into what the view needs (formatting, filtering, grouping).
- Exposes state as an immutable object plus commands for actions.
- Must be unit-testable with zero Flutter widgets in the test.

**Repository — source of truth.**
- One per domain concept (`OrderRepository`, `InventoryRepository`), not one per screen.
- Owns caching, offline/online merge, retry policy, and the mapping from DTO to domain model.
- Returns domain models (and `Result<T>`), never raw HTTP responses or DTOs.
- Declared as an `abstract interface class`, with concrete implementations per environment side by side in `data/repositories/<entity>/` (`_remote`, `_local`, `_dev`). Flutter rates abstract repositories *strongly recommend* — it's what lets you run the whole app against fakes without touching a network, and what makes view-model tests trivially fakeable.

**Service — raw I/O, stateless.**
- One per external data source: `AuthApiClient`, `OrdersApiClient`, `AppDatabase`, `SecureStorageService`.
- Wraps a single API/plugin. Knows nothing about your domain.
- No caching, no business rules, no state.

A concrete slice, top to bottom:

```dart
// lib/domain/models/order.dart
@freezed
sealed class Order with _$Order {
  const factory Order({
    required String id,
    required String number,
    required OrderStatus status,
    required Money total,
    required DateTime createdAt,
  }) = _Order;
}

// lib/data/repositories/orders/orders_repository.dart  (the INTERFACE)
abstract interface class OrdersRepository {
  Future<Result<Paginated<Order>>> list({int page, OrderStatus? status});
  Future<Result<Order>> byId(String id);
  Future<Result<Order>> submit(String id);
}
```

```dart
// lib/data/services/api/orders_api_client.dart  (SERVICE — raw I/O)
final class OrdersApiClient {
  const OrdersApiClient(this._dio);
  final Dio _dio;

  Future<Response<Map<String, dynamic>>> list({
    required int page,
    String? status,
    CancelToken? cancelToken,
  }) =>
      _dio.get<Map<String, dynamic>>(
        '/orders',
        queryParameters: {'page': page, if (status != null) 'status': status},
        cancelToken: cancelToken,
      );
}
```

```dart
// lib/data/repositories/orders/orders_repository_remote.dart  (REPOSITORY)
final class OrdersRepositoryRemote implements OrdersRepository {
  OrdersRepositoryRemote(this._api, this._db);

  final OrdersApiClient _api;
  final AppDatabase _db;

  @override
  Future<Result<Paginated<Order>>> list({int page = 1, OrderStatus? status}) async {
    try {
      final res = await _api.list(page: page, status: status?.name);
      final dto = OrderListResponse.fromJson(res.data!);
      final orders = dto.items.map((e) => e.toDomain()).toList();
      await _db.upsertOrders(orders);                 // cache
      return Result.ok(Paginated(items: orders, total: dto.total, page: page));
    } on DioException catch (e, st) {
      // Offline fallback: serve cache, surface staleness upstream if needed.
      if (e.type == DioExceptionType.connectionError) {
        final cached = await _db.ordersPage(page: page, status: status);
        if (cached.isNotEmpty) {
          return Result.ok(Paginated(items: cached, total: cached.length, page: page));
        }
      }
      return Result.error(ApiException.fromDio(e), st);
    }
  }

  // ...
}
```

```dart
// lib/ui/orders/view_models/orders_viewmodel.dart  (VIEWMODEL)
@riverpod
class OrdersViewModel extends _$OrdersViewModel {
  @override
  Future<Paginated<Order>> build({OrderStatus? status}) async {
    final result = await ref.watch(ordersRepositoryProvider).list(status: status);
    return switch (result) {
      Ok(:final value) => value,
      Err(:final error) => throw error,   // Riverpod turns this into AsyncError
    };
  }

  Future<void> submit(String id) async {
    final repo = ref.read(ordersRepositoryProvider);
    state = const AsyncLoading();
    final res = await repo.submit(id);
    if (res is Err) {
      state = AsyncError(res.error, res.stackTrace ?? StackTrace.current);
      return;
    }
    ref.invalidateSelf();      // refetch the list
  }
}
```

```dart
// lib/ui/orders/widgets/orders_screen.dart  (VIEW)
class OrdersScreen extends ConsumerWidget {
  const OrdersScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final orders = ref.watch(ordersViewModelProvider());

    return Scaffold(
      appBar: AppBar(title: const Text('Orders')),
      body: switch (orders) {
        AsyncLoading() => const Center(child: CircularProgressIndicator()),
        AsyncError(:final error) => ErrorView(
            error: error,
            onRetry: () => ref.invalidate(ordersViewModelProvider()),
          ),
        AsyncData(:final value) => RefreshIndicator(
            onRefresh: () async => ref.invalidate(ordersViewModelProvider()),
            child: ListView.builder(
              itemCount: value.items.length,
              itemBuilder: (context, i) {
                final order = value.items[i];
                return OrderTile(
                  key: ValueKey(order.id),
                  order: order,
                  onSubmit: () => ref
                      .read(ordersViewModelProvider().notifier)
                      .submit(order.id),
                );
              },
            ),
          ),
      },
    );
  }
}
```

Read that view again: it has zero knowledge of HTTP, JSON, caching, or auth. That's the whole point.

---

## 4.3 Mapping your ERP backend to a Flutter client

Since your backend is the source of truth and the client is a presentation layer, a few decisions follow directly.

### 1. Do not re-implement business rules on the client

Validate for UX (required fields, formats) but let the server be authoritative. When the server returns a 422 with field errors, render them — don't try to predict them. This means your `ApiException` needs a first-class "field errors" shape:

```dart
@freezed
sealed class ApiException with _$ApiException implements Exception {
  const factory ApiException.network() = NetworkException;
  const factory ApiException.timeout() = TimeoutException;
  const factory ApiException.unauthorized() = UnauthorizedException;
  const factory ApiException.forbidden() = ForbiddenException;
  const factory ApiException.notFound() = NotFoundException;
  const factory ApiException.validation(Map<String, List<String>> fields) =
      ValidationException;
  const factory ApiException.server(int status, String? message) = ServerException;
  const factory ApiException.unknown(Object cause) = UnknownException;
}
```

FastAPI's default 422 body is `{"detail": [{"loc": ["body","email"], "msg": "...", "type": "..."}]}`. Write one adapter that turns that into `Map<String, List<String>>` ([§6.6](06-networking-auth-data.md#66-error-taxonomy)) and your forms get server-side validation for free across the whole app.

### 2. Mirror your Pydantic schemas as DTOs, then map to domain models

The temptation is to skip the domain model and use the DTO everywhere. Resist it for anything non-trivial: DTOs change when the API changes, and you want that blast radius contained to the data layer. For simple read-only resources, collapsing them is a defensible pragmatic call.

### 3. Model permissions the same way your backend does

If your ERP has role/permission checks, ship the permission set in the JWT claims or a `/me` endpoint, hold it in an `AuthRepository`, and gate UI on it. Always re-check server-side — client-side gating is UX, not security.

### 4. Pagination contract

Pick one (offset/limit, page/size, or cursor) and make `Paginated<T>` match it exactly. Cursor pagination is strictly better for lists that mutate — which in an ERP, they do.

### 5. Idempotency

For mutations triggered from mobile (flaky networks, retry buttons), send an `Idempotency-Key` header generated client-side per logical operation. Your FastAPI layer dedupes on it. This is the single highest-value thing you can do for mobile data integrity, and it's easy because you already own both sides.

### 6. Long-running operations

ERP actions (post a journal, run MRP) can exceed a mobile HTTP timeout. Return `202 Accepted` + a task id, and poll or subscribe (WebSocket/SSE) from the client. Model it as a sealed `JobState` in the view model.

---

## 4.4 Dependency injection

Two viable approaches.

### Option A — Riverpod as the DI container (recommended)

If you're using Riverpod for state anyway, providers *are* the container. Overriding them in tests is first-class.

```dart
// lib/config/dependencies.dart
@riverpod
Dio dio(Ref ref) {
  final config = ref.watch(appConfigProvider);
  final client = Dio(BaseOptions(
    baseUrl: config.apiBaseUrl,
    connectTimeout: const Duration(seconds: 15),
    receiveTimeout: const Duration(seconds: 20),
  ));
  client.interceptors.addAll([
    AuthInterceptor(ref.watch(secureStoreProvider), client),
    if (config.enableLogging) PrettyDioLogger(requestBody: true),
  ]);
  ref.onDispose(client.close);
  return client;
}

@riverpod
OrdersApiClient ordersApiClient(Ref ref) => OrdersApiClient(ref.watch(dioProvider));

@riverpod
OrdersRepository ordersRepository(Ref ref) => OrdersRepositoryRemote(
      ref.watch(ordersApiClientProvider),
      ref.watch(appDatabaseProvider),
    );
```

In a test:

```dart
final container = ProviderContainer(overrides: [
  ordersRepositoryProvider.overrideWithValue(FakeOrdersRepository()),
]);
addTearDown(container.dispose);
```

That's your `app.dependency_overrides` equivalent, and it composes.

### Option B — `get_it` + `injectable`

A classic service locator, closer to a Spring/FastAPI `Depends` container. Works fine, integrates with Bloc, and is preferred by teams who want DI decoupled from state management. Downside: it's a global mutable registry, so it's easier to create ordering bugs, and test isolation requires discipline (`getIt.reset()`).

**Recommendation:** pick Riverpod-as-DI if you're on Riverpod, `get_it` if you're on Bloc. Do not run both.

---

## 4.5 The Command pattern

Every mutating action has the same four states: idle, running, succeeded, failed. Writing `_isLoading`/`_error` pairs by hand in every view model is the boilerplate that makes people hate MVVM. Flutter's official design-patterns docs recommend a `Command` class. It's ~40 lines and pays for itself immediately.

```dart
// lib/utils/command.dart
import 'package:flutter/foundation.dart';

class Command<T> extends ChangeNotifier {
  Command(this._action);
  final Future<Result<T>> Function() _action;

  bool _running = false;
  Result<T>? _result;

  bool get running => _running;
  Result<T>? get result => _result;
  bool get hasError => _result is Err<T>;
  T? get value => _result is Ok<T> ? (_result! as Ok<T>).value : null;

  Future<void> execute() async {
    if (_running) return;              // built-in double-tap protection
    _running = true;
    _result = null;
    notifyListeners();
    try {
      _result = await _action();
    } finally {
      _running = false;
      notifyListeners();
    }
  }

  void clearResult() {
    _result = null;
    notifyListeners();
  }
}
```

```dart
// In a ChangeNotifier-style view model:
class OrderDetailViewModel extends ChangeNotifier {
  OrderDetailViewModel(this._repo, this._id) {
    load = Command(() => _repo.byId(_id))..addListener(notifyListeners);
    submit = Command(() => _repo.submit(_id))..addListener(notifyListeners);
    load.execute();
  }

  final OrdersRepository _repo;
  final String _id;

  late final Command<Order> load;
  late final Command<Order> submit;
}
```

```dart
// In the view:
ListenableBuilder(
  listenable: vm.submit,
  builder: (context, _) => FilledButton(
    onPressed: vm.submit.running ? null : vm.submit.execute,
    child: vm.submit.running
        ? const SizedBox.square(dimension: 18, child: CircularProgressIndicator(strokeWidth: 2))
        : const Text('Submit'),
  ),
);
```

You get: disabled-while-running, no double submissions, uniform error surface, and a single place to hook analytics on every user action. If you use Riverpod, `AsyncValue` covers much of this — but `Command` is still useful for actions (as opposed to reads), and Riverpod 3's experimental "mutations" API is heading in this same direction.

---

[← Structure at Scale](03b-project-structure-at-scale.md) · [Index](README.md) · [Next: State Management →](05-state-management.md)
