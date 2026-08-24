# Troubleshooting — Flutter · Samsung Health

## Known exceptions (ROOK)

`rook_sdk_samsung_health` returns custom exceptions wrapped in the `Future` each function returns — catch
them with `.catchError(...)` or a `try/catch` around `await`. Cause 📜 and fix 🔧 for each:

| Exception | Cause | Fix |
|-----------|-------|-----|
| `ConnectTimeoutException` | An HTTP request waited too long — often an unstable or absent internet connection. | Check connectivity and try again. |
| `DateNotValidForEventsException` | The provided `DateTime` is outside the allowed range for events. | Use a `DateTime` between 29 days ago (inclusive) and today (inclusive). |
| `DateNotValidForSummariesException` | The provided `DateTime` is outside the allowed range for summaries. | Use a `DateTime` between 29 days ago (inclusive) and today (inclusive). |
| `HealthKitDisabledException` | Samsung Health is installed but disabled. | Prompt the user to enable Samsung Health. |
| `HealthKitNotAllowedException` | Your app isn't allowed to use Samsung Health. | Submit your package name and signing key for a Samsung partnership; for testing, enable developer mode in Samsung Health settings (see `references/setup-and-init.md`). |
| `HealthKitNotInstalledException` | Samsung Health isn't installed. | Prompt the user to install Samsung Health. |
| `HealthKitNotReadyException` | Samsung Health is installed but the user hasn't completed onboarding (e.g. accepting the Terms and Conditions). | Prompt the user to open and finish Samsung Health onboarding. |
| `HealthKitOutdatedException` | The installed Samsung Health version is too old. | Prompt the user to update Samsung Health. |
| `HttpRequestException` | An HTTP request failed — usually an error on ROOK servers. | Verify the SDK credentials are configured correctly; if they're fine, read the `message` property and report to ROOK support. |
| `MissingPermissionsException` | The operation needs Samsung Health permissions that aren't granted. | Request the Samsung Health permissions (see `references/permissions.md`). |
| `RecordsNotFoundException` | No health data was found for the requested data type. | Confirm read permissions are granted; then open the app writing to Samsung Health, confirm its write permissions, and force a sync (pull-to-refresh). |
| `SDKNotAuthorizedException` | The operation needs an authorization level your `client_uuid` doesn't have. | Verify credentials are configured correctly; if they're fine, report to ROOK support — some features require a higher authorization level than the basic one. |
| `SDKNotInitializedException` | The SDK wasn't initialized, or initialization failed. | Call `RookSamsung.initRook()` and wait for a successful result. |
| `SessionExpiredException` | The SDK session expired and couldn't be auto-recovered. | Re-initialize the SDK with a valid Client UUID and secret; ensure the secret **and package name** are registered in the ROOK Portal. |
| `UnknownException` | An unknown error from the OS (Android) or the platform (Samsung Health). | Inspect the message; if it persists, report to ROOK support. |
| `UserNotInitializedException` | The user wasn't initialized, or initialization failed. | Call `RookSamsung.updateUserID()` and wait for a successful result. |

## Samsung internal errors

While developing you may also see internal errors from Samsung (`com.samsung.android.sdk.health.data.error`),
each with a numeric code, e.g.:

```text
com.samsung.android.sdk.health.data.error.AuthorizationException: 2003: Could not get policy
```

| Code | Meaning | Notes |
|------|---------|-------|
| 1000 | Invalid UID — the requested data's UID is invalid / not found. | |
| 1001 | Invalid input — data out of range or otherwise unhandleable. | |
| 1002 | Invalid caller — the calling app's UID is invalid (e.g. using the API on another caller's binder). | |
| 2000 | No user permission — a data-access request hasn't acquired permission from the user. | |
| 2001 | Unsupported operation. | |
| 2002 | No ownership to write — an app can only update/delete data it inserted. | ROOK never writes to Samsung Health; documented for completeness. |
| 2003 | Access control — the app isn't allowed to use the feature. | Enable developer mode for testing; in production, request a Samsung partnership. |
| 2004 | Invalid platform signature — the Samsung Health app's signature is invalid (e.g. an improper build installed). | |
| 2005 | Child account access — the Samsung account in Samsung Health is a child account. | |
| 3000 | Platform not installed. | Normally surfaced as `HealthKitNotInstalledException`. |
| 3001 | Old version platform. | Normally surfaced as `HealthKitOutdatedException`. |
| 3002 | Platform disabled. | Normally surfaced as `HealthKitDisabledException`. |
| 3003 | Platform not initialized (onboarding incomplete). | Normally surfaced as `HealthKitNotReadyException`. |
| 9000 | Internal database error in Samsung Health. | |
| 9001 | Invalid encryption key — Samsung Health can't encrypt/decrypt data. | |
| 9002 | Out of space — no room to store more health data. | |
| 9003 | Internal error that won't resolve even on retry. | |
| 9004 | Connection fail — the connection keeps failing. | |
| 9005 | Interrupted — the operation was stopped by an interrupt. | |
| 9006 | Connection timeout. | |

## Missing classes (build error)

If the app fails to compile with an R8 error like:

```text
ERROR: Missing classes detected while running R8. Please add the missing classes...
```

Make sure the version of the `samsung-health-data-api.aar` file matches the `rook_sdk_samsung_health`
version you are using — a mismatch is the usual cause. See the install steps in
`references/setup-and-init.md`.

## Diagnostics (development only)

Call `getDiagnosticState` to inspect the SDK's current state while developing — is it configured, is a user
identified, what's the permission status, and when did background/manual sync last run.

```dart
void example() async {
  try {
    final diagnosticState = await RookSamsung.getDiagnosticState();

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

- **Initialize once** per app launch — all `RookSamsung` calls are static, so there's no instance to manage.
- **Register the `user_id` before syncing**; Background Sync does nothing until a `user_id` and the required
  permissions are in place.
- **Enable logging only on debug builds** (`RookSamsung.enableNativeLogs()` before `initRook`, gated on
  `kDebugMode`).
- **Guard calls with `Platform.isAndroid`** — Samsung Health exists only on Android.
- **Test on a physical device** — Samsung Health doesn't run on emulators, and requires the Samsung Health
  app v6.29+ plus developer mode during development (see `references/setup-and-init.md`).

### Always try to schedule Background Sync

When the user opts into background sync, call `RookSamsung.enableBackground` immediately **and** persist
their acceptance so you can call it again from `main` on app launch — this re-schedules the sync if the
system killed the process.

```dart
void main() {
  // Ensure the plugin is ready
  WidgetsFlutterBinding.ensureInitialized();

  if (Platform.isAndroid) {
    enableAndroidBackgroundSync();
  }

  runApp(App());
}

void enableAndroidBackgroundSync() async {
  try {
    final userAllowedBackgroundSync = await AppPreferences().getUserAllowedBackgroundSync();

    if (userAllowedBackgroundSync) {
      await RookSamsung.enableBackground(enableNativeLogs: kDebugMode);
    }
  } catch (error) {
    // Log
  }
}
```

### Prepare for strict battery scenarios

Power-saving mode and OEM battery restrictions can delay or stop Background Sync. Options:

- **Battery optimizations** — ask the user to disable them. Either use the
  `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS` intent (opens the settings screen; needs per-manufacturer
  instructions) or the `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` intent (an Allow/Deny dialog; requires
  a Play Console justification). On SDK 4.1.0+ the second option is built in — see the battery optimizations
  flow in `references/permissions.md`.
- **Widgets** — an active widget raises your app's resource priority and can trigger refreshes (via
  Background Sync or a manual sync) as often as every 30 minutes (`updatePeriodMillis`). A widget is also a
  good place to show a health summary (latest sleep, yesterday's physical/body).
- **Push notifications** — use FCM (high-priority messages can wake the app even in Doze) to trigger a
  background or manual sync.
- **Brand-specific settings** — a last resort (high user effort); recommend only for power users or when the
  SDK hasn't synced for an extended period. On SDK 4.1.0+ the Auto Start restriction is handled (see
  `references/permissions.md`); <https://dontkillmyapp.com/> helps with manufacturer-specific steps.

## Still stuck

- Re-check the earlier references: `setup-and-init.md`, `permissions.md`, `sync.md`, `background.md`.
- Confirm `CLIENT_UUID` / `SECRET` match the selected `environment` (`sandbox` vs `production`) and that the
  secret **and package name** are registered in the ROOK Portal for that environment.
- Confirm you're testing on a physical device with Samsung Health v6.29+ and developer mode enabled.
- Read the official ROOK Flutter Samsung Health documentation and package reference:
  <https://docs.tryrook.io/docs/category/sdks/flutter/samsung-health/> ·
  <https://pub.dev/packages/rook_sdk_samsung_health>
- Contact ROOK support with the exception name, its `message` (when present), and reproduction steps.
