# Setup & initialization — Android · Samsung Health

This covers installing the SDK, preparing the Android project, initializing the SDK, and registering the
user (`updateUserID`). It is self-contained — everything the integration needs is here.

## About Samsung Health

The ROOK Samsung Health SDK reads data directly from the **Samsung Health app** on the device using
Samsung's Health Data SDK. Because of that, it has hard device requirements:

- The **Samsung Health app v6.29 or later** must be installed.
- The device must run **Android 10 (API level 29) or above**. Samsung Health is available on all Samsung
  smartphones and also on non-Samsung Android smartphones.
- **Emulators are not supported** — you must test on a physical device.
- During development, you must enable Samsung Health
  [developer mode](https://developer.samsung.com/health/data/guide/developer-mode.html) on the test device.

## Requirements

- **Android Studio** Narwhal 4 Feature Drop | 2025.1.4 or higher is recommended.
- **`minSdk` 29**, **`targetSdk` 36** (set in the app module `build.gradle`).
- **ROOK SDK version** — this skill targets the **V4** line; use **4.1.0**. Keep in sync with the official
  docs; never invent a version.
- **Samsung Health Data SDK `.aar`** — `samsung-health-data-api-1.1.0.aar` (bundled separately; see below).
  This is Samsung's own artifact and has its **own** version (1.1.0), independent of the ROOK SDK (4.1.0).

## Install the dependency

1. Copy the `samsung-health-data-api-1.1.0.aar` file into your app module's `libs/` directory.

2. In the app module `build.gradle`, set the SDK levels and add the dependencies:

```groovy
android {
    defaultConfig {
        minSdk 29
        targetSdk 36
    }
}

dependencies {
    implementation("io.tryrook.android:rook-sdk-samsung:4.1.0")
    implementation(files("$rootDir/libs/samsung-health-data-api-1.1.0.aar"))

    // Required by the Samsung Health Data SDK .aar:
    implementation("com.google.code.gson:gson:2.13.2")
}
```

3. Apply the Kotlin **parcelize** plugin to your app module. Using a version catalog
   (`gradle/libs.versions.toml`):

```toml
[versions]
kotlin = "2.1.21"

[plugins]
org-jetbrains-kotlin-plugin-parcelize = { id = "org.jetbrains.kotlin.plugin.parcelize", version.ref = "kotlin" }
```

```groovy
plugins {
    alias(libs.plugins.org.jetbrains.kotlin.plugin.parcelize)
}
```

## Android configuration

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

### Secure storage (optional)

The SDK stores sensitive data (e.g. the current logged-in user) in password-protected storage. You may
optionally set a file name (without extension) and password via manifest `meta-data`. If you omit these,
the SDK uses a default file name and auto-generates a new password on each install / data clear.

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

### Request data access (before release)

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

## Configure & initialize the SDK

All SDK functions live on the `RookSamsung` class; you create an instance with an `applicationContext`.

Rules:

- Use placeholders `CLIENT_UUID` and `SECRET` in every snippet — **never** real values.
- `environment` is `SHEnvironment.SANDBOX` or `SHEnvironment.PRODUCTION`; each environment has its **own**
  `CLIENT_UUID` / `SECRET` and its own package-name+secret pair registered in the ROOK Portal. Typical
  pattern: SANDBOX on debug builds, PRODUCTION on release.
- **Register credentials first.** The app's `applicationId` (package name) and its secret must be
  registered in the ROOK Portal *before* `initRook()`, or initialization fails with
  `SHNotAuthorizedException`.
- Call `enableLocalLogs()` **before** `initRook()` if you want native logs.
- Initialize **once per app launch** — keep `RookSamsung` as a singleton (DI or ServiceLocator).

`SHConfiguration` parameters: `clientUUID`, `secret`, `environment`, and an optional `packageName` (if
omitted, the SDK reads it from `Context`).

Minimal initialization:

```kotlin
val rookSamsung = RookSamsung(applicationContext)

val environment = if (BuildConfig.DEBUG) SHEnvironment.SANDBOX else SHEnvironment.PRODUCTION
val configuration = SHConfiguration(CLIENT_UUID, SECRET, environment)

// Enable logging only on debug builds (must run before initRook)
if (BuildConfig.DEBUG) {
    rookSamsung.enableLocalLogs()
}

rookSamsung.initRook(configuration).fold(
    {
        // Success — SDK is initialized
    },
    {
        // Handle error (e.g. SHNotAuthorizedException if credentials aren't registered)
    },
)
```

### Keep RookSamsung as a singleton

`RookSamsung` must be a **single instance** for the app's lifetime.

- **If the app already uses a DI framework (Hilt, Koin, Dagger, etc.), provide it there** — do **not**
  create a separate ServiceLocator. For example, with Hilt:

```kotlin
@InstallIn(SingletonComponent::class)
@Module
object RookModule {
    @Provides
    @Singleton
    fun provideRookSamsung(@ApplicationContext context: Context): RookSamsung {
        return RookSamsung(context)
    }
}
```

  With Koin, expose it as a `single { RookSamsung(androidContext()) }` in a module.

- **Only if the app has no DI**, fall back to a dependency-free ServiceLocator held by your `Application`:

```kotlin
// AndroidManifest.xml → <application android:name=".RookSamsungApplication">

class RookSamsungApplication : Application() {
    lateinit var serviceLocator: ServiceLocator

    override fun onCreate() {
        super.onCreate()
        serviceLocator = ServiceLocator(applicationContext)
    }
}

class ServiceLocator(context: Context) {
    val rookSamsung: RookSamsung by lazy { RookSamsung(context) }
}
```

### RookSamsungObject alternative

`RookSamsungObject` is a singleton alternative to `RookSamsung` that needs no instance, but requires a
`Context` on every call. It is handy in places where you don't hold an instance (e.g. the `Application`
class or a `BroadcastReceiver`):

```kotlin
val configuration = SHConfiguration(CLIENT_UUID, SECRET, environment)

// RookSamsung (instance)
val rookSamsung = RookSamsung(applicationContext)
rookSamsung.initRook(configuration)

// RookSamsungObject (singleton, pass a Context each call)
RookSamsungObject.initRook(applicationContext, configuration)
```

## Register / update the user

After the SDK is initialized, register the app end-user whose data will be synced. The identifier is
`user_id` (**never** `customer_id`). Call `updateUserID` as part of your **login / initialization** flow.

```kotlin
rookSamsung.updateUserID(userID).fold(
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
val userID = rookSamsung.getUserID()

if (userID != null) {
    // Recovered from storage — no need to call updateUserID again
} else {
    // Not configured yet — you MUST call updateUserID
}
```

### Logout

When the user logs out of your app, call `deleteUserFromRook` to remove the `userID` from the SDK and
disable it on the server (this only disables the Samsung Health data source).

```kotlin
rookSamsung.deleteUserFromRook().fold(
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
rookSamsung.syncUserTimeZone().fold(
    {
        // User timezone updated successfully
    },
    {
        // Handle error
    },
)
```
