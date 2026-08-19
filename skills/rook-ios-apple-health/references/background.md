# Sync health data (automatic) — iOS · Apple Health

ROOK supports two related automatic-sync modes:

- **Continuous (foreground) sync** checks for new and pending data when the user opens the app.
- **Background sync** listens for HealthKit updates and attempts to upload new summaries and events while
  the app is in the background.

`RookConnectConfigurationManager` controls continuous sync and can enable or disable all automatic sync at
once. `RookBackGroundSync` registers the HealthKit listeners and provides separate controls for summary and
event background sync.

## Contents

- [Before enabling automatic sync](#before-enabling-automatic-sync)
- [Register background listeners at launch](#register-background-listeners-at-launch)
- [Set the initial background configuration](#set-the-initial-background-configuration)
- [Enable all automatic sync](#enable-all-automatic-sync)
- [Control summary and event background sync separately](#control-summary-and-event-background-sync-separately)
- [Observe background-sync activity](#observe-background-sync-activity)
- [Disable sync](#disable-sync)
- [Constraints](#constraints)

## Before enabling automatic sync

Complete these steps first:

1. Enable the **HealthKit** capability and **Background Delivery** for the app target.
2. Add **Background Modes** and enable **Background fetch**.
3. Configure and initialize the ROOK SDK.
4. Register the user's stable `user_id` with `updateUserId`.
5. Request the HealthKit permissions required by the summaries and events the app will synchronize.
6. Obtain the user's consent before enabling automatic synchronization.

See [Setup and initialization](setup-and-init.md) and
[Availability and permissions](permissions.md) for the prerequisite flows.

## Register background listeners at launch

Call `setBackListeners()` once on every app launch from
`application(_:didFinishLaunchingWithOptions:)`. Registering the listeners prepares the SDK to receive
HealthKit background-delivery notifications; it does not replace user registration or the HealthKit
permission request.

```swift
import RookSDK
import UIKit

final class AppDelegate: NSObject, UIApplicationDelegate {
  func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
  ) -> Bool {
    RookBackGroundSync.shared.setBackListeners()
    return true
  }
}
```

Keep this call in the launch path even when the user enables sync later from a settings screen.

## Set the initial background configuration

The two background flags in `setConfiguration` select which categories should be enabled when the SDK is
configured:

```swift
RookConnectConfigurationManager.shared.setConfiguration(
  clientUUID: "CLIENT_UUID",
  secret: "SECRET",
  bundleId: nil,
  enableBackgroundSync: true,
  enableEventsBackgroundSync: true
)
```

| Parameter | Effect when `true` |
| --- | --- |
| `enableBackgroundSync` | Enables background synchronization for sleep, physical, and body summaries. |
| `enableEventsBackgroundSync` | Enables background synchronization for supported HealthKit events. |

Use the `RookBackGroundSync` enable and disable methods for runtime changes. In particular, passing `false`
to a configuration flag should not be used as a replacement for an explicit disable call.

## Enable all automatic sync

After the SDK is initialized, the user is registered, and HealthKit permissions have been requested, call
`enableSync()` to enable continuous foreground sync and both background categories:

```swift
let configurationManager = RookConnectConfigurationManager.shared
configurationManager.enableSync()
```

With this option, ROOK checks for new data and pending data from previous days when the app opens, while
`RookBackGroundSync` also listens for summary and event updates in the background.

Use `isSyncEnable()` to read the **foreground** sync status:

```swift
let isForegroundSyncEnabled = configurationManager.isSyncEnable()
```

`isSyncEnable()` does not report the individual summary or event background status. Use the
`RookBackGroundSync` status methods for those values.

## Control summary and event background sync separately

Use the granular controls when the user should be able to enable summaries and events independently, or
when the app needs background delivery without continuous foreground sync.

```swift
let backgroundSync = RookBackGroundSync.shared

backgroundSync.enableBackGroundForSummaries()
backgroundSync.enableBackGroundForEvents()

let areSummariesEnabled = backgroundSync.isBackGroundForSummariesEnable()
let areEventsEnabled = backgroundSync.isBackGroundForEventsEnable()
```

Disable each category with its matching method:

```swift
backgroundSync.disableBackGroundForSummaries()
backgroundSync.disableBackGroundForEvents()
```

`disableBackGroundForEvents()` does not take a completion handler.

## Observe background-sync activity

The optional callbacks on `RookBackGroundSync` can update diagnostics or app state. Assign them before
registering the background listeners, and use a weak capture when they reference an object owned by the
app.

```swift
let backgroundSync = RookBackGroundSync.shared

backgroundSync.handleSummariesBackProcess = { summaryType in
  debugPrint("Processing background summary: \(summaryType)")
}

backgroundSync.handleSummariesUploaded = {
  debugPrint("Background summaries uploaded")
}

backgroundSync.handleEventsUploaded = { eventType in
  debugPrint("Background event uploaded: \(eventType)")
}

backgroundSync.handleErrorObserverQuery = { error in
  debugPrint("Background observer failed: \(error.localizedDescription)")
}

backgroundSync.setBackListeners()
```

Treat these callbacks as activity signals, not as a guaranteed schedule. iOS decides when the app receives
background execution time.

## Disable sync

Call `disableSync()` to turn off continuous foreground sync and both summary and event background sync:

```swift
RookConnectConfigurationManager.shared.disableSync()
```

Use the granular `RookBackGroundSync` disable methods instead when only one background category should be
turned off. When removing a user during logout, `removeUserFromRook` disables automatic sync after the
user is removed successfully; see [Register / update the user](setup-and-init.md#register--update-the-user).

## Constraints

- Background delivery is not real time and does not run at a fixed interval; iOS controls when the app is
  awakened.
- HealthKit data is encrypted while the device is locked, so a background read may be delayed until the
  user unlocks the device.
- ROOK can upload only data that providers have already written to Apple Health and that the user has
  authorized the app to read.
- Network availability, Low Power Mode, force-quitting the app, and system resource limits can delay
  background work. Design the UI so it does not promise an exact upload time.
- Calling `updateUserId` with a different user resets sync status. Reconfirm the intended automatic-sync
  settings for the newly registered user.
- Use the [manual sync reference](sync.md) when the product needs an explicit, user-driven retry or a
  targeted date and data type.
