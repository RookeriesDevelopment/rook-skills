# Availability & permissions — Android · Health Connect

Prerequisite: the SDK is initialized and a `user_id` is registered (see `references/setup-and-init.md`).

## Recommended flow

1. **Check availability** — is Health Connect installed / supported on this device?
2. **Check permissions** — does the app already have what it needs? If so, skip the request.
3. **Request permissions** — request Health Connect (data) permissions; listen for the result via a
   `BroadcastReceiver`.
4. **Handle denial** — if the user denied twice, guide them to open Health Connect settings.
5. **(Optional) Background read** — for background sync, additionally check/request the background-read
   permission.
6. **(Optional) Android + optimization permissions** — needed for background steps / background sync.

## RookPermissionsManager

Use either an instance (recommended as a singleton via DI / ServiceLocator) or the Companion object with a
`context` per call:

```kotlin
// Instance (recommended — provide via DI or a singleton)
val rookPermissionsManager = RookPermissionsManager(context)
rookPermissionsManager.doSomething()

// Or Companion, passing a context each call
RookPermissionsManager.doSomething(context)
```

## Check availability

Call `checkHealthConnectAvailability` before anything else:

| Status          | Meaning                              | What to do                                   |
|-----------------|--------------------------------------|----------------------------------------------|
| `INSTALLED`     | Health Connect APK is installed      | Proceed to check/request permissions         |
| `NOT_INSTALLED` | APK is not installed                 | Prompt the user to install it from the Store |
| `NOT_SUPPORTED` | Device does not support Health Connect | Take the user out of the Health Connect flow |

```kotlin
val message = when (rookPermissionsManager.checkHealthConnectAvailability()) {
    HealthConnectAvailability.INSTALLED -> "Health Connect is installed! You can skip the next step"
    HealthConnectAvailability.NOT_INSTALLED -> "Health Connect is not installed. Please download it from the Play Store"
    else -> "This device is not compatible with Health Connect"
}
```

The Health Connect Play Store listing is
`com.google.android.apps.healthdata` (`https://play.google.com/store/apps/details?id=com.google.android.apps.healthdata`).

## Health Connect permissions

These are the data-read permissions the SDK uses — one per Health Connect data type. They are the same set
declared automatically by `rook-sdk` (see `references/setup-and-init.md`); you can narrow this set under
[Customizing permissions](#customizing-permissions).

`READ_SLEEP`, `READ_STEPS`, `READ_DISTANCE`, `READ_FLOORS_CLIMBED`, `READ_ELEVATION_GAINED`,
`READ_OXYGEN_SATURATION`, `READ_VO2_MAX`, `READ_TOTAL_CALORIES_BURNED`, `READ_ACTIVE_CALORIES_BURNED`,
`READ_HEART_RATE`, `READ_RESTING_HEART_RATE`, `READ_HEART_RATE_VARIABILITY`, `READ_EXERCISE`, `READ_SPEED`,
`READ_WEIGHT`, `READ_HEIGHT`, `READ_BLOOD_GLUCOSE`, `READ_BLOOD_PRESSURE`, `READ_HYDRATION`,
`READ_BODY_TEMPERATURE`, `READ_RESPIRATORY_RATE`, `READ_NUTRITION`, `READ_MENSTRUATION`, `READ_POWER`,
`READ_BASAL_BODY_TEMPERATURE`, `READ_BODY_FAT`, `READ_LEAN_BODY_MASS`, `READ_BODY_WATER_MASS`,
`READ_BONE_MASS`.

### Check permissions

Check whether **all** required permissions are granted:

```kotlin
val hasAllHealthConnectPermissions = rookPermissionsManager.checkHealthConnectPermissions().fold(
    { hasAllPermissions -> hasAllPermissions },
    { throwable -> false },
)
```

Check whether **at least one** permission is granted (`checkHealthConnectPermissionsPartially`):

```kotlin
val hasSomeHealthConnectPermissions = rookPermissionsManager.checkHealthConnectPermissionsPartially().fold(
    { hasSomePermissions -> hasSomePermissions },
    { throwable -> false },
)
```

### Request permissions

`requestHealthConnectPermissions` returns a `RequestPermissionsStatus`:

- `ALREADY_GRANTED` — permissions were already granted; no request was sent.
- `REQUEST_SENT` — the request dialog was shown; the result arrives via a `BroadcastReceiver`.

Register a receiver for `RookPermissionsManager.ACTION_HEALTH_CONNECT_PERMISSIONS` to receive the outcome.
Its extras:

- `EXTRA_HEALTH_CONNECT_PERMISSIONS_GRANTED` — all Health Connect permissions were granted.
- `EXTRA_HEALTH_CONNECT_PERMISSIONS_PARTIALLY_GRANTED` — at least one was granted (also `true` when all are).
- `EXTRA_HEALTH_CONNECT_BACKGROUND_PERMISSION_GRANTED` — background read was granted (always `false` when
  the device doesn't support background read).

```kotlin
// 1. Create the broadcast receiver
private val healthConnectBroadcastReceiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        val allPermissionsGranted = intent?.getBooleanExtra(
            RookPermissionsManager.EXTRA_HEALTH_CONNECT_PERMISSIONS_GRANTED, false,
        ) ?: false

        val permissionsPartiallyGranted = intent?.getBooleanExtra(
            RookPermissionsManager.EXTRA_HEALTH_CONNECT_PERMISSIONS_PARTIALLY_GRANTED, false,
        ) ?: false

        // Being flexible: keep working even if only some permissions were granted
        val permissionsGranted = allPermissionsGranted || permissionsPartiallyGranted

        val backgroundPermissionGranted = intent?.getBooleanExtra(
            RookPermissionsManager.EXTRA_HEALTH_CONNECT_BACKGROUND_PERMISSION_GRANTED, false,
        ) ?: false

        // Update your UI
    }
}

// 2. Register it (e.g. in onCreate)
ContextCompat.registerReceiver(
    context,
    healthConnectBroadcastReceiver,
    IntentFilter(RookPermissionsManager.ACTION_HEALTH_CONNECT_PERMISSIONS),
    ContextCompat.RECEIVER_EXPORTED,
)

// 3. Request permissions
rookPermissionsManager.requestHealthConnectPermissions().fold(
    {
        when (it) {
            RequestPermissionsStatus.ALREADY_GRANTED -> { /* Already granted — update UI */ }
            RequestPermissionsStatus.REQUEST_SENT -> { /* Wait for the broadcast result */ }
        }
    },
    { /* Handle error */ },
)

// 4. Unregister it (e.g. in onDestroy)
context.unregisterReceiver(healthConnectBroadcastReceiver)
```

### Permissions denied

If the user cancels or navigates away from the permissions screen, Health Connect treats it as a denial.
**After two denials, Health Connect blocks your app** and ignores further permission requests — the only
recovery is opening the Health Connect app so the user can grant permissions manually.

Provide an "Open Health Connect" button backed by `openHealthConnectSettings`:

```kotlin
rookPermissionsManager.openHealthConnectSettings().fold(
    { /* Health Connect was opened */ },
    { /* Error opening Health Connect */ },
)
```

### Background read permissions

Full background reads require two things: the user must grant `READ_HEALTH_DATA_IN_BACKGROUND`, **and** the
device's Health Connect app version must support background reads. Check both with
`checkBackgroundReadStatus`:

```kotlin
rookPermissionsManager.checkBackgroundReadStatus().fold(
    {
        when (it) {
            BackgroundReadStatus.UNAVAILABLE -> {
                // Not available on this device — ask the user to update Health Connect
            }
            BackgroundReadStatus.PERMISSION_NOT_GRANTED -> {
                // Request background read permission
            }
            BackgroundReadStatus.PERMISSION_GRANTED -> {
                // Ready for background reads
            }
        }
    },
    { /* Handle error */ },
)
```

`requestHealthConnectPermissions` also requests background read (when the device supports it); the result
comes back in the `EXTRA_HEALTH_CONNECT_BACKGROUND_PERMISSION_GRANTED` extra shown in
[Request permissions](#request-permissions).

### Customizing permissions

To **reduce** the Health Connect permissions the SDK uses, remove them via the
[manifest merge](https://developer.android.com/build/manage-manifests#merge_rule_markers) `tools:node="remove"`
marker. The check/request functions then adapt automatically. For example, to drop `READ_MENSTRUATION`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          xmlns:tools="http://schemas.android.com/tools">
    <uses-permission
        android:name="android.permission.health.READ_MENSTRUATION"
        tools:node="remove" />
</manifest>
```

Notes:

- This only **removes** permissions. Adding one that isn't in the default set does nothing.
- Health Connect permissions use the `android.permission.health.READ_*` syntax; check the **merged
  manifest** tab to see the effective set.

### Revoke permissions

Reset all granted Health Connect permissions with `revokeHealthConnectPermissions`:

```kotlin
rookPermissionsManager.revokeHealthConnectPermissions().fold(
    { /* Success */ },
    { /* Handle error */ },
)
```

> The permissions may still appear granted until the app process is next stopped, at which point they are
> revoked.

## Android permissions

Standard (non-health) Android permissions used to track steps:
`POST_NOTIFICATIONS`, `ACTIVITY_RECOGNITION`, `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_HEALTH`.

Check them:

```kotlin
val hasAndroidPermissions = rookPermissionsManager.checkAndroidPermissions()
```

Request them — like Health Connect, this returns `ALREADY_GRANTED` / `REQUEST_SENT`, with the result
delivered to a receiver registered for `RookPermissionsManager.ACTION_ANDROID_PERMISSIONS`. Extras:

- `EXTRA_ANDROID_PERMISSIONS_GRANTED` — all Android permissions were granted.
- `EXTRA_ANDROID_PERMISSIONS_DIALOG_DISPLAYED` — whether the dialog was shown.

```kotlin
// 1. Create the broadcast receiver
private val androidBroadcastReceiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        val permissionsGranted = intent?.getBooleanExtra(
            RookPermissionsManager.EXTRA_ANDROID_PERMISSIONS_GRANTED, false,
        ) ?: false

        // Update your UI
    }
}

// 2. Register it (e.g. in onCreate)
ContextCompat.registerReceiver(
    context,
    androidBroadcastReceiver,
    IntentFilter(RookPermissionsManager.ACTION_ANDROID_PERMISSIONS),
    ContextCompat.RECEIVER_EXPORTED,
)

// 3. Request permissions
val requestPermissionsStatus = rookPermissionsManager.requestAndroidPermissions()
when (requestPermissionsStatus) {
    RequestPermissionsStatus.ALREADY_GRANTED -> { /* Already granted — update UI */ }
    RequestPermissionsStatus.REQUEST_SENT -> { /* Wait for the broadcast result */ }
}

// 4. Unregister it (e.g. in onDestroy)
context.unregisterReceiver(androidBroadcastReceiver)
```

### Android permissions denied

After a denial, Android won't show the dialog again. Use `shouldRequestAndroidPermissions(activity)` to
decide whether to request or to send the user to your app's settings:

```kotlin
val shouldRequest = RookPermissionsManager.shouldRequestAndroidPermissions(activity)

if (shouldRequest) {
    // Request permissions
} else {
    // Open your app's settings and tell the user to enable permissions manually
}
```

> You can also call `shouldRequestAndroidPermissions` **before** `requestAndroidPermissions` to anticipate
> whether the dialog will appear.

## Optimization permissions (optional)

Power-saving mode and OEM-specific energy restrictions can delay or stop the **Steps Counter** and
**Background Sync** services. These permissions are **optional** — the features work without them — but if
users report interruptions, expose a button (e.g. on a support screen) to grant them.

### Exact alarms

The SDK uses `SCHEDULE_EXACT_ALARM` (already in its manifest) to keep background services alive under
battery constraints.

```kotlin
val hasAlarmPermissions = rookPermissionsManager.checkExactAlarmPermissions()

val requestPermissionsStatus = rookPermissionsManager.requestExactAlarmPermissions()
when (requestPermissionsStatus) {
    RequestPermissionsStatus.ALREADY_GRANTED -> { /* Granted */ }
    RequestPermissionsStatus.REQUEST_SENT -> {
        // Special app-access permission: no callback — re-check with checkExactAlarmPermissions
    }
}
```

> On API 26–30 exact alarms are unrestricted, so these calls always return `true` /
> `RequestPermissionsStatus.REQUEST_SENT`.

### Battery optimizations

```kotlin
val batteryOptimizationsDisabled = rookPermissionsManager.checkBatteryOptimizationsDisabled()

val requestPermissionsStatus = rookPermissionsManager.requestDisableBatteryOptimizations()
when (requestPermissionsStatus) {
    RequestPermissionsStatus.ALREADY_GRANTED -> { /* Already disabled */ }
    RequestPermissionsStatus.REQUEST_SENT -> {
        // Special app-access permission: no callback — re-check with checkBatteryOptimizationsDisabled
    }
}
```

### Auto start (OEM-specific)

Brands like OPPO, OnePlus, and Xiaomi enforce a custom Auto Start restriction. Detect whether the device's
OEM has such a settings screen with `requiresOemAutoStartSetup`, then open it with `openOemAutoStartSetup`:

```kotlin
val hasAutoStartRestriction = rookPermissionsManager.requiresOemAutoStartSetup()

rookPermissionsManager.openOemAutoStartSetup().fold(
    { requestPermissionsStatus ->
        when (requestPermissionsStatus) {
            RequestPermissionsStatus.ALREADY_GRANTED -> { /* Device has no OEM screen */ }
            RequestPermissionsStatus.REQUEST_SENT -> {
                // OEM-specific screen: the user's choice can't be read back
            }
        }
    },
    { /* Handle error */ },
)
```

> A `true` from `requiresOemAutoStartSetup` means the OEM management system **exists**, not that auto start
> is currently blocked for your app. Use it to decide whether to prompt the user.
