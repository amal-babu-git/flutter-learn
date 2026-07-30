# Part 6 — Networking, Auth, and the Data Layer

[← State Management](05-state-management.md) · [Index](README.md) · [Next: Navigation →](07-navigation.md)

---

This is where your backend experience pays off most, and where the mobile-specific gotchas live.

## 6.1 Dio client setup

```dart
// lib/data/services/api/api_client.dart
import 'package:dio/dio.dart';

Dio buildDio({
  required String baseUrl,
  required bool enableLogging,
  required SecureStore store,
}) {
  final dio = Dio(
    BaseOptions(
      baseUrl: baseUrl,
      connectTimeout: const Duration(seconds: 15),
      receiveTimeout: const Duration(seconds: 20),
      sendTimeout: const Duration(seconds: 20),
      headers: {'Accept': 'application/json'},
      // Let interceptors see 4xx instead of throwing immediately.
      validateStatus: (code) => code != null && code < 500,
    ),
  );

  dio.interceptors.addAll([
    RequestIdInterceptor(),          // X-Request-Id for cross-system tracing
    AuthInterceptor(store: store, dio: dio),
    ErrorMappingInterceptor(),       // DioException -> ApiException
    if (enableLogging) PrettyDioLogger(requestBody: true, responseBody: false),
  ]);

  return dio;
}
```

Order matters: interceptors run request-phase top-to-bottom and response-phase bottom-to-top. Put auth before error mapping so a refreshed retry isn't mapped as a failure.

A trace-id interceptor is a five-line change that makes production debugging dramatically easier when you control both sides:

```dart
class RequestIdInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    options.headers['X-Request-Id'] = const Uuid().v4();
    handler.next(options);
  }
}
```

Log that id on both client and FastAPI side and you can join a mobile crash to a server trace.

---

## 6.2 Token storage

**Never** put tokens in `SharedPreferences` — that's plaintext XML on Android and a plist on iOS. Use `flutter_secure_storage`, which wraps the Android Keystore/EncryptedSharedPreferences and the iOS Keychain.

```dart
// lib/data/services/secure_storage_service.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

final class SecureStore {
  SecureStore([FlutterSecureStorage? storage])
      : _s = storage ??
            const FlutterSecureStorage(
              aOptions: AndroidOptions(encryptedSharedPreferences: true),
              iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
            );

  final FlutterSecureStorage _s;

  static const _kAccess = 'access_token';
  static const _kRefresh = 'refresh_token';

  Future<String?> get accessToken => _s.read(key: _kAccess);
  Future<String?> get refreshToken => _s.read(key: _kRefresh);

  Future<void> saveTokens({required String access, required String refresh}) =>
      Future.wait([
        _s.write(key: _kAccess, value: access),
        _s.write(key: _kRefresh, value: refresh),
      ]);

  Future<void> clear() => _s.deleteAll();
}
```

**Design guidance from the backend side:** short-lived access tokens (5–15 min) with rotating refresh tokens, refresh-token reuse detection server-side, and revocation on logout. On mobile you can't use `HttpOnly` cookies for a native app, so the Keychain/Keystore is the closest equivalent to a secure cookie jar. Note `KeychainAccessibility.first_unlock` — it prevents reads before the device has been unlocked once after boot, which is the right default.

Also consider: on logout, call your backend's revoke endpoint *and* clear local storage *and* invalidate all Riverpod providers holding user data. Missing any of the three leaks state between users on a shared device — a real scenario in warehouse/field deployments.

---

## 6.3 The JWT refresh interceptor

This is the piece everyone gets subtly wrong. The naive version fires N refresh requests when N parallel requests get 401 simultaneously, and rotating refresh tokens then invalidate each other, logging the user out.

The correct pattern is **single-flight refresh with a request queue**:

```dart
// lib/data/services/api/auth_interceptor.dart
import 'dart:async';
import 'package:dio/dio.dart';

class AuthInterceptor extends Interceptor {
  AuthInterceptor({required this.store, required this.dio, required this.onLogout});

  final SecureStore store;
  final Dio dio;                       // main client (for retries)
  final Future<void> Function() onLogout;

  /// Non-null while a refresh is in flight. All callers await the same future.
  Completer<String?>? _refreshCompleter;

  static const _retriedKey = 'x-retried';

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    if (options.extra['skipAuth'] != true) {
      final token = await store.accessToken;
      if (token != null) options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  Future<void> onResponse(
    Response response,
    ResponseInterceptorHandler handler,
  ) async {
    // With validateStatus<500, 401s arrive here, not in onError.
    if (response.statusCode != 401 ||
        response.requestOptions.extra[_retriedKey] == true ||
        response.requestOptions.extra['skipAuth'] == true) {
      return handler.next(response);
    }

    final newToken = await _refreshOnce();
    if (newToken == null) {
      await onLogout();
      return handler.next(response);       // let the 401 surface
    }

    try {
      final retried = await _retry(response.requestOptions, newToken);
      handler.resolve(retried);
    } on DioException catch (e) {
      handler.reject(e);
    }
  }

  /// Ensures only ONE refresh request is in flight; concurrent callers await it.
  Future<String?> _refreshOnce() {
    final existing = _refreshCompleter;
    if (existing != null) return existing.future;

    final completer = Completer<String?>();
    _refreshCompleter = completer;

    unawaited(() async {
      try {
        final refresh = await store.refreshToken;
        if (refresh == null) {
          completer.complete(null);
          return;
        }
        // Bare Dio: must not go through this interceptor.
        final fresh = Dio(BaseOptions(baseUrl: dio.options.baseUrl));
        final res = await fresh.post<Map<String, dynamic>>(
          '/auth/refresh',
          data: {'refresh_token': refresh},
        );
        final pair = TokenPair.fromJson(res.data!);
        await store.saveTokens(access: pair.access, refresh: pair.refresh);
        completer.complete(pair.access);
      } catch (_) {
        completer.complete(null);
      } finally {
        _refreshCompleter = null;
      }
    }());

    return completer.future;
  }

  Future<Response<dynamic>> _retry(RequestOptions req, String token) {
    final opts = Options(
      method: req.method,
      headers: {...req.headers, 'Authorization': 'Bearer $token'},
      extra: {...req.extra, _retriedKey: true},
      responseType: req.responseType,
      contentType: req.contentType,
    );
    return dio.request<dynamic>(
      req.path,
      data: req.data,
      queryParameters: req.queryParameters,
      cancelToken: req.cancelToken,
      options: opts,
    );
  }
}
```

**Why each guard exists:**

| Guard | Prevents |
|---|---|
| `_refreshCompleter` single-flight | N parallel refreshes invalidating each other under rotation |
| `_retriedKey` in `extra` | Infinite retry loop when the refreshed token is also rejected |
| `skipAuth` extra flag | The login/refresh endpoints themselves triggering auth logic |
| Bare `Dio` for the refresh call | Recursive interception |
| `onLogout` callback | Silent failure — the user must land on the login screen |

**Additional hardening worth adding in production:** proactively refresh when the access token is within ~60s of expiry (decode the `exp` claim locally — no need to wait for a 401); add jitter if you have many clients; and handle the "refresh token itself expired" case distinctly from "network down" so you don't log people out because they walked into a lift.

**Certificate pinning:** for a high-value ERP client, pin your server's public key via `dio`'s `HttpClientAdapter` `badCertificateCallback` or `http_certificate_pinning`. Weigh it against the operational cost of certificate rotation — pinning has bricked more apps than it has saved.

---

## 6.4 DTOs, models, and serialization

Dart has no runtime reflection in AOT, so JSON mapping is **code-generated**, not reflective. This is `pydantic` with a build step instead of runtime introspection — more verbose to set up, zero runtime cost.

```dart
// lib/data/model/order/order_dto.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'order_dto.freezed.dart';
part 'order_dto.g.dart';

@freezed
sealed class OrderDto with _$OrderDto {
  const OrderDto._();                       // enables custom methods

  const factory OrderDto({
    required String id,
    required String number,
    required String status,
    @JsonKey(name: 'total_cents') required int totalCents,
    required String currency,
    @JsonKey(name: 'created_at') required DateTime createdAt,
  }) = _OrderDto;

  factory OrderDto.fromJson(Map<String, dynamic> json) => _$OrderDtoFromJson(json);

  /// Anti-corruption boundary: API shape -> domain shape.
  Order toDomain() => Order(
        id: id,
        number: number,
        status: OrderStatus.fromApi(status),
        total: Money(totalCents, currency: currency),
        createdAt: createdAt,
      );
}
```

`freezed` generates: an immutable class with real value equality, `copyWith`, `toString`, `hashCode`, and (with `json_serializable`) `fromJson`/`toJson`. Value equality is not cosmetic — Riverpod 3 uses `==` to decide whether to notify listeners, so a model without it causes spurious rebuilds.

`@JsonKey(name: ...)` handles snake_case ↔ camelCase. To avoid annotating every field, set it globally:

```yaml
# build.yaml
targets:
  $default:
    builders:
      json_serializable:
        options:
          field_rename: snake
          create_to_json: true
          include_if_null: false
```

Now `totalCents` maps to `total_cents` automatically — matching FastAPI/Pydantic's typical output.

**Codegen workflow:**

```bash
dart run build_runner watch --delete-conflicting-outputs
```

Leave it running. Commit generated files or don't — either is defensible; **not** committing keeps diffs clean but requires codegen in CI. Add `*.g.dart` and `*.freezed.dart` to `analyzer.exclude` ([§3.3](03-project-setup.md#33-strict-analysis-options)) regardless.

**Parsing large payloads off the main thread:**

```dart
Future<List<Order>> parseOrders(String body) => compute(_parse, body);

List<Order> _parse(String body) => (jsonDecode(body) as List)
    .map((e) => OrderDto.fromJson(e as Map<String, dynamic>).toDomain())
    .toList();
```

Worth it above roughly a few hundred KB. Below that the isolate handoff cost dominates.

---

## 6.5 Repositories

The repository is where the interesting decisions live: caching policy, offline behaviour, and the DTO→domain mapping.

```dart
// lib/data/repositories/orders/orders_repository_remote.dart
final class OrdersRepositoryRemote implements OrdersRepository {
  OrdersRepositoryRemote(this._api, this._db, this._connectivity);

  final OrdersApiClient _api;
  final AppDatabase _db;
  final Connectivity _connectivity;

  final _controller = StreamController<List<Order>>.broadcast();

  /// Reactive read: emits cache immediately, then network.
  Stream<List<Order>> watchOrders() async* {
    yield await _db.allOrders();               // instant paint from cache
    try {
      final fresh = await _fetchAndCache();
      yield fresh;
    } on ApiException {
      // Stale cache already delivered; swallow or surface via a separate channel.
    }
    yield* _controller.stream;
  }

  Future<List<Order>> _fetchAndCache() async {
    final res = await _api.list(page: 1);
    final orders = OrderListResponse.fromJson(res.data!)
        .items.map((e) => e.toDomain()).toList();
    await _db.replaceOrders(orders);
    return orders;
  }

  @override
  Future<Result<Order>> submit(String id) async {
    try {
      final res = await _api.submit(id, idempotencyKey: const Uuid().v4());
      final order = OrderDto.fromJson(res.data!).toDomain();
      await _db.upsertOrder(order);
      _controller.add(await _db.allOrders());
      return Result.ok(order);
    } on DioException catch (e, st) {
      return Result.error(ApiException.fromDio(e), st);
    }
  }

  void dispose() => _controller.close();
}
```

**Repository rules:**

- One per domain entity, shared across all screens that need it. Two repositories for `Order` means two caches and eventual divergence.
- Returns domain models. If a widget can see `OrderDto`, the abstraction has leaked.
- Owns the cache. View models must never touch the database.
- Stateless-looking API, stateful internals is fine — that's the point.

---

## 6.6 Error taxonomy

Map transport errors into a domain vocabulary exactly once, at the data-layer boundary:

```dart
extension ApiExceptionMapper on ApiException {
  static ApiException fromDio(DioException e) {
    if (e.type == DioExceptionType.connectionError ||
        e.error is SocketException) {
      return const ApiException.network();
    }
    if (e.type == DioExceptionType.connectionTimeout ||
        e.type == DioExceptionType.receiveTimeout ||
        e.type == DioExceptionType.sendTimeout) {
      return const ApiException.timeout();
    }
    if (e.type == DioExceptionType.cancel) {
      return const ApiException.unknown('cancelled');
    }

    final status = e.response?.statusCode;
    final data = e.response?.data;

    return switch (status) {
      401 => const ApiException.unauthorized(),
      403 => const ApiException.forbidden(),
      404 => const ApiException.notFound(),
      422 => ApiException.validation(_fastApiFieldErrors(data)),
      final s? when s >= 500 => ApiException.server(s, _message(data)),
      _ => ApiException.unknown(e),
    };
  }

  /// FastAPI: {"detail":[{"loc":["body","email"],"msg":"...","type":"..."}]}
  static Map<String, List<String>> _fastApiFieldErrors(Object? data) {
    final out = <String, List<String>>{};
    if (data case {'detail': final List details}) {
      for (final d in details) {
        if (d case {'loc': final List loc, 'msg': final String msg}) {
          final field = loc.length > 1 ? loc.last.toString() : '_';
          (out[field] ??= []).add(msg);
        }
      }
    }
    return out;
  }
}
```

Then one presentation-layer function turns an `ApiException` into user-facing copy:

```dart
String describe(ApiException e, AppLocalizations l10n) => switch (e) {
      NetworkException() => l10n.errorOffline,
      TimeoutException() => l10n.errorSlowConnection,
      UnauthorizedException() => l10n.errorSessionExpired,
      ForbiddenException() => l10n.errorNoPermission,
      NotFoundException() => l10n.errorNotFound,
      ValidationException() => l10n.errorCheckFields,
      ServerException() => l10n.errorServer,
      UnknownException() => l10n.errorUnknown,
    };
```

Exhaustive by construction: adding a variant breaks the build until you localize it.

---

## 6.7 Pagination, cancellation, retries

**Cancellation** — every request from a screen should be cancellable, because users navigate away:

```dart
@riverpod
Future<Paginated<Order>> ordersPage(Ref ref, int page) async {
  final cancelToken = CancelToken();
  ref.onDispose(cancelToken.cancel);       // auto-cancel on dispose
  return ref.watch(ordersRepositoryProvider).list(page: page, cancelToken: cancelToken);
}
```

**Infinite scroll** with a notifier accumulating pages:

```dart
@riverpod
class OrderFeed extends _$OrderFeed {
  int _page = 1;
  bool _hasMore = true;

  @override
  Future<List<Order>> build() => _fetch(1);

  Future<void> loadMore() async {
    if (!_hasMore || state.isLoading) return;
    final current = state.valueOrNull ?? const [];
    state = AsyncData(current).copyWithPrevious(state, isRefresh: false);
    final next = await _fetch(_page + 1);
    if (next.isEmpty) _hasMore = false; else _page++;
    state = AsyncData([...current, ...next]);
  }

  Future<List<Order>> _fetch(int page) async { /* ... */ }
}
```

Trigger from a `ScrollController` listener at ~80% scroll extent, or use `NotificationListener<ScrollEndNotification>`.

**Retry policy:** retry only idempotent requests (GET, and PUT/DELETE if your API guarantees it) with exponential backoff and jitter. Never blind-retry POSTs — that's what idempotency keys are for. Riverpod 3 auto-retries failed providers by default; for providers wrapping mutations, turn that off explicitly.

---

[← State Management](05-state-management.md) · [Index](README.md) · [Next: Navigation →](07-navigation.md)
