# Part 8 — Persistence and Offline-First

[← Navigation](07-navigation.md) · [Index](README.md) · [Next: Forms & Real UI Work →](09-forms-and-ui.md)

---

## 8.1 Key-value

`shared_preferences` for non-sensitive scalars: theme mode, last selected warehouse, onboarding-seen flag, feature toggles.

```dart
@riverpod
class ThemeModeSetting extends _$ThemeModeSetting {
  static const _key = 'theme_mode';

  @override
  Future<ThemeMode> build() async {
    final prefs = await SharedPreferences.getInstance();
    return ThemeMode.values.byName(prefs.getString(_key) ?? 'system');
  }

  Future<void> set(ThemeMode mode) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(_key, mode.name);
    state = AsyncData(mode);
  }
}
```

Rules: never store tokens, PII, or anything a competitor would enjoy reading. Wrap it behind a typed service rather than sprinkling raw string keys across features.

---

## 8.2 SQL with Drift

You think in SQL and Postgres. **Drift** is the right choice: a typed SQLite layer for Dart with compile-time-checked queries, migrations, and reactive streams. It is the closest thing Flutter has to SQLAlchemy + Alembic.

```dart
// core/storage/app_database.dart
import 'package:drift/drift.dart';

part 'app_database.g.dart';

class Orders extends Table {
  TextColumn get id => text()();
  TextColumn get number => text()();
  TextColumn get status => text()();
  IntColumn get totalCents => integer()();
  TextColumn get currency => text().withLength(min: 3, max: 3)();
  DateTimeColumn get createdAt => dateTime()();
  DateTimeColumn get syncedAt => dateTime().nullable()();
  BoolColumn get isDirty => boolean().withDefault(const Constant(false))();

  @override
  Set<Column> get primaryKey => {id};
}

@DriftDatabase(tables: [Orders])
class AppDatabase extends _$AppDatabase {
  AppDatabase(super.e);

  @override
  int get schemaVersion => 2;

  @override
  MigrationStrategy get migration => MigrationStrategy(
        onCreate: (m) => m.createAll(),
        onUpgrade: (m, from, to) async {
          if (from < 2) await m.addColumn(orders, orders.isDirty);
        },
        beforeOpen: (details) async {
          await customStatement('PRAGMA foreign_keys = ON');
        },
      );

  /// Reactive query — emits a new list whenever the underlying rows change.
  Stream<List<Order>> watchByStatus(OrderStatus status) =>
      (select(orders)..where((t) => t.status.equals(status.name)))
          .watch()
          .map((rows) => rows.map(_toDomain).toList());

  Future<void> upsertOrders(List<Order> items) => batch((b) {
        b.insertAllOnConflictUpdate(orders, items.map(_toRow).toList());
      });
}
```

`.watch()` is the feature that changes how you build offline UIs: your widget subscribes to a SQL query, and any write — from a sync job, a background isolate, whatever — pushes new data to the UI automatically. No manual invalidation.

Drift also ships schema-version testing (`drift_dev schema`) so migrations are verified in CI, which you'll appreciate given how painful a broken mobile migration is (you can't roll back a user's device).

---

## 8.3 Offline-first repository

Offline-first is a spectrum. Pick a level per feature, not per app:

| Level | Behaviour | Cost |
|---|---|---|
| 0 | Online only, error when offline | Trivial |
| 1 | Read cache, write requires network | Low — **best default** |
| 2 | Read cache + queued writes, sync on reconnect | Medium |
| 3 | Full bidirectional sync with conflict resolution | High — needs backend design |

For an ERP field client, **Level 2** is usually the right target: warehouse staff can record counts underground and sync when they surface. Level 3 is a distributed-systems project, not a client feature.

```dart
final class OrdersRepositoryOfflineFirst implements OrdersRepository {
  OrdersRepositoryOfflineFirst(this._api, this._db, this._connectivity) {
    _connectivity.onConnectivityChanged.listen((r) {
      if (!r.contains(ConnectivityResult.none)) unawaited(_drainOutbox());
    });
  }

  /// UI always reads from the DB. Network only ever writes to the DB.
  Stream<List<Order>> watchOrders() => _db.watchAllOrders();

  Future<void> refresh() async {
    final since = await _db.lastSyncedAt();
    final res = await _api.list(updatedSince: since);   // delta sync
    await _db.upsertOrders(res.items.map((e) => e.toDomain()).toList());
    await _db.setLastSyncedAt(DateTime.now().toUtc());
  }

  @override
  Future<Result<Order>> submit(String id) async {
    // 1. Write locally + mark dirty  -> UI updates instantly via watch()
    await _db.markDirty(id, action: 'submit');
    // 2. Try to push now; if it fails, the outbox retries later.
    return _drainOutbox();
  }

  Future<Result<Order>> _drainOutbox() async { /* replay queued mutations */ }
}
```

**Design notes for the backend side (yours):**

- Every syncable entity needs a server-assigned `updated_at` and a stable id. Client-generated UUIDs for offline-created records avoid a whole class of id-reconciliation pain.
- Support `?updated_since=` delta queries, and include **tombstones** for deletes — otherwise deleted rows live forever on devices.
- Return the server's canonical record after a write so the client can replace its optimistic copy.
- Decide the conflict policy explicitly: last-write-wins is fine for user preferences and unacceptable for inventory counts. For the latter, send deltas (`+3`) rather than absolutes (`=17`), or version rows and return `409` on mismatch.

---

## 8.4 Optimistic state

Flutter's docs call this out as a first-class pattern, and for an app over mobile networks it's the difference between "feels instant" and "feels broken."

```dart
@riverpod
class OrderActions extends _$OrderActions {
  @override
  void build() {}

  Future<void> submit(Order order) async {
    final list = ref.read(orderListProvider.notifier);
    final snapshot = ref.read(orderListProvider).valueOrNull ?? const [];

    // 1. Apply optimistically.
    list.replace(order.copyWith(status: OrderStatus.submitted));

    // 2. Fire the real request.
    final res = await ref.read(ordersRepositoryProvider).submit(order.id);

    // 3. Reconcile.
    switch (res) {
      case Ok(:final value):
        list.replace(value);                        // server truth wins
      case Err(:final error):
        list.restore(snapshot);                     // roll back
        ref.read(toastProvider).showError(error);
    }
  }
}
```

Apply optimism where failure is rare and reversible (toggles, marking read, adding to a cart). Don't apply it to irreversible or money-moving operations — show a real pending state there.

---

[← Navigation](07-navigation.md) · [Index](README.md) · [Next: Forms & Real UI Work →](09-forms-and-ui.md)
