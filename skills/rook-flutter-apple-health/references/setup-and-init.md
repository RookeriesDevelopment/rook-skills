# Setup & initialization — Flutter · Apple Health

This covers installing the packages, preparing the iOS project (HealthKit), configuring environments
and logging, initializing the SDK, and registering the user (`updateUserID`). It is self-contained —
everything the integration needs is here.

## About Apple Health

The ROOK Apple Health SDK reads data directly from **Apple Health (HealthKit)** on the device. Because
of that:

- It is **iOS only**. HealthKit is not available on Android — gate every call behind `Platform.isIOS`
  so a Flutter app that also ships Android doesn't crash on the missing plugin.
- **HealthKit is not available on the iOS Simulator** — test on a physical device.
- Two packages work together: **`rook_sdk_apple_health`** (the data source) and **`rook_sdk_core`**
  (shared types such as `RookConfiguration` and `RookEnvironment`).

## Requirements

- **Xcode 26.0 or higher.**
- **Dart** `>=3.10.4 <4.0.0` and **Flutter** `>=3.0.0`.
- **iOS deployment target 13.0** (set in the `Podfile`, below).
- **ROOK SDK versions** — this skill targets the **V4** line:
  - `rook_sdk_apple_health` → **4.1.0**
  - `rook_sdk_core` → **4.1.1**

  Keep these in sync with the official docs; never invent a version.

## Install the packages

From the project root:

```bash
flutter pub add rook_sdk_core
flutter pub add rook_sdk_apple_health
flutter pub get
```

## iOS configuration

1. In `ios/Podfile`, set the platform:

   ```ruby
   platform :ios, '13.0'
   ```

2. From the `ios` folder, install the pods:

   ```bash
   pod install
   ```

3. Open the `ios` folder with Xcode and select your project file (usually **Runner**).

4. **Add the HealthKit capability** — in the **Signing & Capabilities** tab, click **+ Capability**
   and add **HealthKit**. Then check the **Background delivery** option.

5. **Add Background Modes** — click **+ Capability** again, add **Background Modes**, and check
   **Background fetch**. (Required for automatic background sync — see `references/background.md`.)

6. **Link the HealthKit framework** — in the **Build Phases** tab, under **Link Binary With
   Libraries**, click **+** and add **HealthKit.framework**.

7. **Declare privacy usage descriptions** — HealthKit requires usage-description strings in
   `ios/Runner/Info.plist`. These are shown to the user in the permission dialog, so write text that
   explains why your app needs the data:

   ```xml
   <key>NSHealthShareUsageDescription</key>
   <string>This app requires access to your health and fitness data in order to track your workouts and activity levels.</string>
   <key>NSHealthUpdateUsageDescription</key>
   <string>This app requires permission to write workout data to HealthKit.</string>
   ```

## Environments and logging

- **Environment** — `RookEnvironment.sandbox` or `RookEnvironment.production`. Each environment has its
  **own** `CLIENT_UUID` / `SECRET`, registered independently in the ROOK Portal. Use the app's
  `kDebugMode` flag to pick one automatically:

  ```dart
  const environment = kDebugMode ? RookEnvironment.sandbox : RookEnvironment.production;
  ```

  Use `sandbox` during development and `production` only for release builds published to the App Store.

- **Logging** — to see the native SDK logs, call `enableNativeLogs()` **before** `setConfiguration`.
  Guard it with `kDebugMode` so logs never ship in release builds.

## Configure & initialize the SDK

All Apple Health functions live on `AH`-prefixed managers. Configuration and initialization use
`AHRookConfigurationManager`.

Rules:

- Use placeholders `CLIENT_UUID` and `SECRET` in every snippet — **never** real values.
- **Register credentials first.** The app's package name and its secret must be registered in the ROOK
  Portal *before* `initRook`, or initialization fails with `SDKNotAuthorizedException`. Each
  environment (Sandbox / Production) needs its own package-name + secret pair.
- `enableNativeLogs()` must run **before** `setConfiguration`.
- `initRook` makes an HTTP request — **initialize only once per app launch**.

`RookConfiguration` parameters: `clientUUID`, `secret`, `environment`, `enableBackgroundSync` (if
`true`, background sync starts when `initRook` completes — see `references/background.md`), and the
optional `appId`. Leave `appId` unset to use the app's own bundle identifier; set it only to override
that (otherwise the SDK reads it from `Bundle.main.bundleIdentifier`).

```dart
void initialize() async {
  const environment = kDebugMode ? RookEnvironment.sandbox : RookEnvironment.production;

  final rookConfiguration = RookConfiguration(
    clientUUID: CLIENT_UUID,
    secret: SECRET,
    environment: environment,
    // If true, background sync starts when the SDK is initialized.
    enableBackgroundSync: false,
  );

  // MUST be called before setConfiguration if you want native logs.
  if (kDebugMode) {
    AHRookConfigurationManager.enableNativeLogs();
  }

  AHRookConfigurationManager.setConfiguration(rookConfiguration);

  try {
    await AHRookConfigurationManager.initRook();

    // Success — SDK is initialized
  } catch (error) {
    // Handle error (e.g. SDKNotAuthorizedException if credentials aren't registered)
  }
}
```

> **Tip:** ask your users whether they want automatic sync and steps tracking, save that preference in
> local storage, and set `enableBackgroundSync` conditionally.

## Register / update the user

After the SDK is initialized, register the app end-user whose data will be synced. The identifier is
`user_id` (**never** `customer_id`). Call `updateUserID` as part of your **login / initialization**
flow.

```dart
void updateUserID() async {
  try {
    await AHRookConfigurationManager.updateUserID(userID);

    // Success
  } catch (error) {
    // Handle error
  }
}
```

> Calling `updateUserID` with a **different** `userID` overrides the previous one and **resets the sync
> status** — if you use automatic sync, all health data will synchronize again.

### Don't re-register on every launch

Once `updateUserID` has succeeded, the `userID` is persisted. If the app is closed and reopened you do
**not** need to call it again. After initializing, use `getUserID` to check whether it was recovered
from preferences:

```dart
void checkUserIDRegistered() async {
  final userID = await AHRookConfigurationManager.getUserID();

  if (userID != null) {
    // Recovered from preferences — no need to call updateUserID again
  } else {
    // Not configured yet — you MUST call updateUserID
  }
}
```

### Logout

When the user logs out of your app, call `deleteUserFromRook` to remove the `userID` from the SDK and
disable it on the server (this only disables the Apple Health data source).

```dart
void logout() async {
  try {
    await AHRookConfigurationManager.deleteUserFromRook();

    // User removed from the SDK and disabled on the server
  } catch (error) {
    // Handle error
  }
}
```

### User timezone

Every successful `updateUserID` also updates the user's timezone. This is enough in most cases. To
update the timezone manually, call `syncUserTimeZone`:

```dart
void updateTimeZoneInformation() async {
  try {
    await AHRookConfigurationManager.syncUserTimeZone();

    // Success
  } catch (error) {
    // Handle error
  }
}
```
