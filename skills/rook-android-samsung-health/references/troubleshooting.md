# Troubleshooting — Android · Samsung Health

## Known exceptions (ROOK)

`rook-sdk-samsung` returns custom exceptions wrapped in the `Result` of each function. Handle them in the
failure branch of `fold`. Cause 📜 and fix 🔧 for each:

| Exception | Cause | Fix |
|-----------|-------|-----|
| `MissingSamsungHealthPermissionsException` | The operation needs Samsung Health permissions that aren't granted. | Request the Samsung Health permissions (see `references/permissions.md`). |
| `SamsungHealthDisabledException` | Samsung Health is installed but disabled. | Prompt the user to enable Samsung Health. |
| `SamsungHealthNotAllowedException` | Your app isn't allowed to use Samsung Health. | Submit your package name and signing key for a Samsung partnership; for testing, enable developer mode in Samsung Health settings (see `references/setup-and-init.md`). |
| `SamsungHealthNotInstalledException` | Samsung Health isn't installed. | Prompt the user to install Samsung Health. |
| `SamsungHealthNotReadyException` | Samsung Health is installed but the user hasn't completed onboarding (e.g. accepting the Terms and Conditions). | Prompt the user to open and finish Samsung Health onboarding. |
| `SamsungHealthOutdatedException` | The installed Samsung Health version is too old. | Prompt the user to update Samsung Health. |
| `SHDateNotValidForEventsException` | The provided `LocalDate` is outside the allowed range for events. | Use a `LocalDate` between 29 days ago (inclusive) and today (inclusive). |
| `SHDateNotValidForSummariesException` | The provided `LocalDate` is outside the allowed range for summaries. | Use a `LocalDate` between 29 days ago (inclusive) and today (inclusive). |
| `SHHttpRequestException` | An HTTP request failed — usually an error on ROOK servers. | Verify the SDK credentials are configured correctly; if they're fine, read the `httpCode` / `httpMessage` properties and report to ROOK support. |
| `SHNotAuthorizedException` | The operation needs an authorization level your `client_uuid` doesn't have. | Verify credentials are configured correctly; if they're fine, report to ROOK support — some features require a higher authorization level than the basic one. |
| `SHNotInitializedException` | The SDK wasn't initialized, or initialization failed. | Call `rookSamsung.initRook()` and wait for a successful result. |
| `SHRecordsNotFoundException` | No health data was found for the requested data type. | Confirm read permissions are granted; then open the app writing to Samsung Health, confirm its write permissions, and force a sync (pull-to-refresh). |
| `SHSessionExpiredException` | The SDK session expired and couldn't be auto-recovered. | Re-initialize the SDK with a valid Client UUID and secret; ensure the secret **and package name** are registered in the ROOK Portal. |
| `SHTimeoutException` | An HTTP request timed out — often an unstable/absent internet connection. | Check connectivity and try again. |
| `SHUserNotInitializedException` | The user wasn't initialized, or initialization failed. | Call `rookSamsung.updateUserID()` and wait for a successful result. |

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
| 3000 | Platform not installed. | Normally surfaced as `SamsungHealthNotInstalledException`. |
| 3001 | Old version platform. | Normally surfaced as `SamsungHealthOutdatedException`. |
| 3002 | Platform disabled. | Normally surfaced as `SamsungHealthDisabledException`. |
| 3003 | Platform not initialized (onboarding incomplete). | Normally surfaced as `SamsungHealthNotReadyException`. |
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

Make sure the version of the `samsung-health-data-api.aar` file matches the `rook-sdk-samsung` version you
are using — a mismatch is the usual cause. See the install steps in `references/setup-and-init.md`.

## Diagnostics (development only)

Call `getDiagnosticState` to inspect the SDK's current state while developing — is it configured, is a user
identified, what's the permission status, and when did background/manual sync last run.

```kotlin
val diagnosticState = rookSamsung.getDiagnosticState().getOrNull()
```

> **Do not use `getDiagnosticState` in production builds** — it's a development-only aid.

The returned `SHDiagnosticState`:

```kotlin
data class SHDiagnosticState(
    val isConfigured: Boolean,      // SDK has been properly configured
    val userIdentified: Boolean,    // a user has been successfully identified
    val permissions: SHDiagnosticStatePermissions,
    val backgroundSync: SHDiagnosticSyncState,
    val manualSync: SHDiagnosticSyncState,
)

enum class SHDiagnosticStatePermissions {
    NOT_REQUESTED, // permissions have not been requested
    REQUESTED,     // requested but not yet decided/granted
    GRANTED,       // explicitly granted by the user
}

data class SHDiagnosticSyncState(
    val enabled: Boolean,   // sync is currently active (background) or triggered at least once (manual)
    val lastSync: Instant?, // when the last sync was triggered, or null if never
)
```

## Best practices

### Core

- **Initialize once** per app launch; keep `RookSamsung` as a singleton (DI or ServiceLocator).
- **Register the `user_id` before syncing**; Background Sync does nothing until a `user_id` and the required
  permissions are in place.
- **Enable logging only on debug builds** (`enableLocalLogs()` before `initRook`).
- **Test on a physical device** — Samsung Health doesn't run on emulators, and requires the Samsung Health
  app v6.29+ plus developer mode during development (see `references/setup-and-init.md`).

### Always try to schedule Background Sync

When the user opts into background sync, call `RookSamsungObject.schedule` immediately **and** persist their
acceptance so you can call `schedule` again on app launch — this re-schedules the sync if the system killed
the process.

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        if (userAllowedBackgroundSync) {
            RookSamsungObject.schedule(this, enableLogs = isDebug)
        }
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
- Confirm `CLIENT_UUID` / `SECRET` match the selected `environment` (SANDBOX vs PRODUCTION) and that the
  secret **and package name** are registered in the ROOK Portal for that environment.
- Confirm you're testing on a physical device with Samsung Health v6.29+ and developer mode enabled.
- Use `getDiagnosticState` (development builds) to confirm the SDK is configured, the user is identified, and
  permissions are granted.
- Tell the developer to read the official ROOK Samsung Health documentation:
  <https://docs.tryrook.io/docs/category/sdks/android/samsung-health/>
- Contact ROOK support with the exception name, `httpCode` / `httpMessage` (when present), and
  reproduction steps.
