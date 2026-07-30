# Part 10 — Testing

[← Forms & Real UI Work](09-forms-and-ui.md) · [Index](README.md) · [Next: Performance →](11-performance.md)

---

## 10.1 The pyramid

Flutter's testing story is genuinely excellent — better than most web stacks, because widget tests run headless in milliseconds with no browser.

```
        /\        integration_test   — few, slow (seconds), real device
       /  \       widget tests       — many, fast (ms), headless
      /____\      unit tests         — most, instant
```

Target ratios for a business app:

- **~60% unit** — view models, repositories, mappers, pure logic
- **~30% widget** — screens render correct states, interactions call the right methods
- **~10% integration** — 2–5 critical end-to-end journeys: login, create order, sync

The architecture in [Part 4](04-architecture.md) is what makes this achievable — a view model that depends on an interface is trivially testable; one that news up a `Dio` is not.

---

## 10.2 Unit tests

```dart
// test/ui/orders/orders_viewmodel_test.dart   (test/ mirrors lib/)
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockOrdersRepository extends Mock implements OrdersRepository {}

void main() {
  late MockOrdersRepository repo;
  late ProviderContainer container;

  setUp(() {
    repo = MockOrdersRepository();
    container = ProviderContainer(
      overrides: [ordersRepositoryProvider.overrideWithValue(repo)],
    );
    addTearDown(container.dispose);
  });

  test('loads orders on build', () async {
    when(() => repo.list(page: any(named: 'page')))
        .thenAnswer((_) async => Result.ok(
              Paginated(items: [fakeOrder()], total: 1, page: 1),
            ));

    final result = await container.read(ordersViewModelProvider().future);

    expect(result.items, hasLength(1));
    verify(() => repo.list(page: 1)).called(1);
  });

  test('surfaces network failure as AsyncError', () async {
    when(() => repo.list(page: any(named: 'page')))
        .thenAnswer((_) async => const Result.error(ApiException.network()));

    await expectLater(
      container.read(ordersViewModelProvider().future),
      throwsA(isA<NetworkException>()),
    );
  });
}
```

Use `mocktail` over `mockito` — no codegen, null-safe by design, better syntax. Test the *behaviour* (what state results), not the implementation (which private method ran).

For repositories, test against a mocked `Dio` using `DioAdapter` from `http_mock_adapter`, or better: run your real FastAPI test server and point an integration test at it.

> **Contract drift between client and server is a real failure mode that mocks cannot catch.** If you can generate a Dart client from your OpenAPI schema in CI and diff it, do that.

---

## 10.3 Widget tests

```dart
testWidgets('renders error state with retry', (tester) async {
  final repo = MockOrdersRepository();
  when(() => repo.list(page: any(named: 'page')))
      .thenAnswer((_) async => const Result.error(ApiException.network()));

  await tester.pumpWidget(
    ProviderScope(
      overrides: [ordersRepositoryProvider.overrideWithValue(repo)],
      child: const MaterialApp(home: OrdersScreen()),
    ),
  );

  await tester.pump();                       // let the future resolve

  expect(find.text('You appear to be offline'), findsOneWidget);
  expect(find.byType(OrderTile), findsNothing);

  await tester.tap(find.text('Retry'));
  await tester.pumpAndSettle();

  verify(() => repo.list(page: 1)).called(2);
});
```

Key APIs:

- `pump()` — advances one frame
- `pump(Duration)` — advances time
- `pumpAndSettle()` — pumps until no frames are scheduled. **Hangs on infinite animations** — never use it with a looping `CircularProgressIndicator` still on screen.

Finders: `find.text`, `find.byType`, `find.byKey`, `find.byIcon`, `find.widgetWithText(ElevatedButton, 'Save')`. Prefer `find.byKey` with explicit keys for anything you assert on repeatedly — it survives copy changes.

Test the states, not the pixels: loading renders a spinner, error renders retry, data renders N tiles, tapping a tile calls the right method. That's four tests per screen and they catch real regressions.

---

## 10.4 Golden tests

Golden (snapshot) tests catch unintended visual change. Worth it for a design system, overkill for every screen.

```dart
testWidgets('OrderTile matches golden', (tester) async {
  await tester.pumpWidget(TestApp(child: OrderTile(order: fakeOrder())));
  await expectLater(
    find.byType(OrderTile),
    matchesGoldenFile('goldens/order_tile.png'),
  );
});
```

Regenerate with `flutter test --update-goldens`.

> **Caveat:** goldens are font- and platform-sensitive; run them on a single pinned CI platform or they'll flap. `golden_toolkit` / `alchemist` add device-size matrices and font loading to make this manageable.

---

## 10.5 Integration tests

```dart
// integration_test/login_flow_test.dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('user can log in and see orders', (tester) async {
    await tester.pumpWidget(const ProviderScope(child: App()));
    await tester.pumpAndSettle();

    await tester.enterText(find.byKey(const Key('email')), 'qa@example.com');
    await tester.enterText(find.byKey(const Key('password')), 'password123');
    await tester.tap(find.byKey(const Key('submit')));
    await tester.pumpAndSettle(const Duration(seconds: 5));

    expect(find.text('Orders'), findsOneWidget);
  });
}
```

```bash
flutter test integration_test/
```

Run these against a **seeded staging backend**, not mocks — the point is to catch integration drift. Keep the count small (5–10 max); they're slow and flaky by nature. Firebase Test Lab or a self-hosted emulator matrix runs them across device configs in CI.

---

[← Forms & Real UI Work](09-forms-and-ui.md) · [Index](README.md) · [Next: Performance →](11-performance.md)
