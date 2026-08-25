# Troubleshooting — Flutter · Health Connect

## Known exceptions

`rook_sdk_health_connect` throws custom exceptions from the `Future` returned by each function. Handle them
in `catchError` (or the `catch` block of a `try`/`await`). Cause 📜 and fix 🔧 for each:

| Exception | Cause | Fix |
|-----------|-------|-----|
| `ConnectTimeoutException` | An HTTP request waited too long — often an unstable or absent internet connection. | Check connectivity and try again. |
| `DateNotValidForEventsException` | The provided `DateTime` is outside the allowed range for events. | Use a `DateTime` between 29 days ago (inclusive) and today (inclusive). |
| `DateNotValidForSummariesException` | The provided `DateTime` is outside the allowed range for summaries. | Use a `DateTime` between 29 days ago (inclusive) and today (inclusive). |
| `HealthConnectQuotaExceededException` | The Health Connect request quota was exceeded (calling `HCRookSyncManager` functions too often). | Stop syncing for a while; the quota restores over time. Wait ~15–90 minutes. |
| `HealthKitNotInstalledException` | Health Connect is not installed. | Prompt the user to install Health Connect (see the Play Store listing below). |
| `HealthKitNotSupportedException` | The device doesn't support Health Connect. | Don't run Health Connect operations; use availability checks to detect these devices (see `references/permissions.md`). |
| `HttpRequestException` | An HTTP request failed — usually an error on ROOK servers. | Verify the SDK credentials are configured correctly; if they're fine, read the `message` property and report to ROOK support. |
| `MissingConfigurationException` | No `RookConfiguration` was provided to the `HCRookConfigurationManager`. | Provide a `RookConfiguration` and call `setConfiguration` before initializing (see `references/setup-and-init.md`). |
| `MissingPermissionsException` | The operation needs Health Connect permissions that aren't granted. | Request the Health Connect permissions (see `references/permissions.md`). |
| `RecordsNotFoundException` | No health data was found for the requested data type. | Confirm read permissions are granted; then open the writing app (Fitbit, Google Fit, …), confirm its write permissions, and force a sync (pull-to-refresh). |
| `SDKNotAuthorizedException` | The operation needs an authorization level your `client_uuid` doesn't have. | Verify credentials are configured correctly; if they're fine, report to ROOK support — some features require a higher authorization level than the basic one. |
| `SDKNotInitializedException` | The SDK wasn't initialized, or initialization failed. | Call `HCRookConfigurationManager.initRook()` and wait for a successful result. |
| `SessionExpiredException` | The SDK session expired and couldn't be auto-recovered. | Re-initialize the SDK with a valid Client UUID and secret; ensure the secret **and package name** are registered in the ROOK Portal. |
| `UserNotInitializedException` | The user wasn't initialized, or initialization failed. | Call `HCRookConfigurationManager.updateUserID()` and wait for a successful result. |
| `UnknownException` | An unknown error from the OS (Android) or the platform (Health Connect). | Read the `message` property and, if it persists, report to ROOK support. |

The Health Connect Play Store listing (for `HealthKitNotInstalledException`) is
`com.google.android.apps.healthdata`:
`https://play.google.com/store/apps/details?id=com.google.android.apps.healthdata`

## Diagnostics (development only)

Call `getDiagnosticState` on `HCRookConfigurationManager` to inspect the SDK's current state while
developing — is it configured, is a user identified, what's the permission status, and when did
background/manual sync last run.

```dart
void example() async {
  try {
    final diagnosticState = await HCRookConfigurationManager.getDiagnosticState();

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
  final bool isConfigured;    // SDK has been properly configured
  final bool userIdentified;  // a user has been successfully identified
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
  final bool enabled;       // sync is active (background) or triggered at least once (manual)
  final DateTime? lastSync; // when the last sync was triggered, or null if never
}
```

## Best practices

### Core

- **Initialize once** per app launch; `initRook()` makes an HTTP request (see `references/setup-and-init.md`).
- **Register the `user_id` before syncing**; Background Sync does nothing until a `user_id` and the required
  permissions are in place.
- **Enable native logs only on debug builds** (`enableNativeLogs()` before `setConfiguration`).
- **Sync summaries once per day** and back off on `HealthConnectQuotaExceededException` (see
  `references/sync.md`).

### Treat Android Steps and Health Connect as separate data sources

Think of [Android Steps](background-steps.md) and Health Connect as two different data sources. Check Health
Connect **background read** availability first; if available, show a Health Connect connection button that
routes to its permission screen. Then check Android Steps availability; if available, show an Android
connection button that requests Android permissions. This way users only see the connection options their
device actually supports.

### Health Connect permissions screen

Rather than requesting permissions straight from the connection button, take the user to a screen that
explains which permissions will be requested and why. Health Connect's official UI guidelines have
promotable examples:
<https://developer.android.com/health-and-fitness/guides/health-connect/design/ui-guidelines#promote>

### Always try to schedule Background Sync

When the user grants background read permission, call `HCRookBackgroundSync.enableBackground` immediately
**and** persist their acceptance so you can call `enableBackground` again on app launch — this re-schedules
the sync if the system killed the process.

```dart
void main() {
  // Ensure that the plugin is ready
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
      await HCRookBackgroundSync.enableBackground(enableNativeLogs: isDebug);
    }
  } catch (error) {
    // Log
  }
}
```

### Prepare for strict battery scenarios

Power-saving mode and OEM battery restrictions can delay or stop the Steps Counter and Background Sync.
Options:

- **Battery optimizations** — ask the user to disable them. There are two intents you can use:
  `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS` (opens the settings screen; requires per-manufacturer
  instructions) or `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` (shows an Allow/Deny dialog but must be
  justified in the Play Console). On SDK 4.1.0+ the second option is built in — see the battery
  optimizations flow in `references/permissions.md`.
- **Widgets** — an active widget raises your app's resource priority and can trigger refreshes (via
  Background Sync or a manual sync) as often as every 30 minutes (`updatePeriodMillis`).
- **Push notifications** — use FCM (high-priority messages can wake the app even in Doze) to trigger a
  background or manual sync.
- **Brand-specific settings** — a last resort (high user effort); recommend only for power users or when the
  SDK hasn't synced for an extended period. On SDK 4.1.0+ the Auto Start restriction is handled (see
  `references/permissions.md`); <https://dontkillmyapp.com/> helps with manufacturer-specific steps.

## Still stuck

- Re-check the earlier references: `setup-and-init.md`, `permissions.md`, `sync.md`, `background.md` (and
  `background-steps.md` if you enabled the optional Android step tracker).
- Confirm `CLIENT_UUID` / `SECRET` match the selected `environment` (sandbox vs production) and that the
  secret **and package name** are registered in the ROOK Portal for that environment.
- Use `getDiagnosticState` (development builds) to confirm the SDK is configured, the user is identified, and
  permissions are granted.
- Read the official ROOK Flutter Health Connect documentation:
  <https://docs.tryrook.io/docs/category/sdks/flutter/health-connect/>
- Contact ROOK support with the exception name, `httpCode` / `httpMessage` (when present), and
  reproduction steps.
