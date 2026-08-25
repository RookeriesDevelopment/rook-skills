# Availability & permissions — Flutter · Samsung Health

Prerequisite: the SDK is initialized and a `user_id` is registered (see `references/setup-and-init.md`).
All calls are **static** on `RookSamsung` and return a `Future`. Samsung Health exists only on Android —
guard these calls with `Platform.isAndroid`.

## Recommended flow

1. **Build the permission set** — the `List<SamsungHealthPermission>` your app actually needs.
2. **Check availability** — is the Samsung Health app installed, up to date, enabled, and onboarded?
3. **Check permissions** — does the app already have what it needs? If so, skip the request.
4. **Request permissions** — request the Samsung Health permissions; listen for the result via the
   `requestSamsungHealthPermissionsUpdates` stream.
5. **(Optional) Optimization permissions** — exact alarms, battery, and OEM auto-start, to keep
   **Background Sync** reliable on battery-constrained devices.

## Check availability

Before requesting permissions, confirm the Samsung Health app is installed and ready. Call
`checkSamsungHealthAvailability`, which returns a `SamsungHealthAvailability`:

| Status         | Meaning                                                            | What to do                                                     |
|----------------|--------------------------------------------------------------------|---------------------------------------------------------------|
| `installed`    | Samsung Health is installed and ready to be used.                  | Proceed to check/request permissions.                         |
| `notInstalled` | Samsung Health is not installed.                                   | Prompt the user to install Samsung Health.                    |
| `outdated`     | The installed Samsung Health version is too old.                   | Prompt the user to update Samsung Health.                     |
| `disabled`     | Samsung Health is disabled.                                        | Prompt the user to enable Samsung Health.                     |
| `notReady`     | The user hasn't completed onboarding (e.g. Samsung Health Terms).  | Prompt the user to open and finish Samsung Health onboarding. |

```dart
void checkAvailability() async {
  try {
    final availability = await RookSamsung.checkSamsungHealthAvailability();

    switch (availability) {
      case SamsungHealthAvailability.installed:
        // Proceed to check/request permissions
        break;
      case SamsungHealthAvailability.notInstalled:
        // Prompt the user to install Samsung Health
        break;
      case SamsungHealthAvailability.outdated:
        // Prompt the user to update Samsung Health
        break;
      case SamsungHealthAvailability.disabled:
        // Prompt the user to enable Samsung Health
        break;
      case SamsungHealthAvailability.notReady:
        // Prompt the user to finish Samsung Health onboarding
        break;
    }
  } catch (error) {
    // Handle error
  }
}
```

## Samsung Health permissions

Each Samsung Health data type is guarded by its own permission. Samsung Health permissions are **not**
declared in the manifest — you build a `List<SamsungHealthPermission>` in code and pass it to every
check/request call. The full set the SDK can use:

```dart
final samsungPermissions = [
  SamsungHealthPermission.activitySummary,
  SamsungHealthPermission.bloodGlucose,
  SamsungHealthPermission.bloodOxygen,
  SamsungHealthPermission.bloodPressure,
  SamsungHealthPermission.bodyComposition,
  SamsungHealthPermission.exercise,
  SamsungHealthPermission.exerciseLocation,
  SamsungHealthPermission.floorsClimbed,
  SamsungHealthPermission.heartRate,
  SamsungHealthPermission.nutrition,
  SamsungHealthPermission.sleep,
  SamsungHealthPermission.sleepApnea,
  SamsungHealthPermission.steps,
  SamsungHealthPermission.waterIntake,
  SamsungHealthPermission.bodyTemperature,
];
```

> **Keep the set consistent with your Samsung partner registration.** If you excluded a data type from the
> Samsung Partnership Request Form (see `references/setup-and-init.md`), **do not** include it here — passing
> a permission you didn't register will fail.

Reuse the **same** `samsungPermissions` list for the check and request calls below.

### Check permissions

Check whether **all** the requested permissions are granted with `checkSamsungHealthPermissions`:

```dart
void checkPermissions() async {
  try {
    final permissionsGranted = await RookSamsung.checkSamsungHealthPermissions(samsungPermissions);

    // permissionsGranted is a bool — update your UI
  } catch (error) {
    // Handle error
  }
}
```

Check whether **at least one** permission is granted with `checkSamsungHealthPermissionsPartially`:

```dart
void checkPermissionsPartially() async {
  try {
    final permissionsPartiallyGranted =
        await RookSamsung.checkSamsungHealthPermissionsPartially(samsungPermissions);

    // permissionsPartiallyGranted is a bool — update your UI
  } catch (error) {
    // Handle error
  }
}
```

### Request permissions

`requestSamsungHealthPermissions` returns a `RequestPermissionsStatus`:

- `alreadyGranted` — permissions were already granted; no request was sent.
- `requestSent` — the request dialog was shown; the result arrives asynchronously through the
  `requestSamsungHealthPermissionsUpdates` stream (which emits a `SamsungHealthPermissionsSummary`).

Because the outcome is delivered on a stream, subscribe **before** requesting, then cancel the
subscription when you no longer need it:

```dart
StreamSubscription<SamsungHealthPermissionsSummary>? streamSubscription;

void requestPermissions() async {
  // 1. Listen to the stream before requesting
  streamSubscription =
      RookSamsung.requestSamsungHealthPermissionsUpdates.listen((permissionsSummary) {
    // Permissions granted/denied — update your UI
  });

  // 2. Request permissions
  try {
    final requestPermissionsStatus =
        await RookSamsung.requestSamsungHealthPermissions(samsungPermissions);

    if (requestPermissionsStatus == RequestPermissionsStatus.alreadyGranted) {
      // Permissions already granted, update your UI
    } else {
      // requestSent — wait for the result on the stream
    }
  } catch (error) {
    // Handle error
  }
}

// 3. Stop listening when done (e.g. in dispose)
void dispose() {
  streamSubscription?.cancel();
}
```

### Customizing permissions

`requestSamsungHealthPermissions`, `checkSamsungHealthPermissions`, and
`checkSamsungHealthPermissionsPartially` all accept a `List<SamsungHealthPermission>`, so request only what
your app needs. For example, an activity-focused app:

```dart
final samsungPermissions = [
  SamsungHealthPermission.activitySummary,
  SamsungHealthPermission.bodyComposition,
  SamsungHealthPermission.exercise,
  SamsungHealthPermission.exerciseLocation,
  SamsungHealthPermission.heartRate,
  SamsungHealthPermission.sleep,
  SamsungHealthPermission.steps,
];
```

## Optimization permissions (optional)

Power-saving mode and OEM-specific energy restrictions can delay or stop **Background Sync**. These
permissions are **optional** — Background Sync works without them — but if users report interruptions,
expose a button (e.g. on a support screen) to grant them.

Requests for these special app-access permissions return a `RequestPermissionsStatus` but have **no result
callback** — after `requestSent`, re-run the matching `check...` call to read the final state.

### Exact alarms

Background Sync uses `SCHEDULE_EXACT_ALARM` (already in the SDK's manifest) to keep its background work
alive under battery constraints.

```dart
void checkExactAlarm() async {
  try {
    final hasAlarmPermissions = await RookSamsung.checkExactAlarmPermissions();

    // hasAlarmPermissions is a bool
  } catch (error) {
    // Handle error
  }
}

void requestExactAlarm() async {
  try {
    final requestPermissionsStatus = await RookSamsung.requestExactAlarmPermissions();

    switch (requestPermissionsStatus) {
      case RequestPermissionsStatus.alreadyGranted:
        // Permissions already granted
        break;
      case RequestPermissionsStatus.requestSent:
        // No callback — re-check with checkExactAlarmPermissions
        break;
    }
  } catch (error) {
    // Handle error
  }
}
```

> On API 26–30 exact alarms are unrestricted, so these calls always return `true` /
> `RequestPermissionsStatus.requestSent`.

### Battery optimizations

```dart
void checkBatteryOptimizations() async {
  try {
    final batteryOptimizationsDisabled = await RookSamsung.checkBatteryOptimizationsDisabled();

    // batteryOptimizationsDisabled is a bool
  } catch (error) {
    // Handle error
  }
}

void requestDisableBatteryOptimizations() async {
  try {
    final requestPermissionsStatus = await RookSamsung.requestDisableBatteryOptimizations();

    switch (requestPermissionsStatus) {
      case RequestPermissionsStatus.alreadyGranted:
        // Battery optimizations are already disabled
        break;
      case RequestPermissionsStatus.requestSent:
        // No callback — re-check with checkBatteryOptimizationsDisabled
        break;
    }
  } catch (error) {
    // Handle error
  }
}
```

### Auto start (OEM-specific)

Brands like OPPO, OnePlus, and Xiaomi enforce a custom Auto Start restriction that limits background
execution. Detect whether the device's OEM has such a settings screen with `requiresOemAutoStartSetup`,
then open it with `openOemAutoStartSetup`:

```dart
void checkAutoStart() async {
  try {
    final hasAutoStartRestriction = await RookSamsung.requiresOemAutoStartSetup();

    // hasAutoStartRestriction is a bool
  } catch (error) {
    // Handle error
  }
}

void openAutoStart() async {
  try {
    final requestPermissionsStatus = await RookSamsung.openOemAutoStartSetup();

    switch (requestPermissionsStatus) {
      case RequestPermissionsStatus.alreadyGranted:
        // The device has no OEM screen
        break;
      case RequestPermissionsStatus.requestSent:
        // Brand-specific screen — the user's choice can't be read back
        break;
    }
  } catch (error) {
    // Handle error
  }
}
```

> A `true` from `requiresOemAutoStartSetup` means the OEM management system **exists**, not that auto start
> is currently blocked for your app. Use it to decide whether to prompt the user.
