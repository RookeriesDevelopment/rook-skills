# Setup & initialization — Flutter · Samsung Health

This covers installing the SDK, preparing the Android project, initializing the SDK, and registering the
user (`updateUserID`). It is self-contained — everything the integration needs is here.

## About Samsung Health

The ROOK Samsung Health SDK reads data directly from the **Samsung Health app** on the device using
Samsung's Health Data SDK. Because of that, it has hard requirements:

- The **Samsung Health app v6.29 or later** must be installed.
- The device must run **Android 10 (API level 29) or above**. Samsung Health is available on all Samsung
  smartphones and also on non-Samsung Android smartphones.
- **Emulators are not supported** — you must test on a physical device.
- Samsung Health exists **only on Android**. In a cross-platform Flutter app, guard every `RookSamsung`
  call with a platform check (`Platform.isAndroid`).
- During development, you must enable Samsung Health
  [developer mode](https://developer.samsung.com/health/data/guide/developer-mode.html) on the test device.

## Requirements

- **Dart** `>=3.10.4 <4.0.0`, **Flutter** `>=3.0.0`.
- **Android Studio** Narwhal 4 Feature Drop | 2025.1.4 or higher is recommended.
- **`minSdk` 29**, **`targetSdk` 36** (set in the app module `build.gradle`).
- **ROOK packages** — this skill targets the **V4** line:
  - `rook_sdk_samsung_health` **4.1.0**
  - `rook_sdk_core` **4.1.1** (required dependency)

  Keep these in sync with the official docs; never invent a version.
- **Samsung Health Data SDK `.aar`** — `samsung-health-data-api-1.1.0.aar` (downloaded and added to the
  Android `libs/` directory; see below). This is Samsung's own artifact and has its **own** version
  (1.1.0), independent of the ROOK packages.

## Install the packages

Add both ROOK packages to your `pubspec.yaml`:

```text
flutter pub add rook_sdk_core
flutter pub add rook_sdk_samsung_health
```

## Android configuration

All native configuration lives under your Flutter project's `android/` directory.

### 1. Add the Samsung Health Data SDK `.aar`

Download the Samsung Health Data SDK `.aar` into the Android module's `libs/` directory, saved as
`samsung-health-data-api-1.1.0.aar` (the exact filename the gradle `files(...)` reference below expects).
`$rootDir` in a Flutter Android build resolves to the `android/` folder, so `$rootDir/libs/` is
`android/libs/`:

```bash
mkdir -p android/libs
curl -L -o android/libs/samsung-health-data-api-1.1.0.aar \
  https://docs.tryrook.io/assets/files/samsung-health-data-api-1.1.0-b358f7bf60e47cf2ca1c4aab4ae1df3b.aar
```

### 2. Configure the app module `build.gradle`

In `android/app/build.gradle`, set the SDK levels and add the `.aar` dependency:

```groovy
android {
    defaultConfig {
        minSdk 29
        targetSdk 36
    }
}

dependencies {
    implementation(files("$rootDir/libs/samsung-health-data-api-1.1.0.aar"))
}
```

> The `rook_sdk_samsung_health` plugin already bundles `gson`, so you do **not** need to add it yourself.

### 3. Obfuscation (if using R8/ProGuard)

In `android/app/proguard-rules.pro` add:

```text
-keep class * extends com.google.protobuf.GeneratedMessageLite { *; }
```

Reference it from the `release` build type in `android/app/build.gradle`:

```groovy
buildTypes {
    release {
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
}
```

Disable R8 full mode in `android/gradle.properties`:

```properties
android.enableR8.fullMode=false
```

Or, to keep full mode enabled, add these rules to `proguard-rules.pro` instead:

```text
# Keep generic signature of Call, Response (R8 full mode strips signatures from non-kept items).
-keep,allowobfuscation,allowshrinking interface retrofit2.Call
-keep,allowobfuscation,allowshrinking class retrofit2.Response

# Suspend functions are wrapped in continuations whose type argument is used.
-keep,allowobfuscation,allowshrinking class kotlin.coroutines.Continuation
```

### 4. Secure storage (optional)

The SDK stores sensitive data (e.g. the current logged-in user) in password-protected storage. You may
optionally set a file name (without extension) and password via manifest `meta-data` in
`android/app/src/main/AndroidManifest.xml`. If you omit these, the SDK uses a default file name and
auto-generates a new password on each install / data clear.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>
        <!-- BOTH tags are required if you use this. If you lose the password the SDK can no longer
             function; changing the file name recovers it but loses all previously stored data. -->
        <meta-data
            android:name="io.tryrook.samsung.security.storage.FILE_NAME"
            android:value="secure_storage" />
        <meta-data
            android:name="io.tryrook.samsung.security.storage.PASSWORD"
            android:value="SECURE_STORAGE_PASSWORD" />
    </application>
</manifest>
```

### 5. Request data access (before release)

During development, enabling Samsung Health developer mode is enough to read data. To access data in
**production**, you must submit Samsung's Partnership Request Form before distributing your app:
<https://developer.samsung.com/SHealth/partner/m20yrq4swlsx1chf>

Samsung registers your app's **package name** and **signature (SHA-256)** after approval. If your published
app's signature or package name differs from what you submitted, your app will not have access to Samsung's
data. In the form, select only the data types under the **"Read"** section.

The SDK can use the following data types — you only need to declare the ones you actually use:

`ACTIVITY_SUMMARY`, `BLOOD_GLUCOSE`, `BLOOD_OXYGEN`, `BLOOD_PRESSURE`, `BODY_COMPOSITION`, `EXERCISE`,
`EXERCISE_LOCATION`, `FLOORS_CLIMBED`, `HEART_RATE`, `NUTRITION`, `SLEEP`, `SLEEP_APNEA`, `STEPS`,
`WATER_INTAKE`, `BODY_TEMPERATURE`.

> If you exclude a data type from the list above, **do not** use it in the permission functions
> (check/request) — see `references/permissions.md`.

## Environment & logging

- `environment` is `RookEnvironment.sandbox` or `RookEnvironment.production` (from `rook_sdk_core`); each
  environment has its **own** `CLIENT_UUID` / `SECRET` and its own package-name+secret pair registered in
  the ROOK Portal. Gate it with `kDebugMode` — sandbox on debug builds, production on release:

  ```dart
  const environment = kDebugMode ? RookEnvironment.sandbox : RookEnvironment.production;
  ```

- To see native SDK logs, call `RookSamsung.enableNativeLogs()` **before** `initRook`. Guard it with
  `kDebugMode` so logs never ship in release builds.

## Configure & initialize the SDK

All SDK functions are **static** calls on `RookSamsung` — there is no instance to construct and no
`Context` to pass.

Rules:

- Use placeholders `CLIENT_UUID` and `SECRET` in every snippet — **never** real values.
- **Register credentials first.** The app's `applicationId` (package name) and its secret must be
  registered in the ROOK Portal *before* `initRook`, or initialization fails with
  `SDKNotAuthorizedException`.
- Call `enableNativeLogs()` **before** `initRook` if you want native logs.
- Initialize **once per app launch**.
- Setting `enableBackgroundSync: true` starts background sync as soon as the SDK is initialized. Leave it
  `false` and enable it explicitly later (see `references/background.md`); the recommended pattern is to
  ask the user's preference, store it locally, and set this flag conditionally.

`RookConfiguration` parameters: `clientUUID`, `secret`, `environment`, `enableBackgroundSync`.

Minimal initialization:

```dart
void initialize() async {
  final configuration = RookConfiguration(
    clientUUID: CLIENT_UUID,
    secret: SECRET,
    environment: kDebugMode ? RookEnvironment.sandbox : RookEnvironment.production,
    // If true, background sync starts when the SDK is initialized
    enableBackgroundSync: false,
  );

  // Enable logging only on debug builds (must run before initRook)
  if (kDebugMode) {
    RookSamsung.enableNativeLogs();
  }

  try {
    await RookSamsung.initRook(configuration);

    // Success — SDK is initialized
  } catch (error) {
    // Handle error (e.g. SDKNotAuthorizedException if credentials aren't registered)
  }
}
```

## Register / update the user

After the SDK is initialized, register the app end-user whose data will be synced. The identifier is
`user_id` (**never** `customer_id`). Call `updateUserID` as part of your **login / initialization** flow.

```dart
void updateUserID() async {
  try {
    await RookSamsung.updateUserID(userID);

    // Success
  } catch (error) {
    // Handle error
  }
}
```

> `updateUserID` trims its input (`" userID "` becomes `"userID"`); a value that is empty after trimming
> returns an `IllegalStateException`.

> Calling `updateUserID` with a **different** `userID` overrides the previous one and **resets the sync
> status** — if you use automatic sync, all health data will synchronize again.

### Don't re-register on every launch

Once `updateUserID` has succeeded, the `userID` is persisted. If the app is closed and reopened you do
**not** need to call it again. After initializing, use `getUserID` to check whether it was recovered from
storage:

```dart
void checkUserIDRegistered() async {
  final userID = await RookSamsung.getUserID();

  if (userID != null) {
    // Recovered from storage — no need to call updateUserID again
  } else {
    // Not configured yet — you MUST call updateUserID
  }
}
```

### Logout

When the user logs out of your app, call `deleteUserFromRook` to remove the `userID` from the SDK **and**
disable it on the server (this only disables the Samsung Health data source).

```dart
void deleteUser() async {
  try {
    await RookSamsung.deleteUserFromRook();

    // User removed from the SDK and disabled on the server
  } catch (error) {
    // Handle error
  }
}
```

### User timezone

Every successful `updateUserID` also updates the user's timezone (look for `Timezone information updated`
in the logs). This is enough in most cases. To update the timezone manually, call `syncUserTimeZone`:

```dart
void updateTimeZoneInformation() async {
  try {
    await RookSamsung.syncUserTimeZone();

    // User timezone updated successfully
  } catch (error) {
    // Handle error
  }
}
```
