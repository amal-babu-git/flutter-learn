# Part 7 — Navigation

[← Networking, Auth & Data](06-networking-auth-data.md) · [Index](README.md) · [Next: Persistence & Offline-First →](08-persistence-offline.md)

---

## 7.1 go_router fundamentals

`go_router` is maintained by the Flutter team and is the right default: URL-based, declarative, deep-link-capable, and web-friendly. It sits on Navigator 2.0 so you never have to touch that API directly.

```dart
// core/router/app_router.dart
final router = GoRouter(
  initialLocation: '/orders',
  debugLogDiagnostics: true,
  routes: [
    GoRoute(
      path: '/login',
      name: 'login',
      builder: (context, state) => const LoginScreen(),
    ),
    GoRoute(
      path: '/orders',
      name: 'orders',
      builder: (context, state) => const OrdersScreen(),
      routes: [
        GoRoute(
          path: ':id',                       // -> /orders/abc123
          name: 'orderDetail',
          builder: (context, state) =>
              OrderDetailScreen(id: state.pathParameters['id']!),
        ),
      ],
    ),
  ],
  errorBuilder: (context, state) => NotFoundScreen(error: state.error),
);

// app.dart
MaterialApp.router(
  routerConfig: router,
  theme: AppTheme.light(),
  darkTheme: AppTheme.dark(),
);
```

Navigation verbs:

```dart
context.go('/orders/42');                  // replace the stack (declarative)
context.push('/orders/42');                // push onto the stack (returns a Future)
context.pushNamed('orderDetail', pathParameters: {'id': '42'});
context.pop(result);
context.replace('/login');
```

Use `go` for top-level navigation, `push` when you want a back-returns-a-value flow (like a picker).

> **Do not pass complex objects through routes.** Pass an id, fetch in the screen. Route state must survive a cold deep link, and an `Order` object doesn't.

---

## 7.2 Auth guards via redirect

All auth gating lives in one `redirect` callback. Screens stay dumb.

```dart
@riverpod
GoRouter appRouter(Ref ref) {
  final auth = ref.watch(authControllerProvider.notifier);

  return GoRouter(
    initialLocation: '/orders',
    refreshListenable: GoRouterRefreshStream(auth.stream),  // re-evaluate on change
    redirect: (context, state) {
      final status = ref.read(authControllerProvider).valueOrNull;
      final loggingIn = state.matchedLocation == '/login';
      final booting   = state.matchedLocation == '/splash';

      if (status == null) return booting ? null : '/splash';

      final authed = status is Authenticated;

      if (!authed && !loggingIn) {
        // Preserve intended destination for post-login return.
        return '/login?from=${Uri.encodeComponent(state.matchedLocation)}';
      }
      if (authed && (loggingIn || booting)) {
        final from = state.uri.queryParameters['from'];
        return from != null ? Uri.decodeComponent(from) : '/orders';
      }
      return null;   // no redirect
    },
    routes: [...],
  );
}
```

`refreshListenable` is the mechanism that makes this reactive: when auth state changes, go_router re-runs `redirect` and the user is moved automatically. A tiny adapter turns a `Stream` into a `Listenable`:

```dart
class GoRouterRefreshStream extends ChangeNotifier {
  GoRouterRefreshStream(Stream<dynamic> stream) {
    notifyListeners();
    _sub = stream.asBroadcastStream().listen((_) => notifyListeners());
  }
  late final StreamSubscription<dynamic> _sub;

  @override
  void dispose() { _sub.cancel(); super.dispose(); }
}
```

**Role-based gating** goes on individual routes:

```dart
GoRoute(
  path: '/admin',
  redirect: (context, state) {
    final session = ref.read(sessionProvider).valueOrNull;
    return session?.hasPermission('admin.access') ?? false ? null : '/forbidden';
  },
  builder: (context, state) => const AdminScreen(),
),
```

And, to say it once more: **this is UX, not security.** Every one of those permissions is re-checked in your FastAPI dependency chain.

---

## 7.3 Shell routes and nested navigation

`StatefulShellRoute` gives you a persistent bottom navigation bar where each tab keeps its own navigation stack and scroll position — the behaviour users expect from native apps.

```dart
StatefulShellRoute.indexedStack(
  builder: (context, state, navigationShell) =>
      AppScaffold(navigationShell: navigationShell),
  branches: [
    StatefulShellBranch(routes: [
      GoRoute(path: '/orders', builder: (c, s) => const OrdersScreen(), routes: [
        GoRoute(path: ':id', builder: (c, s) => OrderDetailScreen(id: s.pathParameters['id']!)),
      ]),
    ]),
    StatefulShellBranch(routes: [
      GoRoute(path: '/inventory', builder: (c, s) => const InventoryScreen()),
    ]),
    StatefulShellBranch(routes: [
      GoRoute(path: '/profile', builder: (c, s) => const ProfileScreen()),
    ]),
  ],
),
```

```dart
class AppScaffold extends StatelessWidget {
  const AppScaffold({super.key, required this.navigationShell});
  final StatefulNavigationShell navigationShell;

  @override
  Widget build(BuildContext context) => Scaffold(
        body: navigationShell,
        bottomNavigationBar: NavigationBar(
          selectedIndex: navigationShell.currentIndex,
          onDestinationSelected: (i) =>
              navigationShell.goBranch(i, initialLocation: i == navigationShell.currentIndex),
          destinations: const [
            NavigationDestination(icon: Icon(Icons.receipt_long), label: 'Orders'),
            NavigationDestination(icon: Icon(Icons.inventory_2), label: 'Inventory'),
            NavigationDestination(icon: Icon(Icons.person), label: 'Profile'),
          ],
        ),
      );
}
```

The `initialLocation: i == currentIndex` trick makes tapping the active tab pop that branch to its root — standard iOS/Android behaviour that users notice when it's missing.

---

## 7.4 Type-safe routes and deep links

String paths are a runtime-error surface. `go_router_builder` generates typed route classes:

```dart
@TypedGoRoute<OrderDetailRoute>(path: '/orders/:id')
class OrderDetailRoute extends GoRouteData {
  const OrderDetailRoute({required this.id, this.tab});
  final String id;
  final String? tab;

  @override
  Widget build(BuildContext context, GoRouterState state) =>
      OrderDetailScreen(id: id, tab: tab);
}

// Call site — compile-checked:
const OrderDetailRoute(id: '42', tab: 'lines').go(context);
```

**Deep links** require platform config:

- **Android App Links** — an `intent-filter` in `AndroidManifest.xml` plus a hosted `/.well-known/assetlinks.json`.
- **iOS Universal Links** — an Associated Domains entitlement plus `/.well-known/apple-app-site-association`.

Since you own the API domain, hosting those two files from FastAPI is trivial — just make sure they're served with `Content-Type: application/json` and no redirect. DevTools has a "Deep links" validator that checks both platforms' configuration for you.

---

[← Networking, Auth & Data](06-networking-auth-data.md) · [Index](README.md) · [Next: Persistence & Offline-First →](08-persistence-offline.md)
