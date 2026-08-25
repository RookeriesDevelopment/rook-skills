# Availability & permissions — Flutter · Apple Health

Prerequisite: the SDK is initialized and a `user_id` is registered (see `references/setup-and-init.md`).

## Availability

Apple Health (HealthKit) is **built into iOS** — there is no companion app to install, update, or
enable, so the SDK has **no availability-check method** (unlike the Android data sources). The only
availability concerns are platform ones, already covered in `references/setup-and-init.md`:

- The device must be running **iOS** — gate every call behind `Platform.isIOS`.
- **HealthKit is not available on the iOS Simulator** — test on a physical device.

## Request permissions

Call `requestPermissions` to launch the Apple Health permission dialog. With no arguments it requests
**all** the data types the SDK supports:

```dart
void requestPermissions() async {
  try {
    await AHRookHealthPermissionsManager.requestPermissions();

    // Success — the permission dialog was presented
  } catch (error) {
    // Handle error
  }
}
```

## Customizing permissions

`requestPermissions` accepts a list of `AppleHealthPermission` so you only ask for the data types your
app actually needs. There is one enum value per data type (e.g. `activeEnergyBurned`,
`basalEnergyBurned`, `stepCount`, …):

```dart
void requestCaloriesAndStepsPermissions() async {
  final caloriesAndStepsPermissions = [
    AppleHealthPermission.activeEnergyBurned,
    AppleHealthPermission.basalEnergyBurned,
    AppleHealthPermission.stepCount,
  ];

  try {
    await AHRookHealthPermissionsManager.requestPermissions(caloriesAndStepsPermissions);

    // Success
  } catch (error) {
    // Handle error
  }
}
```

### Recommended baseline

No single data type is required for every integration — request the minimum your product needs. A good
starting set for ROOK's common summaries and events:

```dart
final baselinePermissions = [
  AppleHealthPermission.stepCount,
  AppleHealthPermission.height,
  AppleHealthPermission.bodyMass,
  AppleHealthPermission.heartRate,
  AppleHealthPermission.heartRateVariabilitySDNN,
  AppleHealthPermission.workout,
  AppleHealthPermission.sleepAnalysis,
  AppleHealthPermission.oxygenSaturation,
];
```

| Permission(s) | Enables |
|---|---|
| `stepCount` | Step totals and physical-activity data |
| `height`, `bodyMass` | Body measurements and body summaries |
| `heartRate`, `heartRateVariabilitySDNN` | Heart-rate events and cardiovascular metrics in summaries |
| `workout` | Workout and activity events |
| `sleepAnalysis` | Sleep sessions and sleep summaries |
| `oxygenSaturation` | Oxygenation events and oxygen metrics in summaries |

Add other `AppleHealthPermission` values only when a feature needs them — e.g. energy and distance
(`activeEnergyBurned`, `basalEnergyBurned`, `distanceWalkingRunning`), temperature, `respiratoryRate`,
blood pressure (`bloodPressureSystolic` / `bloodPressureDiastolic`), `bloodGlucose`, `electrocardiogram`,
body composition, or dietary types.

## Notes

- **HealthKit hides read-permission status by design.** For privacy, iOS does **not** tell an app
  whether the user granted *read* access to a data type — so there is no reliable "are permissions
  granted?" check. Request permissions, then attempt a sync; if no data comes back, treat it as
  "not granted / no data" rather than an error. See `references/troubleshooting.md`.
- **The dialog only appears once per data type.** iOS shows the permission prompt the first time you
  request a given type. If the user needs to change a decision later, they must do it in the iOS
  **Settings → Health → Data Access & Devices** screen — your app cannot re-prompt for a type it
  already asked about.
- **Ask in context.** Request permissions right before the feature that needs the data, after
  explaining why, rather than on first launch — this follows Apple's guidance and improves grant rates.
