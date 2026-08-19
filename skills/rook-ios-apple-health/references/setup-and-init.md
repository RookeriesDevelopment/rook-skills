# Setup & initialization — iOS · Apple Health

<!-- FOLDED CORE: this file carries the shared auth/config/environment/user setup so the skill is self-contained.
     Do NOT link out to a separate "core" skill. -->

This covers installing the SDK, preparing the iOS project, initializing the SDK, and registering the
user (`updateUserId`).

## About Apple Health

Apple Health uses HealthKit as a **container**: apps and devices write data into it, and authorized apps
read from it. For a clean sync, both the provider (writer) and consumer (reader) must be working correctly.
Apple Health has **no guaranteed real-time behaviour** — when data appears depends on each provider's sync
rules, device availability, and iOS background scheduling. Because HealthKit data is encrypted while the
device is locked, background reads may also be delayed until the user unlocks the device.


## Requirements

- **iOS deployment target:** iOS 13.0 or later.
- **Development environment:** Xcode 26.0 or later.
- **Dependency manager:** Swift Package Manager, or CocoaPods 1.12.0 or later.
- **Device access:** HealthKit must be available on the device, and the user must authorize access to the
  health data types your app reads or writes.

## Project configuration

Before initializing the SDK, configure the app target to access HealthKit and declare why the app needs
to read or write health data.

### Enable the Xcode capabilities

1. Open the project or workspace in Xcode and select the app target.
2. Open **Signing & Capabilities**, click **+ Capability**, and add **HealthKit**.
3. In the HealthKit capability, enable **Background Delivery** if the app will synchronize data in the
   background.
4. If background synchronization is enabled, click **+ Capability** again, add **Background Modes**, and
   enable **Background fetch**.

### Add the HealthKit usage descriptions

Add the following privacy keys to the app target's `Info.plist`. Replace the example values with clear,
app-specific explanations that will be shown to the user when access is requested.

```xml
<key>NSHealthShareUsageDescription</key>
<string>Explain why your app needs to read the user's health and fitness data.</string>
<key>NSHealthUpdateUsageDescription</key>
<string>Explain why your app needs to write health data.</string>
```

`NSHealthShareUsageDescription` describes why the app reads HealthKit data, while
`NSHealthUpdateUsageDescription` describes why it writes data. These entries only declare the purpose;
the app must still request authorization for the required HealthKit data types at runtime.

## Install the dependency

ROOK supports installation through Swift Package Manager or CocoaPods. For new integrations, use the
newest available ROOK SDK release.

### Swift Package Manager

1. In Xcode, select **File > Add Package Dependencies**.
2. Enter the ROOK SDK repository URL:

```text
https://github.com/RookeriesDevelopment/rook-ios-sdk.git
```

3. Select the newest available version and add the package to your app target. Xcode will resolve and
   download the dependency automatically.

### CocoaPods

1. If the project does not already have a `Podfile`, create one from the project's root directory:

```bash
pod init
```

2. Add `RookSDK` to the app target in the `Podfile`:

```ruby
pod "RookSDK"
```

3. Install or update the pod repository and dependencies:

```bash
pod install --repo-update
```

4. Close the Xcode project and continue development from the generated `.xcworkspace` file.

## Configure & initialize the SDK

Import `RookSDK`, select the environment, provide the credentials for that environment, and initialize
the shared `RookConnectConfigurationManager`. Do this once during app launch, typically from
`application(_:didFinishLaunchingWithOptions:)` in the app delegate.

Use `.sandbox` for development and testing and `.production` for release builds. Each environment has
its own `CLIENT_UUID` and `SECRET`; supply them through your app's secure configuration and do not commit
real credentials to source control.

```swift
import RookSDK
import UIKit

final class AppDelegate: NSObject, UIApplicationDelegate {
  func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
  ) -> Bool {
    let configurationManager = RookConnectConfigurationManager.shared

    #if DEBUG
    configurationManager.setEnvironment(.sandbox)
    #else
    configurationManager.setEnvironment(.production)
    #endif

    configurationManager.setConfiguration(
      clientUUID: "CLIENT_UUID",
      secret: "SECRET",
      bundleId: nil,
      enableBackgroundSync: false,
      enableEventsBackgroundSync: false
    )

    configurationManager.initRook { result in
      switch result {
      case .success:
        debugPrint("ROOK SDK initialized")
      case .failure(let error):
        debugPrint("ROOK SDK initialization failed: \(error.localizedDescription)")
      }
    }

    RookBackGroundSync.shared.setBackListeners()
    return true
  }
}
```

Set `bundleId` to a custom bundle identifier only when the ROOK configuration requires one; otherwise,
pass `nil` to use the application's bundle identifier. Set `enableBackgroundSync` and
`enableEventsBackgroundSync` according to whether the app should synchronize summaries and events in
the background. If background synchronization is enabled, register the background listeners during app
launch as shown above.


## Register / update the user

After the user signs in to the app and the ROOK SDK has initialized successfully, register the user's
stable `user_id` with `updateUserId`. Use the identifier from your own user system—never a `customer_id`—
and wait for a successful result before starting manual or background synchronization.

```swift
func registerRookUser(_ userId: String) {
  RookConnectConfigurationManager.shared.updateUserId(userId) { result in
    switch result {
    case .success(let registered):
      debugPrint("ROOK user registered: \(registered)")
      // After registration succeeds, request HealthKit permissions and start synchronization.
    case .failure(let error):
      debugPrint("ROOK user registration failed: \(error.localizedDescription)")
    }
  }
}
```

Calling `updateUserId` with a different value replaces the previously stored user ID and resets the
sync status. If background synchronization is enabled, the available Apple Health data will be
synchronized again for the new user the next time the app launches.

### Remove the user during logout

If logout should disconnect the current user from ROOK, call `removeUserFromRook` while the SDK is still
configured and the current user is available. On success, the SDK removes the user's authorization to
upload Apple Health data, deletes the locally stored user ID, and disables synchronization.

```swift
func removeRookUserForLogout() {
  RookConnectConfigurationManager.shared.removeUserFromRook { result in
    switch result {
    case .success(let removed):
      debugPrint("ROOK user removed: \(removed)")
      // Complete the app's local logout flow after handling the result.
    case .failure(let error):
      debugPrint("ROOK user removal failed: \(error.localizedDescription)")
      // Apply the app's retry or error-handling policy before discarding the ROOK user context.
    }
  }
}
```

In SDK 4.x, use `removeUserFromRook`; the former `clearUser` method has been removed.
