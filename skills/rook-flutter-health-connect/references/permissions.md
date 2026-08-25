# Availability & permissions — Flutter · Health Connect

Prerequisite: the SDK is initialized and a `user_id` is registered (see `references/setup-and-init.md`).
Everything here is **Android only** — guard calls with `Platform.isAndroid`.

The availability and permission API is exposed through the static `HCRookHealthPermissionsManager`.

## Recommended flow

1. **Check availability** — is Health Connect installed / supported on this device?
2. **Check permissions** — does the app already have what it needs? If so, skip the request.
3. **Request permissions** — request Health Connect (data) permissions; listen for the result via the
   updates stream.
4. **Handle denial** — if the user denied twice, guide them to open Health Connect settings.
5. **(Optional) Background read** — for background sync, additionally check/request the background-read
   permission.
6. **(Optional) Android + optimization permissions** — needed for background steps / background sync.

## Check availability

Call `checkHealthConnectAvailability` before anything else and act on the result:

| Status         | Meaning                                | What to do                                   |
|----------------|----------------------------------------|----------------------------------------------|
| `installed`    | Health Connect APK is installed        | Proceed to check/request permissions         |
| `notInstalled` | APK is not installed                   | Prompt the user to install it from the Store |
| `notSupported` | Device does not support Health Connect | Take the user out of the Health Connect flow |

```dart
void checkAvailability() {
  HCRookHealthPermissionsManager.checkHealthConnectAvailability().then((availability) {
    // React to HCAvailabilityStatus.installed / notInstalled / notSupported
  }).catchError((exception) {
    // Handle error
  });
}
```

The Health Connect Play Store listing is
`com.google.android.apps.healthdata` (`https://play.google.com/store/apps/details?id=com.google.android.apps.healthdata`).

## Health Connect permissions

These are the data-read permissions the SDK uses — one per Health Connect data type. They are the same set
declared automatically by `rook_sdk_health_connect` (see `references/setup-and-init.md`); you can narrow
this set under [Customizing permissions](#customizing-permissions).

`READ_SLEEP`, `READ_STEPS`, `READ_DISTANCE`, `READ_FLOORS_CLIMBED`, `READ_ELEVATION_GAINED`,
`READ_OXYGEN_SATURATION`, `READ_VO2_MAX`, `READ_TOTAL_CALORIES_BURNED`, `READ_ACTIVE_CALORIES_BURNED`,
`READ_HEART_RATE`, `READ_RESTING_HEART_RATE`, `READ_HEART_RATE_VARIABILITY`, `READ_EXERCISE`, `READ_SPEED`,
`READ_WEIGHT`, `READ_HEIGHT`, `READ_BLOOD_GLUCOSE`, `READ_BLOOD_PRESSURE`, `READ_HYDRATION`,
`READ_BODY_TEMPERATURE`, `READ_RESPIRATORY_RATE`, `READ_NUTRITION`, `READ_MENSTRUATION`, `READ_POWER`,
`READ_BASAL_BODY_TEMPERATURE`, `READ_BODY_FAT`, `READ_LEAN_BODY_MASS`, `READ_BODY_WATER_MASS`,
`READ_BONE_MASS`.

### Check permissions

Check whether **all** required permissions are granted:

```dart
void checkHealthConnectPermissions() {
  HCRookHealthPermissionsManager.checkHealthConnectPermissions().then((permissionsGranted) {
    // Update your UI
  }).catchError((exception) {
    // Handle error
  });
}
```

Check whether **at least one** permission is granted (`checkHealthConnectPermissionsPartially`):

```dart
void checkHealthConnectPermissionsPartially() {
  HCRookHealthPermissionsManager.checkHealthConnectPermissionsPartially().then((permissionsPartiallyGranted) {
    // Update your UI
  }).catchError((error) {
    // Handle error
  });
}
```

### Request permissions

`requestHealthConnectPermissions` returns a `RequestPermissionsStatus`:

- `alreadyGranted` — permissions were already granted; no request was sent.
- `requestSent` — the request dialog was shown; the result arrives on the
  `requestHealthConnectPermissionsUpdates` stream (emits a `HealthConnectPermissionsSummary`).

```dart
// 1. Create a stream subscription
StreamSubscription<HealthConnectPermissionsSummary>? streamSubscription;

// 2. Listen to the stream
streamSubscription =
    HCRookHealthPermissionsManager.requestHealthConnectPermissionsUpdates.listen((permissionsSummary) {
  // Update your UI
});

// 3. Request permissions
HCRookHealthPermissionsManager.requestHealthConnectPermissions().then((requestPermissionsStatus) {
  if (requestPermissionsStatus == RequestPermissionsStatus.alreadyGranted) {
    // Permissions already granted, update your UI
  } else {
    // Wait for the result on the stream
  }
}).catchError((error) {
  // Handle error
});

// 4. Stop listening when you no longer need updates (e.g. on dispose)
streamSubscription?.cancel();
```

### Permissions denied

If the user cancels or navigates away from the permissions screen, Health Connect treats it as a denial.
**After two denials, Health Connect blocks your app** and ignores further permission requests — the only
recovery is opening the Health Connect app so the user can grant permissions manually.

Provide an "Open Health Connect" button backed by `openHealthConnectSettings`:

```dart
void openHealthConnect() {
  HCRookHealthPermissionsManager.openHealthConnectSettings().then((_) {
    // Health Connect was opened
  }).catchError((exception) {
    // Handle error
  });
}
```

### Background read permissions

Full background reads require two things: the user must grant `READ_HEALTH_DATA_IN_BACKGROUND`, **and** the
device's Health Connect app version must support background reads. Check both with `checkBackgroundReadStatus`,
which returns an `HCBackgroundReadStatus`:

```dart
void checkBackgroundReadStatus() async {
  try {
    final backgroundReadStatus = await HCRookHealthPermissionsManager.checkBackgroundReadStatus();

    switch (backgroundReadStatus) {
      case HCBackgroundReadStatus.unavailable:
        // Not available on this device — ask the user to update Health Connect
        break;
      case HCBackgroundReadStatus.permissionNotGranted:
        // Request background read permission
        break;
      case HCBackgroundReadStatus.permissionGranted:
        // Ready for background reads
        break;
    }
  } catch (error) {
    // Handle error
  }
}
```

`requestHealthConnectPermissions` also requests background read when the device supports it (otherwise that
permission is left out of the request). The result arrives on the `requestHealthConnectPermissionsUpdates`
stream shown in [Request permissions](#request-permissions).

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

```dart
void revokeHealthConnectPermissions() {
  HCRookHealthPermissionsManager.revokeHealthConnectPermissions().then((_) {
    // Health Connect permissions revoked, restart the app to apply the changes
  }).catchError((error) {
    // Handle error
  });
}
```

> The permissions may still appear granted until the app process is next stopped, at which point they are
> revoked.

## Android permissions

Standard (non-health) Android permissions used to track steps:
`POST_NOTIFICATIONS`, `ACTIVITY_RECOGNITION`, `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_HEALTH`.

Check them:

```dart
void checkAndroidPermissions() {
  HCRookHealthPermissionsManager.checkAndroidPermissions().then((permissionsGranted) {
    // Update your UI
  }).catchError((exception) {
    // Handle error
  });
}
```

Request them — like Health Connect, this returns `RequestPermissionsStatus` (`alreadyGranted` /
`requestSent`), with the result delivered on the `requestAndroidPermissionsUpdates` stream (emits an
`AndroidPermissionsSummary`):

```dart
// 1. Create a stream subscription
StreamSubscription<AndroidPermissionsSummary>? streamSubscription;

// 2. Listen to the stream
streamSubscription =
    HCRookHealthPermissionsManager.requestAndroidPermissionsUpdates.listen((permissionsSummary) {
  // Update your UI
});

// 3. Request permissions
try {
  final requestPermissionsStatus = await HCRookHealthPermissionsManager.requestAndroidPermissions();

  if (requestPermissionsStatus == RequestPermissionsStatus.alreadyGranted) {
    // Permissions already granted, update your UI
  } else {
    // Wait for the result on the stream
  }
} catch (error) {
  // Handle error
}

// 4. Stop listening when you no longer need updates (e.g. on dispose)
streamSubscription?.cancel();
```

> Tip: call `shouldRequestAndroidPermissions()` **before** `requestAndroidPermissions` to know whether the
> dialog will be shown, and handle the denied case with more anticipation.

### Android permissions denied

After a denial, Android won't show the dialog again. Use `shouldRequestAndroidPermissions` to decide
whether to request or to send the user to your app's settings:

```dart
void requestAndroidPermissions() async {
  try {
    final shouldRequestPermissions =
        await HCRookHealthPermissionsManager.shouldRequestAndroidPermissions();

    if (shouldRequestPermissions) {
      // Request permissions
    } else {
      // Open your app's settings and tell the user to enable permissions manually
    }
  } catch (error) {
    // Handle error
  }
}
```

## Optimization permissions (optional)

Power-saving mode and OEM-specific energy restrictions can delay or stop the **Steps Counter** and
**Background Sync** services. These permissions are **optional** — the features work without them — but if
users report interruptions, expose a button (e.g. on a support screen) to grant them.

### Exact alarms

The SDK uses `SCHEDULE_EXACT_ALARM` (already in its manifest) to keep background services alive under
battery constraints.

```dart
HCRookHealthPermissionsManager.checkExactAlarmPermissions().then((hasAlarmPermissions) {
  // Success
}).catchError((error) {
  // Handle error
});

HCRookHealthPermissionsManager.requestExactAlarmPermissions().then((requestPermissionsStatus) {
  switch (requestPermissionsStatus) {
    case RequestPermissionsStatus.alreadyGranted:
      // Permissions already granted
    case RequestPermissionsStatus.requestSent:
      // Special app-access permission: no callback — re-check with checkExactAlarmPermissions
  }
}).catchError((error) {
  // Handle error
});
```

> On API 26–30 exact alarms are unrestricted, so these calls always return `true` /
> `RequestPermissionsStatus.requestSent`.

### Battery optimizations

```dart
HCRookHealthPermissionsManager.checkBatteryOptimizationsDisabled().then((batteryOptimizationsDisabled) {
  // Success
}).catchError((error) {
  // Handle error
});

HCRookHealthPermissionsManager.requestDisableBatteryOptimizations().then((requestPermissionsStatus) {
  switch (requestPermissionsStatus) {
    case RequestPermissionsStatus.alreadyGranted:
      // Battery optimizations are already disabled
    case RequestPermissionsStatus.requestSent:
      // Special app-access permission: no callback — re-check with checkBatteryOptimizationsDisabled
  }
}).catchError((error) {
  // Handle error
});
```

### Auto start (OEM-specific)

Brands like OPPO, OnePlus, and Xiaomi enforce a custom Auto Start restriction. Detect whether the device's
OEM has such a settings screen with `requiresOemAutoStartSetup`, then open it with `openOemAutoStartSetup`:

```dart
HCRookHealthPermissionsManager.requiresOemAutoStartSetup().then((hasAutoStartRestriction) {
  // Success
}).catchError((error) {
  // Handle error
});

HCRookHealthPermissionsManager.openOemAutoStartSetup().then((requestPermissionsStatus) {
  switch (requestPermissionsStatus) {
    case RequestPermissionsStatus.alreadyGranted:
      // The device has no OEM screen
    case RequestPermissionsStatus.requestSent:
      // OEM-specific screen: the user's choice can't be read back
  }
}).catchError((error) {
  // Handle error
});
```

> A `true` from `requiresOemAutoStartSetup` means the OEM management system **exists**, not that auto start
> is currently blocked for your app. Use it to decide whether to prompt the user.
