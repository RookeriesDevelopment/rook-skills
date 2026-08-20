# Availability & permissions — Android · Samsung Health

Prerequisite: the SDK is initialized and a `user_id` is registered (see `references/setup-and-init.md`).

## Recommended flow

1. **Build the permission set** — the `Set<SamsungHealthPermission>` your app actually needs.
2. **Check availability** — is the Samsung Health app installed, up to date, enabled, and onboarded?
3. **Check permissions** — does the app already have what it needs? If so, skip the request.
4. **Request permissions** — request the Samsung Health permissions; listen for the result via a
   `BroadcastReceiver`.
5. **(Optional) Optimization permissions** — exact alarms, battery, and OEM auto-start, to keep
   **Background Sync** reliable on battery-constrained devices.

## RookSamsung instance vs. RookSamsungObject

All the functions below exist on both the `RookSamsung` instance (recommended as a singleton via DI /
ServiceLocator) and the `RookSamsungObject` singleton (pass a `Context` on each call). The snippets use an
instance named `rookSamsung`.

## Check availability

Before requesting permissions, confirm the Samsung Health app is installed and ready. Call
`checkSamsungHealthAvailability`, which returns a `SamsungHealthAvailability`:

| Status          | Meaning                                                                  | What to do                                                     |
|-----------------|--------------------------------------------------------------------------|---------------------------------------------------------------|
| `INSTALLED`     | Samsung Health is installed and ready to be used.                        | Proceed to check/request permissions.                         |
| `NOT_INSTALLED` | Samsung Health is not installed.                                         | Prompt the user to install Samsung Health.                    |
| `OUTDATED`      | The installed Samsung Health version is too old.                         | Prompt the user to update Samsung Health.                     |
| `DISABLED`      | Samsung Health is disabled.                                              | Prompt the user to enable Samsung Health.                     |
| `NOT_READY`     | The user hasn't completed onboarding (e.g. Samsung Health Terms).        | Prompt the user to open and finish Samsung Health onboarding. |

```kotlin
rookSamsung.checkSamsungHealthAvailability().fold(
    { availability ->
        when (availability) {
            SamsungHealthAvailability.INSTALLED -> { /* Proceed to check/request permissions */ }
            SamsungHealthAvailability.NOT_INSTALLED -> { /* Prompt to install Samsung Health */ }
            SamsungHealthAvailability.OUTDATED -> { /* Prompt to update Samsung Health */ }
            SamsungHealthAvailability.DISABLED -> { /* Prompt to enable Samsung Health */ }
            SamsungHealthAvailability.NOT_READY -> { /* Prompt to finish Samsung Health onboarding */ }
        }
    },
    {
        // Handle error
    },
)
```

## Samsung Health permissions

Each Samsung Health data type is guarded by its own permission. Unlike some SDKs, Samsung Health
permissions are **not** declared in the manifest — you build a `Set<SamsungHealthPermission>` in code and
pass it to every check/request call. The full set the SDK can use:

`ACTIVITY_SUMMARY`, `BLOOD_GLUCOSE`, `BLOOD_OXYGEN`, `BLOOD_PRESSURE`, `BODY_COMPOSITION`, `EXERCISE`,
`EXERCISE_LOCATION`, `FLOORS_CLIMBED`, `HEART_RATE`, `NUTRITION`, `SLEEP`, `SLEEP_APNEA`, `STEPS`,
`WATER_INTAKE`, `BODY_TEMPERATURE`.

```kotlin
val samsungPermissions: Set<SamsungHealthPermission> = setOf(
    SamsungHealthPermission.ACTIVITY_SUMMARY,
    SamsungHealthPermission.BLOOD_GLUCOSE,
    SamsungHealthPermission.BLOOD_OXYGEN,
    SamsungHealthPermission.BLOOD_PRESSURE,
    SamsungHealthPermission.BODY_COMPOSITION,
    SamsungHealthPermission.EXERCISE,
    SamsungHealthPermission.EXERCISE_LOCATION,
    SamsungHealthPermission.FLOORS_CLIMBED,
    SamsungHealthPermission.HEART_RATE,
    SamsungHealthPermission.NUTRITION,
    SamsungHealthPermission.SLEEP,
    SamsungHealthPermission.SLEEP_APNEA,
    SamsungHealthPermission.STEPS,
    SamsungHealthPermission.WATER_INTAKE,
    SamsungHealthPermission.BODY_TEMPERATURE,
)
```

> **Keep the set consistent with your Samsung partner registration.** If you excluded a data type from the
> Samsung Partnership Request Form (see `references/setup-and-init.md`), **do not** include it here — passing
> a permission you didn't register will fail.

Reuse the **same** `samsungPermissions` set for the check and request calls below.

### Check permissions

Check whether **all** the requested permissions are granted with `checkSamsungHealthPermissions`:

```kotlin
val hasAllSamsungHealthPermissions = rookSamsung.checkSamsungHealthPermissions(samsungPermissions).fold(
    { hasAllPermissions -> hasAllPermissions },
    { throwable -> false },
)
```

Check whether **at least one** permission is granted with `checkSamsungHealthPermissionsPartially`:

```kotlin
val hasSomeSamsungHealthPermissions = rookSamsung.checkSamsungHealthPermissionsPartially(samsungPermissions).fold(
    { hasSomePermissions -> hasSomePermissions },
    { throwable -> false },
)
```

### Request permissions

`requestSamsungHealthPermissions` returns a `SHRequestPermissionsStatus`:

- `ALREADY_GRANTED` — permissions were already granted; no request was sent.
- `REQUEST_SENT` — the request dialog was shown; the result arrives via a `BroadcastReceiver`.

Register a receiver for `SamsungHealthPermission.ACTION_SAMSUNG_HEALTH_PERMISSIONS` to receive the outcome.
Its extras:

- `EXTRA_SAMSUNG_HEALTH_PERMISSIONS_GRANTED` — all Samsung Health permissions were granted.
- `EXTRA_SAMSUNG_HEALTH_PERMISSIONS_PARTIALLY_GRANTED` — at least one was granted (also `true` when all are).

```kotlin
// 1. Create the broadcast receiver
private val samsungHealthBroadcastReceiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        val allPermissionsGranted = intent?.getBooleanExtra(
            SamsungHealthPermission.EXTRA_SAMSUNG_HEALTH_PERMISSIONS_GRANTED, false,
        ) ?: false

        val permissionsPartiallyGranted = intent?.getBooleanExtra(
            SamsungHealthPermission.EXTRA_SAMSUNG_HEALTH_PERMISSIONS_PARTIALLY_GRANTED, false,
        ) ?: false

        // Being flexible: keep working even if only some permissions were granted
        val permissionsGranted = allPermissionsGranted || permissionsPartiallyGranted

        // Update your UI
    }
}

// 2. Register it (e.g. in onCreate)
ContextCompat.registerReceiver(
    context,
    samsungHealthBroadcastReceiver,
    IntentFilter(SamsungHealthPermission.ACTION_SAMSUNG_HEALTH_PERMISSIONS),
    ContextCompat.RECEIVER_EXPORTED,
)

// 3. Request permissions
rookSamsung.requestSamsungHealthPermissions(samsungPermissions).fold(
    {
        when (it) {
            SHRequestPermissionsStatus.ALREADY_GRANTED -> { /* Already granted — update UI */ }
            SHRequestPermissionsStatus.REQUEST_SENT -> { /* Wait for the broadcast result */ }
        }
    },
    { /* Handle error */ },
)

// 4. Unregister it (e.g. in onDestroy)
context.unregisterReceiver(samsungHealthBroadcastReceiver)
```

### Customizing permissions

Request only what your app needs: build a smaller `Set<SamsungHealthPermission>` and pass it to the
check/request functions — they adapt automatically. For example, activity-focused apps:

```kotlin
val samsungPermissions: Set<SamsungHealthPermission> = setOf(
    SamsungHealthPermission.ACTIVITY_SUMMARY,
    SamsungHealthPermission.BODY_COMPOSITION,
    SamsungHealthPermission.EXERCISE,
    SamsungHealthPermission.EXERCISE_LOCATION,
    SamsungHealthPermission.HEART_RATE,
    SamsungHealthPermission.SLEEP,
    SamsungHealthPermission.STEPS,
)
```

## Optimization permissions (optional)

Power-saving mode and OEM-specific energy restrictions can delay or stop **Background Sync**. These
permissions are **optional** — Background Sync works without them — but if users report interruptions,
expose a button (e.g. on a support screen) to grant them.

### Exact alarms

Background Sync uses `SCHEDULE_EXACT_ALARM` (already in the SDK's manifest) to keep its background work
alive under battery constraints. If you grant this permission you must call `schedule` again afterwards
(see `references/background.md`).

```kotlin
val hasAlarmPermissions = rookSamsung.checkExactAlarmPermissions()

val requestPermissionsStatus = rookSamsung.requestExactAlarmPermissions()
when (requestPermissionsStatus) {
    SHRequestPermissionsStatus.ALREADY_GRANTED -> { /* Granted */ }
    SHRequestPermissionsStatus.REQUEST_SENT -> {
        // Special app-access permission: no callback — re-check with checkExactAlarmPermissions
    }
}
```

> On API 26–30 exact alarms are unrestricted, so these calls always return `true` /
> `SHRequestPermissionsStatus.REQUEST_SENT`.

### Battery optimizations

```kotlin
val batteryOptimizationsDisabled = rookSamsung.checkBatteryOptimizationsDisabled()

val requestPermissionsStatus = rookSamsung.requestDisableBatteryOptimizations()
when (requestPermissionsStatus) {
    SHRequestPermissionsStatus.ALREADY_GRANTED -> { /* Already disabled */ }
    SHRequestPermissionsStatus.REQUEST_SENT -> {
        // Special app-access permission: no callback — re-check with checkBatteryOptimizationsDisabled
    }
}
```

### Auto start (OEM-specific)

Brands like OPPO, OnePlus, and Xiaomi enforce a custom Auto Start restriction that limits background
execution. Detect whether the device's OEM has such a settings screen with `requiresOemAutoStartSetup`,
then open it with `openOemAutoStartSetup`:

```kotlin
val hasAutoStartRestriction = rookSamsung.requiresOemAutoStartSetup()

rookSamsung.openOemAutoStartSetup().fold(
    { requestPermissionsStatus ->
        when (requestPermissionsStatus) {
            SHRequestPermissionsStatus.ALREADY_GRANTED -> { /* Device has no OEM screen */ }
            SHRequestPermissionsStatus.REQUEST_SENT -> {
                // OEM-specific screen: the user's choice can't be read back
            }
        }
    },
    { /* Handle error */ },
)
```

> A `true` from `requiresOemAutoStartSetup` means the OEM management system **exists**, not that auto start
> is currently blocked for your app. Use it to decide whether to prompt the user.
