# Part 9 — Forms and Real UI Work

[← Persistence & Offline-First](08-persistence-offline.md) · [Index](README.md) · [Next: Testing →](10-testing.md)

---

## 9.1 Forms and validation

```dart
class LoginForm extends ConsumerStatefulWidget {
  const LoginForm({super.key});
  @override
  ConsumerState<LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends ConsumerState<LoginForm> {
  final _formKey = GlobalKey<FormState>();
  final _email = TextEditingController();
  final _password = TextEditingController();
  bool _obscure = true;

  @override
  void dispose() {
    _email.dispose();
    _password.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) return;
    await ref.read(authControllerProvider.notifier)
        .login(_email.text.trim(), _password.text);
  }

  @override
  Widget build(BuildContext context) {
    final state = ref.watch(authControllerProvider);
    // Server-side field errors, surfaced from a 422.
    final fieldErrors = switch (state) {
      AsyncError(error: ValidationException(:final fields)) => fields,
      _ => const <String, List<String>>{},
    };

    return Form(
      key: _formKey,
      autovalidateMode: AutovalidateMode.onUserInteraction,
      child: Column(
        children: [
          TextFormField(
            controller: _email,
            keyboardType: TextInputType.emailAddress,
            autofillHints: const [AutofillHints.username],
            textInputAction: TextInputAction.next,
            decoration: InputDecoration(
              labelText: 'Email',
              errorText: fieldErrors['email']?.first,
            ),
            validator: (v) => switch (v) {
              null || '' => 'Email is required',
              final s when !s.contains('@') => 'Enter a valid email',
              _ => null,
            },
          ),
          const SizedBox(height: 16),
          TextFormField(
            controller: _password,
            obscureText: _obscure,
            autofillHints: const [AutofillHints.password],
            textInputAction: TextInputAction.done,
            onFieldSubmitted: (_) => _submit(),
            decoration: InputDecoration(
              labelText: 'Password',
              errorText: fieldErrors['password']?.first,
              suffixIcon: IconButton(
                icon: Icon(_obscure ? Icons.visibility : Icons.visibility_off),
                onPressed: () => setState(() => _obscure = !_obscure),
              ),
            ),
            validator: (v) =>
                (v == null || v.length < 8) ? 'Minimum 8 characters' : null,
          ),
          const SizedBox(height: 24),
          FilledButton(
            onPressed: state.isLoading ? null : _submit,
            child: state.isLoading
                ? const SizedBox.square(
                    dimension: 20,
                    child: CircularProgressIndicator(strokeWidth: 2))
                : const Text('Sign in'),
          ),
        ],
      ),
    );
  }
}
```

Details that separate a professional form from a student one:

- `autofillHints` — enables password managers
- `textInputAction` + `onFieldSubmitted` — keyboard "next"/"done" chaining
- `autovalidateMode: onUserInteraction` — don't shout at users before they've typed
- Disabled button while submitting
- Server field errors rendered on the correct field

For very large forms, `flutter_form_builder` or `reactive_forms` reduce boilerplate — but the built-in `Form` handles most cases and adds no dependency.

---

## 9.2 Lists, tables, and large data

**Always `ListView.builder`** for anything unbounded — it only builds visible items.

```dart
ListView.separated(
  padding: const EdgeInsets.symmetric(vertical: 8),
  itemCount: orders.length,
  separatorBuilder: (_, __) => const Divider(height: 1),
  itemBuilder: (context, i) => OrderTile(key: ValueKey(orders[i].id), order: orders[i]),
);
```

**Tabular ERP data on mobile** is a genuine design problem. Options:

- `DataTable` — fine for < 50 rows, non-lazy. Wrap in `SingleChildScrollView(scrollDirection: Axis.horizontal)`.
- `PaginatedDataTable` — server-side pagination built in, but dated styling.
- `two_dimensional_scrollables` (`TableView`) — first-party, lazily builds in both axes with frozen headers. **This is the right answer for large grids** and is what to reach for when someone asks for "the spreadsheet view."
- Honest alternative: on phones, don't render a table. Render cards with the 3–4 fields that matter and put the full grid on tablet/desktop breakpoints.

**Pull to refresh:**

```dart
RefreshIndicator(
  onRefresh: () => ref.refresh(ordersProvider.future),
  child: ListView.builder(...),
);
```

**Loading skeletons** beat spinners for perceived performance — `skeletonizer` wraps your real widget tree and shimmer-izes it, so the skeleton can't drift out of sync with the layout.

---

## 9.3 Responsive and adaptive layout

Two distinct concepts, often conflated:

- **Responsive** = layout adapts to size (phone → tablet → desktop).
- **Adaptive** = behaviour/appearance adapts to platform conventions (Cupertino scrollbars on iOS, right-click menus on desktop, keyboard shortcuts).

```dart
class Breakpoints {
  static const compact = 600.0;    // phone
  static const medium = 840.0;     // tablet / foldable
  static const expanded = 1200.0;  // desktop
}

class AdaptiveShell extends StatelessWidget {
  const AdaptiveShell({super.key, required this.child});
  final Widget child;

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final w = constraints.maxWidth;
        if (w >= Breakpoints.expanded) {
          return Row(children: [
            const SizedBox(width: 260, child: AppNavigationDrawer()),
            Expanded(child: child),
          ]);
        }
        if (w >= Breakpoints.compact) {
          return Row(children: [const AppNavigationRail(), Expanded(child: child)]);
        }
        return Scaffold(body: child, bottomNavigationBar: const AppBottomNav());
      },
    );
  }
}
```

Use `LayoutBuilder` (local constraints) over `MediaQuery` (window size) when you're inside a split view — otherwise a pane thinks it's as wide as the whole screen. Always wrap edge content in `SafeArea`. Test with a foldable emulator if your users have them; `MediaQuery.of(context).displayFeatures` exposes hinges.

---

## 9.4 Internationalization

Even single-language apps benefit: it centralizes copy, which makes it reviewable and changeable without hunting through widgets.

```yaml
# l10n.yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
nullable-getter: false
```

```json
// lib/l10n/app_en.arb
{
  "ordersTitle": "Orders",
  "orderCount": "{count, plural, =0{No orders} =1{1 order} other{{count} orders}}",
  "@orderCount": { "placeholders": { "count": { "type": "int" } } }
}
```

```dart
Text(AppLocalizations.of(context).orderCount(orders.length));
```

Use `intl`'s `NumberFormat`/`DateFormat` for currency and dates rather than string interpolation — an ERP that prints `1234.5` instead of `₹1,234.50` looks broken. Set `NumberFormat.currency(locale: ..., symbol: ...)` from the tenant's configured currency, not the device locale.

---

[← Persistence & Offline-First](08-persistence-offline.md) · [Index](README.md) · [Next: Testing →](10-testing.md)
