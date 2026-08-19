# Setup & initialization — Android · Health Connect

This covers installing the SDK, preparing the Android project, initializing the SDK, and registering the
user (`updateUserID`).

## About Health Connect

Health Connect is a **container**: apps write data into it and other apps read from it. For a clean sync,
both the provider (writer) and consumer (reader) must be working correctly. Health Connect has **no
real-time behaviour** — when data appears depends on each provider's own sync rules — and the platform is
still in an alpha state, so results are not always as expected.

> **Samsung devices:** for Samsung Health data, ROOK recommends the direct Samsung Health SDK integration
> instead of Health Connect — it is more accurate and avoids the Health Connect platform limitations.

## Requirements

- **Android Studio** Narwhal 4 Feature Drop | 2025.1.4 or higher is recommended.
- **`minSdk` 26**, **`targetSdk` 36** (set in the app module `build.gradle`).
- **ROOK SDK version** — this skill targets the **V4** line; use **4.1.0**. Keep in sync with the
  official docs; never invent a version.

## Install the dependency

In the app module `build.gradle`, set the SDK levels and add the dependency:

```groovy
android {
    defaultConfig {
        minSdk 26
        targetSdk 36
    }
}

dependencies {
    implementation("com.rookmotion.android:rook-sdk:4.1.0")
}
```

## Android configuration

### Permissions

The `rook-sdk` **already declares** the permissions it needs in its own manifest — you do **not** need to
re-declare them. They are merged into your app automatically and include Health Connect read permissions
(steps, sleep, heart rate, calories, etc.) plus `ACTIVITY_RECOGNITION`, `FOREGROUND_SERVICE`,
`FOREGROUND_SERVICE_HEALTH`, `POST_NOTIFICATIONS`, `RECEIVE_BOOT_COMPLETED`, and
`READ_HEALTH_DATA_IN_BACKGROUND`.

> Google may require an explanation (and sometimes a video) for the `FOREGROUND_SERVICE` /
> `FOREGROUND_SERVICE_HEALTH` permissions, which power the Background Steps feature (tracking steps and
> uploading them to ROOK servers). Ask users to opt in before enabling that feature.

### Privacy policy intent filter

Health Connect requires a privacy policy screen. Add an intent filter so Health Connect can show your
rationale when the user taps the privacy policy link. Use a dedicated activity (or a deeplink if you use a
single-activity architecture):

```xml
<application>
    <!-- Android 13 and below: an activity that shows the Health Connect permissions rationale. -->
    <activity
        android:name=".features.privacypolicy.HCPrivacyPolicyActivity"
        android:enabled="true"
        android:exported="true">
        <intent-filter>
            <action android:name="androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE" />
        </intent-filter>
    </activity>

    <!-- Android 14 and above: an activity-alias for the same rationale. -->
    <activity-alias
        android:name="ViewPermissionUsageActivity"
        android:exported="true"
        android:permission="android.permission.START_VIEW_PERMISSION_USAGE"
        android:targetActivity=".features.privacypolicy.HCPrivacyPolicyActivity">
        <intent-filter>
            <action android:name="android.intent.action.VIEW_PERMISSION_USAGE" />
            <category android:name="android.intent.category.HEALTH_PERMISSIONS" />
        </intent-filter>
    </activity-alias>
</application>
```

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

Disable R8 full mode in `gradle.properties`:

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
```

### Request data access (before release)

During development, Health Connect data access is unrestricted. Before publishing to the Play Store you
must complete the Play Console **Health apps declaration** and describe how your app uses each data type.
See Google's guide: <https://developer.android.com/health-and-fitness/guides/health-connect/publish/request-access>

> If you don't want access to a particular Health Connect data type, you can remove it — see the
> permissions reference (`references/permissions.md`).

### Completed AndroidManifest

Putting the pieces together, your `AndroidManifest.xml` should look like this (the SDK's own permissions
are merged automatically, so they do not appear here):

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application
        android:name=".RookApplication"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme">

        <activity
            android:name=".features.HomeActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- Android 13 and below: privacy policy rationale activity. -->
        <activity
            android:name=".features.privacypolicy.HCPrivacyPolicyActivity"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE" />
            </intent-filter>
        </activity>

        <!-- Android 14 and above: privacy policy rationale activity-alias. -->
        <activity-alias
            android:name="ViewPermissionUsageActivity"
            android:exported="true"
            android:permission="android.permission.START_VIEW_PERMISSION_USAGE"
            android:targetActivity=".features.privacypolicy.HCPrivacyPolicyActivity">
            <intent-filter>
                <action android:name="android.intent.action.VIEW_PERMISSION_USAGE" />
                <category android:name="android.intent.category.HEALTH_PERMISSIONS" />
            </intent-filter>
        </activity-alias>

        <!-- Optional secure storage tags go here (see "Secure storage" above). -->
    </application>
</manifest>
```

## Configure & initialize the SDK

Rules:

- Use placeholders `CLIENT_UUID` and `SECRET` in every snippet — **never** real values.
- `environment` is `RookEnvironment.SANDBOX` or `RookEnvironment.PRODUCTION`; each environment has its
  **own** `CLIENT_UUID` / `SECRET` and its own package-name+secret pair registered in the ROOK Portal.
  Typical pattern: SANDBOX on debug builds, PRODUCTION on release.
- **Register credentials first.** The app's `applicationId` (package name) and its secret must be
  registered in the ROOK Portal *before* `initRook()`, or initialization fails with
  `HCNotAuthorizedException`.
- Call `enableLocalLogs()` **before** `setConfiguration()` if you want native logs.
- Initialize **once per app launch** — keep `RookConfigurationManager` as a singleton (ServiceLocator or DI).

`RookConfiguration` parameters: `clientUUID`, `secret`, `environment`, and an optional `packageName`
(if omitted, the SDK reads it from `Context`).

Minimal initialization:

```kotlin
val rookConfigurationManager = RookConfigurationManager(context)

val environment = if (BuildConfig.DEBUG) RookEnvironment.SANDBOX else RookEnvironment.PRODUCTION
val configuration = RookConfiguration(CLIENT_UUID, SECRET, environment)

// Enable logging only on debug builds (must run before setConfiguration)
if (BuildConfig.DEBUG) {
    rookConfigurationManager.enableLocalLogs()
}

rookConfigurationManager.setConfiguration(configuration)
rookConfigurationManager.initRook().fold(
    {
        // Success — SDK is initialized
    },
    {
        // Handle error (e.g. HCNotAuthorizedException if credentials aren't registered)
    },
)
```

Keep `RookConfigurationManager` as a **single instance** for the app's lifetime. **If the app already uses
a DI framework (Hilt, Koin, Dagger, etc.), provide it as a singleton there** — e.g. an `@Provides
@Singleton` function in Hilt, or a `single { }` in a Koin module. The ServiceLocator below is only a
generic, dependency-free illustration of the same idea; prefer the app's existing DI when there is one.

Recommended singleton pattern via a ServiceLocator held by your `Application`:

```kotlin
// AndroidManifest.xml → <application android:name=".RookApplication">

class RookApplication : Application() {
    lateinit var serviceLocator: ServiceLocator

    override fun onCreate() {
        super.onCreate()
        serviceLocator = ServiceLocator(applicationContext)
    }
}

class ServiceLocator(context: Context) {
    val rookConfigurationManager: RookConfigurationManager by lazy {
        RookConfigurationManager(context).apply {
            val environment =
                if (BuildConfig.DEBUG) RookEnvironment.SANDBOX else RookEnvironment.PRODUCTION
            val rookConfiguration = RookConfiguration(CLIENT_UUID, SECRET, environment)

            if (BuildConfig.DEBUG) {
                enableLocalLogs() // MUST be called before setConfiguration to enable native logs
            }

            setConfiguration(rookConfiguration)
        }
    }
}
```

Then call `initRook()` on that singleton once (e.g. after the user opts in, or at first use).

## Register / update the user

After the SDK is initialized, register the app end-user whose data will be synced. The identifier is
`user_id` (**never** `customer_id`). Call `updateUserID` as part of your **login / initialization** flow.

```kotlin
rookConfigurationManager.updateUserID(userID).fold(
    {
        // Success
    },
    {
        // Handle error
    },
)
```

> Calling `updateUserID` with a **different** `userID` overrides the previous one and **resets the sync
> status** — if you use automatic sync, all health data will synchronize again.

### Don't re-register on every launch

Once `updateUserID` has succeeded, the `userID` is persisted. If the app is closed and reopened you do
**not** need to call it again. After initializing, use `getUserID` to check whether it was recovered from
storage:

```kotlin
// After initializing the SDK
val userID = rookConfigurationManager.getUserID()

if (userID != null) {
    // Recovered from storage — no need to call updateUserID again
} else {
    // Not configured yet — you MUST call updateUserID
}
```

### Logout

When the user logs out of your app, call `deleteUserFromRook` to remove the `userID` from the SDK and
disable it on the server (this only disables Health Connect and Android data sources).

```kotlin
rookConfigurationManager.deleteUserFromRook().fold(
    {
        // User removed from the SDK and disabled on the server
    },
    {
        // Handle error
    },
)
```

### User timezone

Every successful `updateUserID` also updates the user's timezone (look for `Timezone information updated`
in the logs). This is enough in most cases. To update the timezone manually, call `syncUserTimeZone`:

```kotlin
rookConfigurationManager.syncUserTimeZone().fold(
    {
        // User timezone updated successfully
    },
    {
      // Handle error
    },
)
```
