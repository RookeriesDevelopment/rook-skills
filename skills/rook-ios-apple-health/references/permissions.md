# Availability & permissions — iOS · Apple Health

## Contents

- [Check availability](#check-availability)
- [Check permission status](#check-permission-status)
- [Request permissions](#request-permissions)
- [Xcode declarations](#xcode-declarations)
- [Consent timing](#consent-timing)
- [Notes](#notes)

## Check availability

Check that HealthKit is available before presenting Apple Health controls or requesting permissions.

```swift
import RookSDK

let isAppleHealthAvailable =
  RookConnectConfigurationManager.shared.isHealthDataAvailable()

guard isAppleHealthAvailable else {
  // Hide or disable Apple Health features and explain that HealthKit is unavailable.
  return
}
```

When HealthKit is unavailable, do not request permissions or start a sync. Keep the rest of the app
usable when its core functionality does not depend on Apple Health.

## Check permission status

Use `RookConnectPermissionsManager.checkPermissionStatus` to inspect one `HealthDataType`. Passing `nil`
checks `.stepCount` by default.

```swift
let permissionsManager = RookConnectPermissionsManager()

permissionsManager.checkPermissionStatus(type: .stepCount) { status in
  switch status {
  case .permissionGranted:
    debugPrint("Step data is readable")
  case .permissionNotRequested:
    debugPrint("Step permission has not been requested")
  case .permissionIndeterminate:
    debugPrint("Step permission status cannot be determined")
  }
}
```

Apple does not expose whether the user denied read access. ROOK infers status by querying Apple Health:

- `.permissionGranted` means the SDK could read at least one sample for the requested type.
- `.permissionNotRequested` means the SDK determined that access has not been requested.
- `.permissionIndeterminate` can mean there is no data, access was denied, the device is locked, or the
  query could not determine a result.

Do not treat `.permissionIndeterminate` as a confirmed denial or `.permissionGranted` as a complete
authorization report for every requested type.

## Request permissions

Request permissions after the SDK is initialized and the user is registered. Pass only the data types
needed by the enabled features. Passing `nil` or an empty array requests every supported read permission.

```swift
let permissionsManager = RookConnectPermissionsManager()
let readPermissions: [HealthDataType] = [
  .stepCount,
  .height,
  .bodyMass,
  .heartRate,
  .heartRateVariabilitySDNN,
  .workout,
  .sleepAnalysis,
  .oxygenSaturation
]

permissionsManager.requestPermissions(readPermissions) { result in
  switch result {
  case .success(let presented):
    debugPrint("HealthKit authorization request completed: \(presented)")
  case .failure(let error):
    debugPrint("Could not request HealthKit permissions: \(error.localizedDescription)")
  }
}
```

The boolean result indicates whether the authorization request was presented successfully. It does
**not** indicate that the user granted every requested read permission.

### Choose the read permissions

No single data type is required for every integration. Request the minimum set needed for the product.
The ROOK documentation recommends the following baseline for its common summaries and events:

| Data type | Enables |
| --- | --- |
| `.stepCount` | Step totals and physical activity data |
| `.height`, `.bodyMass` | Body measurements and body summaries |
| `.heartRate`, `.heartRateVariabilitySDNN` | Heart-rate events and cardiovascular metrics in summaries |
| `.workout` | Workout and activity events |
| `.sleepAnalysis` | Sleep sessions and sleep summaries |
| `.oxygenSaturation` | Oxygenation events and oxygen metrics in summaries |

Add other `HealthDataType` values only when the feature needs them—for example energy and distance,
temperature and respiratory metrics, blood pressure, blood glucose, ECG, body composition, or dietary
data.

### Request dietary write permissions

Writing nutrition data requires a separate authorization request:

```swift
permissionsManager.requestDietaryWritePermissions { result in
  switch result {
  case .success(let presented):
    debugPrint("Dietary write authorization request completed: \(presented)")
  case .failure(let error):
    debugPrint("Could not request dietary write permission: \(error.localizedDescription)")
  }
}
```

Request these write permissions only if the app uses ROOK's nutrition-writing feature.

## Xcode declarations

The app target must include the **HealthKit** capability and these privacy keys in `Info.plist`:

```xml
<key>NSHealthShareUsageDescription</key>
<string>Explain why your app needs to read the user's health and fitness data.</string>
<key>NSHealthUpdateUsageDescription</key>
<string>Explain why your app needs to write health data.</string>
```

Use an app-specific purpose string for each key. `NSHealthShareUsageDescription` is required for reads;
`NSHealthUpdateUsageDescription` is required when requesting write access. For capability and background
mode steps, see [Project configuration](setup-and-init.md#project-configuration).

## Consent timing

Before designing the permission flow, ask the developer where Apple Health consent belongs in their
onboarding or feature journey. Default to a user-initiated action after a short explanation of the value
and requested data. Do not request every permission automatically at launch, and keep requests limited
to data used by visible product features.

## Notes

- Users may grant only some of the requested data types.
- A successful request callback does not prove that read access was granted.
- Missing data can mean the HealthKit store has no samples, the source has not synced yet, or access was
  denied; Apple does not let apps distinguish all of these cases.
- Users must change read-permission decisions in the Apple Health app.
- HealthKit data is encrypted while the device is locked, so checks and background reads can be delayed
  or return an indeterminate result until the device is unlocked.
- Register a `user_id` before starting manual or background synchronization.
