# Setup & initialization — Flutter · Health Connect

This covers installing the SDK, preparing the Android project, initializing the SDK, and registering the
user (`updateUserID`).

## About Health Connect

Health Connect is a **container**: apps write data into it and other apps read from it. For a clean sync,
both the provider (writer) and consumer (reader) must be working correctly. Health Connect has **no
real-time behaviour** — when data appears depends on each provider's own sync rules — and the platform is
still in an alpha state, so results are not always as expected.

> **Android only.** Health Connect is an Android platform; this SDK does nothing on iOS. Guard
> Android-specific calls with `Platform.isAndroid` in shared Flutter code.

> **Samsung devices:** for Samsung Health data, ROOK recommends the direct Samsung Health SDK integration
> instead of Health Connect — it is more accurate and avoids the Health Connect platform limitations.

## Requirements

- **Dart** `>=3.10.4 <4.0.0`, **Flutter** `>=3.0.0`.
- **Android Studio** Narwhal 4 Feature Drop | 2025.1.4 or higher is recommended.
- **`minSdk` 26**, **`targetSdk` 36** (set in the app module `build.gradle`).
- **ROOK SDK versions** — this skill targets the **V4** line: `rook_sdk_health_connect` **4.1.0** and its
  required dependency `rook_sdk_core` **4.1.1**. Keep in sync with the official docs; never invent a version.

## Install the dependencies

`rook_sdk_health_connect` requires `rook_sdk_core`. Add both from your project root:

```text
flutter pub add rook_sdk_core
flutter pub add rook_sdk_health_connect
```

In the app module `build.gradle`, set the SDK levels:

```groovy
android {
    defaultConfig {
        minSdk 26
        targetSdk 36
    }
}
```

## Android configuration

### Permissions

`rook_sdk_health_connect` **already declares** the permissions it needs in its own manifest — you do
**not** need to re-declare them. They are merged into your app automatically and include the Health
Connect read permissions (steps, sleep, heart rate, calories, distance, weight, blood glucose, etc.) plus
`ACTIVITY_RECOGNITION`, `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_HEALTH`, `POST_NOTIFICATIONS`, and
`RECEIVE_BOOT_COMPLETED`.

> Google may require an explanation (and sometimes a video) for the `FOREGROUND_SERVICE` /
> `FOREGROUND_SERVICE_HEALTH` permissions, which power the Background Steps feature (tracking steps and
> uploading them to ROOK servers — see the optional `references/background-steps.md`). Ask users to opt in
> before enabling that feature.

### Privacy policy intent filter

Health Connect requires a privacy policy screen. Add an intent filter to your `MainActivity` so Health
Connect can show your rationale when the user taps the privacy policy link:

```xml
<application>
    <activity android:name=".MainActivity">
        <!-- Android 13 and below: show the Health Connect permissions rationale. -->
        <intent-filter>
            <action android:name="androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE" />
        </intent-filter>
    </activity>

    <!-- Android 14 and above: an activity-alias for the same rationale. -->
    <activity-alias
        android:name="ViewPermissionUsageActivity"
        android:exported="true"
        android:permission="android.permission.START_VIEW_PERMISSION_USAGE"
        android:targetActivity=".MainActivity">
        <intent-filter>
            <action android:name="android.intent.action.VIEW_PERMISSION_USAGE" />
            <category android:name="android.intent.category.HEALTH_PERMISSIONS" />
        </intent-filter>
    </activity-alias>
</application>
```

To react when your app is launched from this intent, implement your own intent handling or use a package
like [`receive_intent`](https://pub.dev/packages/receive_intent).

### Secure storage (optional)

The SDK stores sensitive data (e.g. the current logged-in user) in password-protected storage. You may
optionally set a file name and password via manifest `meta-data`. If you omit these, the SDK uses a
default file name and auto-generates a new password on each install / data clear.

```xml
<application>
    <!-- BOTH tags are required if you use this. If you lose the password the SDK can no longer
         function; changing the file name recovers it but loses all previously stored data. -->
    <meta-data
        android:name="io.tryrook.security.storage.FILE_NAME"
        android:value="secure_storage" />
    <meta-data
        android:name="io.tryrook.security.storage.PASSWORD"
        android:value="SECURE_STORAGE_PASSWORD" />
</application>
```

### Obfuscation (if using R8/ProGuard)

In the app folder create a `proguard-rules.pro` file with:

```text
-keep class * extends com.google.protobuf.GeneratedMessageLite { *; }
```

Reference it from the app module `build.gradle`:

```groovy
buildTypes {
    release {
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
}
```

Then disable R8 full mode in `gradle.properties` (project level):

```properties
android.enableR8.fullMode=false
```

Or, to keep full mode enabled, add these rules to `proguard-rules.pro`:

```text
# Keep generic signature of Call, Response (R8 full mode strips signatures from non-kept items).
-keep,allowobfuscation,allowshrinking interface retrofit2.Call
-keep,allowobfuscation,allowshrinking class retrofit2.Response

# Suspend functions are wrapped in continuations whose type argument is used.
-keep,allowobfuscation,allowshrinking class kotlin.coroutines.Continuation
# Protobuf
-keep class * extends com.google.protobuf.GeneratedMessageLite { *; }
```

### Request data access (before release)

During development, Health Connect data access is unrestricted. Before publishing to the Play Store you
must complete the Play Console **Health apps declaration** and describe how your app uses each data type.
See Google's guide: <https://developer.android.com/health-and-fitness/guides/health-connect/publish/request-access>

> If you don't want access to a particular Health Connect data type, you can remove it — see the
> permissions reference (`references/permissions.md`).

## Configure & initialize the SDK

The configuration and initialization API is exposed through the static `HCRookConfigurationManager`.

Rules:

- Use placeholders `CLIENT_UUID` and `SECRET` in every snippet — **never** real values.
- `environment` is `RookEnvironment.sandbox` or `RookEnvironment.production`; each environment has its
  **own** `CLIENT_UUID` / `SECRET` and its own package-name+secret pair registered in the ROOK Portal.
  Typical pattern: sandbox on debug builds (`kDebugMode`), production on release.
- **Register credentials first.** The app's `applicationId` (package name) and its secret must be
  registered in the ROOK Portal *before* `initRook()`, or initialization fails with
  `SDKNotAuthorizedException`.
- Call `enableNativeLogs()` **before** `setConfiguration()` if you want native logs.
- `initRook()` makes an HTTP request — call it **once per app launch**.

`RookConfiguration` parameters: `clientUUID`, `secret`, `environment`, and `enableBackgroundSync` (when
`true`, background sync starts as soon as the SDK is initialized — see `references/background.md`).

```dart
void initialize() async {
  const environment =
      kDebugMode ? RookEnvironment.sandbox : RookEnvironment.production;

  final rookConfiguration = RookConfiguration(
    clientUUID: CLIENT_UUID,
    secret: SECRET,
    environment: environment,
    // If true, background sync starts when the SDK is initialized.
    enableBackgroundSync: false,
  );

  // MUST be called before setConfiguration to enable native logs.
  if (kDebugMode) {
    HCRookConfigurationManager.enableNativeLogs();
  }

  HCRookConfigurationManager.setConfiguration(rookConfiguration);

  try {
    await HCRookConfigurationManager.initRook();

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
    await HCRookConfigurationManager.updateUserID(userID);

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
**not** need to call it again. After initializing, use `getUserID` to check whether it was recovered from
storage:

```dart
void checkUserIDRegistered() async {
  final userID = await HCRookConfigurationManager.getUserID();

  if (userID != null) {
    // Recovered from storage — no need to call updateUserID again
  } else {
    // Not configured yet — you MUST call updateUserID
  }
}
```

### Logout

When the user logs out of your app, call `deleteUserFromRook` to remove the `userID` from the SDK and
disable it on the server (server disable applies to the Samsung Health data source only).

```dart
void logout() async {
  try {
    await HCRookConfigurationManager.deleteUserFromRook();

    // User removed from the SDK
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
    await HCRookConfigurationManager.syncUserTimeZone();

    // User timezone updated successfully
  } catch (error) {
    // Handle error
  }
}
```
