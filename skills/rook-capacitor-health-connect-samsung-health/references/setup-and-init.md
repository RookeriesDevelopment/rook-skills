# Setup and initialization — Capacitor · Health Connect and Samsung Health

## Contents

- [Requirements](#requirements)
- [Install the package](#install-the-package)
- [Configure Samsung Health](#configure-samsung-health)
- [Configure Health Connect](#configure-health-connect)
- [Optional Android configuration](#optional-android-configuration)
- [Initialize ROOK](#initialize-rook)
- [Register or remove the user](#register-or-remove-the-user)

## Requirements

- `capacitor-rook-sdk` **4.1.1**; the current project is built with Capacitor 8 tooling.
- Android **API 29** or later. The package's current Android module sets `minSdkVersion 29`.
- `compileSdk` and `targetSdk` **36**, with compile SDK extension **19** in the plugin project.
- Java/JVM **17**; the package uses Kotlin **2.1.21** internally.
- Android Studio Narwhal 4 Feature Drop | 2025.1.4 or later is recommended.
- A physical Android device for Samsung Health; Samsung Health does not support emulators.
- Samsung Health **6.29+** on Android 10+ for direct Samsung Health access.
- A ROOK `CLIENT_UUID` and `SECRET` registered for the selected environment and package name.

The published Capacitor guide still mentions Android API 26, but the package 4.1.1 source declares API 29.
Use API 29 unless the installed package's Gradle metadata changes.

## Install the package

From the Capacitor project root:

```bash
npm install capacitor-rook-sdk@4.1.1
npx cap sync android
```

The package currently includes native Health Connect SDK 4.1.0 and Samsung Health SDK 4.1.0. Do not add
different ROOK native versions to the app module unless the plugin itself is updated to match.

## Configure Samsung Health

### Add Samsung's `.aar`

Place `samsung-health-data-api-1.1.0.aar` in the Capacitor Android Gradle root:

```text
android/libs/samsung-health-data-api-1.1.0.aar
```

Add the local artifact and its Gson dependency to `android/app/build.gradle`:

```groovy
android {
    defaultConfig {
        minSdkVersion 29
        targetSdkVersion 36
    }
}

dependencies {
    implementation files("$rootDir/libs/samsung-health-data-api-1.1.0.aar")
    implementation "com.google.code.gson:gson:2.13.0"
}
```

The filename and version must match the actual `.aar`; an older `1.0.0-b2` filename appears in historical
docs and must not be mixed with the 1.1.0 configuration.

### Obtain data access

- During development, enable Samsung Health developer mode on the physical test device.
- Before production distribution, submit Samsung's Partnership Request Form.
- Register the exact release package name and SHA-256 signing certificate.
- Select only the Samsung data types the app reads. Keep this set aligned with the runtime request in
  `permissions.md`.

Samsung Health direct access can report `NOT_INSTALLED`, `OUTDATED`, `DISABLED`, or `NOT_READY`; handle
these before asking for permissions.

## Configure Health Connect

The plugin manifest already declares its Health Connect read permissions and Android service permissions;
manifest merging adds them to the app. Do not duplicate the full list unless deliberately narrowing it.

Add a privacy-policy rationale destination for Android 13 and lower, plus the Android 14+ usage alias. Use
a native activity that displays the app's privacy policy, or route the Capacitor activity to an equivalent
screen.

```xml
<application>
    <activity
        android:name=".HealthConnectPrivacyPolicyActivity"
        android:enabled="true"
        android:exported="true">
        <intent-filter>
            <action android:name="androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE" />
        </intent-filter>
    </activity>

    <activity-alias
        android:name="ViewPermissionUsageActivity"
        android:exported="true"
        android:permission="android.permission.START_VIEW_PERMISSION_USAGE"
        android:targetActivity=".HealthConnectPrivacyPolicyActivity">
        <intent-filter>
            <action android:name="android.intent.action.VIEW_PERMISSION_USAGE" />
            <category android:name="android.intent.category.HEALTH_PERMISSIONS" />
        </intent-filter>
    </activity-alias>
</application>
```

Before Play Store release, complete the Play Console Health apps declaration for every Health Connect
record type the application reads.

### R8/ProGuard

At minimum, keep the crypto classes:

```text
-keep class com.google.crypto.** { *; }
```

If R8 full mode causes missing generic signatures, either set
`android.enableR8.fullMode=false` in `gradle.properties`, or add:

```text
-keep,allowobfuscation,allowshrinking interface retrofit2.Call
-keep,allowobfuscation,allowshrinking class retrofit2.Response
-keep,allowobfuscation,allowshrinking class kotlin.coroutines.Continuation
-keep class com.google.crypto.** { *; }
```

## Optional Android configuration

The SDK can store the current user in password-protected storage. If overriding defaults, provide both
name and password tags for each provider being used. Supply the password through the project's secure
build configuration.

```xml
<application>
    <meta-data
        android:name="io.tryrook.security.storage.FILE_NAME"
        android:value="rook_health_connect_storage" />
    <meta-data
        android:name="io.tryrook.security.storage.PASSWORD"
        android:value="SECURE_STORAGE_PASSWORD" />

    <meta-data
        android:name="io.tryrook.samsung.security.storage.FILE_NAME"
        android:value="rook_samsung_storage" />
    <meta-data
        android:name="io.tryrook.samsung.security.storage.PASSWORD"
        android:value="SECURE_STORAGE_PASSWORD" />
</application>
```

If either password is lost, that provider cannot read its stored state. Changing its filename recovers
operation but discards the prior local ROOK state.

## Initialize ROOK

The Android bridge configures both Health Connect and Samsung Health. Package 4.1.1 rejects initialization
when either native initialization fails, so complete the Samsung `.aar` setup even when Health Connect is
the primary source.

```ts
import { Capacitor } from '@capacitor/core';
import { RookConfig } from 'capacitor-rook-sdk';

export async function initializeRook(): Promise<void> {
  if (Capacitor.getPlatform() !== 'android') return;

  await RookConfig.initRook({
    environment: 'sandbox', // Use 'production' only with production credentials.
    clientUUID: 'CLIENT_UUID',
    secret: 'SECRET',
    // Omit packageName to use the Android application's package name.
    enableBackgroundSync: false,
    // Apple Health-specific in the current Android bridge; still required by the TS type.
    enableEventsBackgroundSync: false,
    enableLogs: false,
  });
}
```

Initialize once and expose the promise from an app-level service so screens cannot race initialization.
Enable logs only for development builds.

## Register or remove the user

Register the app's stable `user_id` after login and before permissions or sync. The Android bridge updates
both native providers.

```ts
const { result } = await RookConfig.updateUserId({ userId: 'USER_ID' });
```

Calling this method with a different user resets sync status and can cause history to be processed again.
Read the persisted ID with `RookConfig.getUserId()` rather than updating it on every launch.

During logout, remove the user from both providers before clearing the app session:

```ts
const { result } = await RookConfig.deleteUserFromRook();
```

Use `RookConfig.syncUserTimeZone()` after a meaningful timezone change. Do not use the obsolete
`clearUserId()` examples from older documentation.
