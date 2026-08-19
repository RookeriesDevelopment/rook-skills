# Troubleshooting — Android · Health Connect

## Known exceptions

`rook-sdk` returns custom exceptions wrapped in the `Result` of each function. Handle them in the failure
branch of `fold`. Cause 📜 and fix 🔧 for each:

| Exception | Cause | Fix |
|-----------|-------|-----|
| `HCDateNotValidForEventsException` | The provided `LocalDate` is outside the allowed range for events. | Use a `LocalDate` between 29 days ago (inclusive) and today (inclusive). |
| `HCDateNotValidForSummariesException` | The provided `LocalDate` is outside the allowed range for summaries. | Use a `LocalDate` between 29 days ago (inclusive) and today (inclusive). |
| `HCHttpRequestException` | An HTTP request failed — usually an error on ROOK servers. | Verify the SDK credentials are configured correctly; if they're fine, read the `httpCode` / `httpMessage` properties and report to ROOK support. |
| `HCMissingConfigurationException` | No `RookConfiguration` was provided to the `RookConfigurationManager`. | Provide a `RookConfiguration` and call `setConfiguration` before initializing (see `references/setup-and-init.md`). |
| `HCNotAuthorizedException` | The operation needs an authorization level your `client_uuid` doesn't have. | Verify credentials are configured correctly; if they're fine, report to ROOK support — some features require a higher authorization level than the basic one. |
| `HCNotInitializedException` | The SDK wasn't initialized, or initialization failed. | Call `rookConfigurationManager.initRook()` and wait for a successful result. |
| `HCQuotaExceededException` | The Health Connect request quota was exceeded (calling `RookSyncManager` functions too often). | Stop syncing for a while; the quota restores over time. Wait ~15–90 minutes. |
| `HCRecordsNotFoundException` | No health data was found for the requested data type. | Confirm read permissions are granted; then open the writing app (Fitbit, Google Fit, …), confirm its write permissions, and force a sync (pull-to-refresh). |
| `HCSessionExpiredException` | The SDK session expired and couldn't be auto-recovered. | Re-initialize the SDK with a valid Client UUID and secret; ensure the secret **and package name** are registered in the ROOK Portal. |
| `HCTimeoutException` | An HTTP request timed out — often an unstable/absent internet connection. | Check connectivity and try again. |
| `HCUserNotInitializedException` | The user wasn't initialized, or initialization failed. | Call `rookConfigurationManager.updateUserID()` and wait for a successful result. |
| `HealthConnectNotInstalledException` | Health Connect isn't installed (only Android 13 and below — it's preinstalled on Android 14+). | Prompt the user to install Health Connect (see snippet below). |
| `HealthConnectNotSupportedException` | The device doesn't support Health Connect. | Don't run Health Connect operations; use availability checks to detect these devices (see `references/permissions.md`). |
| `MissingAndroidPermissionsException` | The operation needs Android permissions that aren't granted. | Request the Android permissions (see `references/permissions.md`). |
| `MissingHealthConnectPermissionsException` | The operation needs Health Connect permissions that aren't granted. | Request the Health Connect permissions (see `references/permissions.md`). |

Prompt to install Health Connect (for `HealthConnectNotInstalledException`):

```kotlin
val intent = Intent(
    Intent.ACTION_VIEW,
    Uri.parse("https://play.google.com/store/apps/details?id=com.google.android.apps.healthdata"),
)
```

## Diagnostics (development only)

Call `getDiagnosticState` to inspect the SDK's current state while developing — is it configured, is a user
identified, what's the permission status, and when did background/manual sync last run.

```kotlin
val diagnosticState = rookConfigurationManager.getDiagnosticState().getOrNull()
```

> **Do not use `getDiagnosticState` in production builds** — it's a development-only aid.

The returned `HCDiagnosticState`:

```kotlin
data class HCDiagnosticState(
    val isConfigured: Boolean,      // SDK has been properly configured
    val userIdentified: Boolean,    // a user has been successfully identified
    val permissions: HCDiagnosticStatePermissions,
    val backgroundSync: HCDiagnosticSyncState,
    val manualSync: HCDiagnosticSyncState,
)

enum class HCDiagnosticStatePermissions {
    NOT_REQUESTED, // permissions have not been requested
    REQUESTED,     // requested but not yet decided/granted
    GRANTED,       // explicitly granted by the user
}

data class HCDiagnosticSyncState(
    val enabled: Boolean,   // sync is currently active (background) or triggered at least once (manual)
    val lastSync: Instant?, // when the last sync was triggered, or null if never
)
```

## Best practices

### Core

- **Initialize once** per app launch; keep managers as singletons (DI or ServiceLocator).
- **Register the `user_id` before syncing**; Background Sync does nothing until a `user_id` and the required
  permissions are in place.
- **Enable logging only on debug builds** (`enableLocalLogs()` before `setConfiguration`).
- **Sync summaries once per day** and back off on `HCQuotaExceededException` (see `references/sync.md`).

### Treat Android Steps and Health Connect as separate data sources

Check Health Connect background availability first; if available, show a Health Connect "connect" button that
routes to its permission screen. Then check Android Steps availability; if available, show an Android
connect button that requests Android permissions. This way users only see the connection options their
device actually supports.

### Health Connect permissions screen

Rather than requesting permissions straight from the connect button, take the user to a screen that explains
which permissions will be requested and why. Health Connect's official UI guidelines have promotable
examples: <https://developer.android.com/health-and-fitness/guides/health-connect/design/ui-guidelines#promote>

### Always try to schedule Background Sync

When the user grants background read permission, call `RookBackgroundSyncManager.schedule` immediately **and**
persist their acceptance so you can call `schedule` again on app launch — this re-schedules the sync if the
system killed the process.

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        if (userAllowedBackgroundSync) {
            RookBackgroundSyncManager.schedule(this, enableLogs = isDebug)
        }
    }
}
```

### Prepare for strict battery scenarios

Power-saving mode and OEM battery restrictions can delay or stop the Steps Counter and Background Sync.
Options:

- **Battery optimizations** — ask the user to disable them. On SDK 4.1.0+ this is built in; see the battery
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
- Confirm `CLIENT_UUID` / `SECRET` match the selected `environment` (SANDBOX vs PRODUCTION) and that the
  secret **and package name** are registered in the ROOK Portal for that environment.
- Consult the official ROOK documentation and contact ROOK support with the exception name, `httpCode` /
  `httpMessage` (when present), and reproduction steps.
