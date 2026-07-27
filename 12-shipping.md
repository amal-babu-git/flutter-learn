# Part 12 — Ship It

[← Performance](11-performance.md) · [Index](README.md) · [Next: Capstone & Resources →](13-capstone-and-resources.md)

---

## 12.1 Secrets and configuration

**A compiled Flutter app contains no secrets.** Anyone can unzip an APK, run `strings`, and read every `--dart-define` value, every hardcoded constant, and much of your Dart symbol table. This is identical to the `NEXT_PUBLIC_*` rule you already follow.

| Item | Where it lives |
|---|---|
| API base URL | `--dart-define` — fine, not secret |
| Feature flags | `--dart-define` or a remote config endpoint |
| Third-party **publishable** keys (Maps, Sentry DSN) | In the app, restricted by bundle id / referrer in the vendor console |
| Third-party **secret** keys, DB creds, signing keys | **Your FastAPI backend. Never the app.** |
| Payment/webhook secrets | Backend |
| Signing keystores | CI secret store; never in git |

Add `--obfuscate --split-debug-info=build/symbols` to release builds. It renames symbols, raising the effort of reverse engineering — it is a speed bump, not a lock. Keep the symbol files; you need them to de-obfuscate crash reports.

For genuinely sensitive client-side operations, the answer is architectural: move the operation server-side and give the client a narrowly-scoped, short-lived token.

---

## 12.2 Build and release

```bash
# Android — App Bundle for Play Store
flutter build appbundle --release \
  --dart-define=FLAVOR=prod --flavor prod \
  --obfuscate --split-debug-info=build/symbols

# Android — APK for direct distribution (e.g. internal MDM)
flutter build apk --release --split-per-abi \
  --dart-define=FLAVOR=prod --flavor prod

# iOS
flutter build ipa --release \
  --dart-define=FLAVOR=prod --flavor prod \
  --export-options-plist=ios/ExportOptions.plist

# Web (Wasm — significantly faster than the JS output)
flutter build web --wasm --release --dart-define=FLAVOR=prod
```

**Android signing** — `android/key.properties` (gitignored), referenced from `build.gradle.kts`:

```properties
storePassword=***
keyPassword=***
keyAlias=upload
storeFile=/absolute/path/upload-keystore.jks
```

Enable Play App Signing so Google holds the release key; you only manage the upload key. Losing an unmanaged release key means you can never update the app.

**iOS** — Apple Developer account, App Store Connect app record, provisioning profiles. Use `fastlane match` to keep certificates in a shared encrypted repo rather than on one laptop.

**Size:** check with `flutter build apk --analyze-size`. Usual offenders are uncompressed images and unused font glyphs (Flutter tree-shakes icon fonts automatically in release; verify it's not disabled).

---

## 12.3 CI/CD

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.44.x'
          channel: stable
          cache: true

      - run: flutter pub get
      - run: dart run build_runner build --delete-conflicting-outputs
      - run: dart format --output=none --set-exit-if-changed .
      - run: flutter analyze --fatal-infos
      - run: flutter test --coverage
      - uses: codecov/codecov-action@v4

  build-android:
    needs: verify
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with: { flutter-version: '3.44.x', channel: stable, cache: true }
      - run: echo "${{ secrets.KEYSTORE_B64 }}" | base64 -d > android/upload.jks
      - run: flutter build appbundle --release --dart-define=FLAVOR=prod --flavor prod
      - uses: actions/upload-artifact@v4
        with: { name: app-bundle, path: build/app/outputs/bundle/prodRelease/*.aab }
```

iOS builds need a macOS runner (`macos-latest`) and are slower and more expensive; consider Codemagic or Bitrise, which are Flutter-native and handle signing well.

**Distribution:** Firebase App Distribution or TestFlight for internal builds; `fastlane` for store submission. For an internal ERP, MDM distribution (Intune, Jamf) often beats the public stores entirely — no review latency.

**Versioning:** `version: 1.4.2+87` in `pubspec.yaml` — `1.4.2` is the user-visible name, `87` is the build number, which must increase monotonically for every upload. Derive the build number from the CI run number.

---

## 12.4 Observability

```dart
// bootstrap.dart
Future<void> bootstrap(Widget Function() builder) async {
  WidgetsFlutterBinding.ensureInitialized();

  // Flutter framework errors (build/layout/paint).
  FlutterError.onError = (details) {
    FlutterError.presentError(details);
    Sentry.captureException(details.exception, stackTrace: details.stack);
  };

  // Errors from the platform / outside the Flutter zone.
  PlatformDispatcher.instance.onError = (error, stack) {
    Sentry.captureException(error, stackTrace: stack);
    return true;
  };

  await SentryFlutter.init((o) {
    o.dsn = const String.fromEnvironment('SENTRY_DSN');
    o.tracesSampleRate = 0.2;
    o.environment = AppConfig.current().flavor.name;
  });

  runApp(builder());
}
```

Upload your obfuscation symbols (`build/symbols`) to Sentry/Crashlytics or every stack trace is unreadable. Automate it in the release job.

**What to instrument:** app start time, screen transitions, API latency per endpoint (tag with the `X-Request-Id` you already send so you can join to backend traces), error rate by `ApiException` variant, and crash-free session rate.

If you're on Bloc, `BlocObserver` gives you a free breadcrumb trail of every user action preceding a crash — genuinely the best debugging affordance in the ecosystem.

---

## 12.5 Security checklist

- [ ] Tokens in `flutter_secure_storage`, never `SharedPreferences`
- [ ] HTTPS only; consider certificate pinning for high-value APIs (weigh rotation cost)
- [ ] Short-lived access tokens + rotating refresh tokens with server-side reuse detection
- [ ] Logout clears secure storage **and** local DB **and** in-memory providers
- [ ] `--obfuscate --split-debug-info` on release builds
- [ ] No secrets in the binary — audit your `--dart-define`s
- [ ] Every permission re-checked server-side; client gating is UX only
- [ ] Sensitive screens: `FLAG_SECURE` on Android (blocks screenshots/screen sharing) — 3.44 adds first-class sensitive-content protection for exactly this
- [ ] Biometric re-auth (`local_auth`) for high-value actions
- [ ] Root/jailbreak detection if your threat model warrants it
- [ ] Disable logging of request/response bodies in release (`PrettyDioLogger` behind the flavor flag)
- [ ] Dependency audit: `flutter pub outdated`, review transitive packages before adding
- [ ] Deep link parameters validated and never trusted (they're user input)

---

[← Performance](11-performance.md) · [Index](README.md) · [Next: Capstone & Resources →](13-capstone-and-resources.md)
