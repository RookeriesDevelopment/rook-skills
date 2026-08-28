# Setup and initialization — React Native · Health Connect and Samsung Health

Health Connect and Samsung Health are independent Android data sources. Health Connect is an on-device container
that authorized apps write to and read from; delivery timing depends on each writer. Samsung Health is accessed
through Samsung's Health Data SDK and requires its app, SDK `.aar`, and approved application identity. Check,
authorize, and select each provider independently.

## Contents

- [Requirements](#requirements)
- [Install the package](#install-the-package)
- [Add the Samsung Health Data SDK](#add-the-samsung-health-data-sdk)
- [Configure Samsung access](#configure-samsung-access)
- [Configure Health Connect](#configure-health-connect)
- [Configure R8](#configure-r8)
- [Initialize with RookSyncGate](#initialize-with-rooksyncgate)
- [Register or update the user](#register-or-update-the-user)
- [Remove the user during logout](#remove-the-user-during-logout)

## Requirements

For the documented 4.x line, use:

- `react-native-rook-sdk` 4.1.1.
- React Native 0.70 through 0.77 according to the 4.x documentation. The 4.1.1 source project is built with
  React Native 0.73.6.
- Android API 29 or newer, `compileSdk` and `targetSdk` 36, SDK extension 19, and Kotlin 2.1.21. These values come
  from the 4.1.1 package source and supersede older API 26 examples.
- A physical Android device. Health Connect and Samsung Health behavior cannot be validated reliably in an emulator.
- For Samsung Health: the Samsung Health app, Samsung Health Data SDK 1.1.0, developer mode for development, and
  approved package name plus SHA-256 signing certificate for production.
- A ROOK `client_uuid`, `secret`, and Android package name registered for the selected environment.

Do not copy APIs from the separate 5.x TurboModule repository into a 4.x integration.

## Install the package

Use the app's existing package manager:

```bash
npm install react-native-rook-sdk@4.1.1
```

```bash
yarn add react-native-rook-sdk@4.1.1
```

```bash
pnpm add react-native-rook-sdk@4.1.1
```

Rebuild the Android application after installation; Metro refresh cannot link the native modules.

Align the app's Android configuration with the package requirements:

```groovy
android {
    compileSdkVersion 36
    compileSdkExtension 19

    defaultConfig {
        minSdkVersion 29
        targetSdkVersion 36
    }
}
```

## Add the Samsung Health Data SDK

The 4.1.1 Android bridge references Samsung Health classes even when the product primarily uses Health Connect, so
keep the Samsung `.aar` available to the Android build.

Place `samsung-health-data-api-1.1.0.aar` at:

```text
android/libs/samsung-health-data-api-1.1.0.aar
```

Add it to `android/app/build.gradle`:

```groovy
dependencies {
    implementation(files("$rootDir/libs/samsung-health-data-api-1.1.0.aar"))
}
```

The older React Native getting-started page names version 1.0.0, but the current 4.1.1 example and native Samsung
dependency use 1.1.0. Keep the copied artifact name and Gradle path identical.

## Configure Samsung access

- Enable Samsung Health developer mode while developing.
- Before distribution, complete Samsung's partnership request with the exact application package name and release
  signing-certificate SHA-256.
- Request only Samsung data types approved for that application. A mismatched package name or signing certificate
  prevents production access even if debug testing worked.

See `permissions.md` for the exported `SamsungHealthPermission` values.

## Configure Health Connect

Add the Health Connect permission-rationale routes to the app's `AndroidManifest.xml`. Merge these into the existing
main activity and application; do not create a second launcher activity:

```xml
<application ...>
  <activity android:name=".MainActivity" ...>
    <intent-filter>
      <action android:name="androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE" />
    </intent-filter>
  </activity>

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

The library manifest supplies its default Health Connect read and Android service permissions. To remove an unused
Health Connect permission, add `xmlns:tools="http://schemas.android.com/tools"` to the app manifest and remove the
merged entry explicitly:

```xml
<uses-permission
  android:name="android.permission.health.READ_MENSTRUATION"
  tools:node="remove" />
```

Keep Play Console's Health apps declaration aligned with the final merged manifest. Do not remove permissions that
the application later requests or depends on.

## Configure R8

At minimum, add this to `android/app/proguard-rules.pro` when shrinking:

```text
-keep class com.google.crypto.** { *; }
```

Either disable R8 full mode in project `gradle.properties`:

```properties
android.enableR8.fullMode=false
```

or preserve Retrofit and coroutine generic signatures as documented by ROOK:

```text
-keep,allowobfuscation,allowshrinking interface retrofit2.Call
-keep,allowobfuscation,allowshrinking class retrofit2.Response
-keep,allowobfuscation,allowshrinking class kotlin.coroutines.Continuation
-keep class com.google.crypto.** { *; }
```

## Initialize with RookSyncGate

Wrap the native application once near the root:

```tsx
import React from 'react';
import { RookSyncGate } from 'react-native-rook-sdk';

export default function App() {
  return (
    <RookSyncGate
      environment="sandbox"
      clientUUID="CLIENT_UUID"
      secret="SECRET"
      packageName="com.example.app"
      enableLogs={__DEV__}
      enableBackgroundSync={false}
    >
      <AppNavigator />
    </RookSyncGate>
  );
}
```

- Use `environment="production"` only with production credentials and the release package name registered in ROOK.
- Use `useRookConfiguration().ready` as the shared call gate and honor hook-specific `ready` values where exposed.
- Keep `enableBackgroundSync={false}` until the user is registered, provider permissions are granted, and the
  product intentionally enables automatic upload.
- React Native build-time environment values are included in the bundle. Follow the app's approved credential
  provisioning design and never log the secret.

## Register or update the user

Register after login and before requesting provider permissions or syncing:

```tsx
const { ready, getUserID, updateUserID, syncUserTimeZone } =
  useRookConfiguration();

const registerRookUser = async (userID: string) => {
  if (!ready) return false;

  const current = await getUserID();
  if (current !== userID) {
    await updateUserID(userID);
  }

  await syncUserTimeZone();
  return true;
};
```

`updateUserID` replaces the stored ID and resets sync state. In 4.1.1, Android updates Samsung first when it is
available and then Health Connect; do not repeatedly call it for an unchanged user.

## Remove the user during logout

Stop all enabled services, then delete the user for the sources the application used:

```tsx
await cancelBackgroundSync();
await disableSamsungSync();
await disableStepsCounter();

await removeUserFromRook([
  SDKDataSource.HEALTH_CONNECT,
  SDKDataSource.SAMSUNG_HEALTH,
]);
```

Pass only applicable Android source values. Removing a user does not revoke Health Connect or Samsung Health OS
permissions. Also set the root gate's `enableBackgroundSync` state to `false` so app-state changes do not restart
automatic services after logout.
