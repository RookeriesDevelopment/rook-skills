# Troubleshooting — Flutter · Apple Health

## Known exceptions (ROOK)

`rook_sdk_apple_health` returns custom exceptions wrapped in the `Future` each function returns — catch
them with `.catchError(...)` or a `try/catch` around `await`. Cause 📜 and fix 🔧 for each:

| Exception | Cause | Fix |
|-----------|-------|-----|
| `ConnectTimeoutException` | An HTTP request waited too long — often an unstable or absent internet connection. | Check connectivity and try again. |
| `DateNotValidForEventsException` | The provided `DateTime` is outside the allowed range for events. | Use a `DateTime` between 29 days ago (inclusive) and today (inclusive). |
| `DateNotValidForSummariesException` | The provided `DateTime` is outside the allowed range for summaries. | Use a `DateTime` between 29 days ago (inclusive) and today (inclusive). |
| `HealthKitNotInstalledException` | Apple Health is not available on the device. | Prompt the user to install / enable Apple Health (not available on iPad or the Simulator). |
| `HttpRequestException` | An HTTP request failed — usually an error on ROOK servers. | Verify the SDK credentials are configured correctly; if they're fine, read the `message` property and report to ROOK support. |
| `MissingConfigurationException` | No `RookConfiguration` was provided to the configuration manager. | Provide a `RookConfiguration` via `setConfiguration` before initializing (see `references/setup-and-init.md`). |
| `MissingPermissionsException` | The operation needs Apple Health permissions that aren't granted. | Request the Apple Health permissions (see `references/permissions.md`). |
| `RecordsNotFoundException` | No health data was found for the requested data type. | Confirm read permissions are granted; then open the app writing to Apple Health, confirm its write permissions, and force a sync (pull-to-refresh). |
| `SDKNotAuthorizedException` | The operation needs an authorization level your `client_uuid` doesn't have. | Verify credentials are configured correctly; if they're fine, report to ROOK support — some features require a higher authorization level than the basic one. |
| `SDKNotInitializedException` | The SDK wasn't initialized, or initialization failed. | Call `AHRookConfigurationManager.initRook()` and wait for a successful result. |
| `SessionExpiredException` | The SDK session expired and couldn't be auto-recovered. | Re-initialize the SDK with a valid Client UUID and secret; ensure the secret **and package name** are registered in the ROOK Portal. |
| `UnknownException` | An unknown error from the OS (iOS) or the platform (Apple Health / HealthKit). | Inspect the message; if it persists, report to ROOK support. |
| `UserNotInitializedException` | The user wasn't initialized, or initialization failed. | Call `AHRookConfigurationManager.updateUserID()` and wait for a successful result. |

## Diagnostics (development only)

Call `getDiagnosticState` to inspect the SDK's current state while developing — is it configured, is a
user identified, what's the permission status, and when did background/manual sync last run.

```dart
void example() async {
  try {
    final diagnosticState = await AHRookConfigurationManager.getDiagnosticState();

    // Log diagnostic state
  } catch (error) {
    // Handle error
  }
}
```

> **Do not use `getDiagnosticState` in production builds** — it's a development-only aid.

The returned `DiagnosticState`:

```dart
final class DiagnosticState {
  final bool isConfigured;      // SDK has been properly configured
  final bool userIdentified;    // a user has been successfully identified
  final DiagnosticStatePermissions permissions;
  final DiagnosticSyncState backgroundSync;
  final DiagnosticSyncState manualSync;
}

enum DiagnosticStatePermissions {
  notRequested, // permissions have not been requested
  requested,    // requested but not yet decided/granted
  granted,      // explicitly granted by the user
}

final class DiagnosticSyncState {
  final bool enabled;        // sync is currently active (background) or triggered at least once (manual)
  final DateTime? lastSync;  // when the last sync was triggered, or null if never
}
```

## Best practices

### Core

- **Initialize once** per app launch — all `AHRookConfigurationManager` / `AHRookSyncManager` calls are
  static, so there's no instance to manage.
- **Register the `user_id` before syncing**; automatic sync does nothing until a `user_id` and the
  required permissions are in place.
- **Enable logging only on debug builds** — `AHRookConfigurationManager.enableNativeLogs()` **before**
  `setConfiguration`, gated on `kDebugMode`.
- **Guard calls with `Platform.isIOS`** — Apple Health / HealthKit exists only on iOS.
- **Test on a physical device** — HealthKit is not available on the iOS Simulator.

### HealthKit specifics

- **You cannot read whether *read* permission was granted.** For privacy, iOS never reports read-access
  status. Don't try to detect it — request permissions, then attempt a sync; if no data comes back,
  treat it as "not granted / no data" (see `RecordsNotFoundException` above).
- **The permission dialog appears only once per data type.** If the user needs to change a decision, they
  must do it in iOS **Settings → Health → Data Access & Devices** — your app can't re-prompt.
- **iOS locks HealthKit while the device is locked.** iOS encrypts HealthKit storage when the device is
  locked, so background reads may fail until the device is unlocked. This is expected iOS behavior.

### Always try to schedule automatic sync

When the user opts into background sync, call `AHRookBackgroundSync.enableBackground` immediately **and**
persist their acceptance so you can call it again from `main` on app launch. Combine it with
`AHRookContinuousUpload.enableContinuousUpload` for the best coverage (see `references/background.md`).

```dart
void main() {
  // Ensure the plugin is ready
  WidgetsFlutterBinding.ensureInitialized();

  if (Platform.isIOS) {
    enableIOSBackgroundSync();
  }

  runApp(App());
}

void enableIOSBackgroundSync() async {
  try {
    final userAllowedBackgroundSync = await AppPreferences().getUserAllowedBackgroundSync();

    if (userAllowedBackgroundSync) {
      await AHRookBackgroundSync.enableBackground(enableNativeLogs: kDebugMode);
    }
  } catch (error) {
    // Log
  }
}
```

> **Background sync silently won't run without the Xcode capabilities.** Confirm **HealthKit → Background
> delivery** and **Background Modes → Background fetch** are enabled (see `references/setup-and-init.md`).

## Still stuck

- Re-check the earlier references: `setup-and-init.md`, `permissions.md`, `sync.md`, `background.md`.
- Confirm `CLIENT_UUID` / `SECRET` match the selected `environment` (sandbox vs production) and that the
  secret **and package name** are registered in the ROOK Portal for that environment.
- Confirm you're testing on a physical iOS device (HealthKit doesn't run on the Simulator).
- Use `getDiagnosticState` (development builds) to confirm the SDK is configured, the user is identified,
  and permissions are granted.
- Tell the developer to read the official ROOK Flutter Apple Health documentation:
  <https://docs.tryrook.io/docs/category/sdks/flutter/apple-health/>
- Contact ROOK support with the exception name, `message` (when present), and reproduction steps.
