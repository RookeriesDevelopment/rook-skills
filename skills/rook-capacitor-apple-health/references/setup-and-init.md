# Setup and initialization — Capacitor · Apple Health

## About Apple Health

Apple Health uses HealthKit as a container: apps and devices write data, and authorized apps read it.
Provider sync timing, device state, and iOS background scheduling determine when data becomes available.
HealthKit storage is encrypted while the device is locked, so background reads can be delayed until unlock.

## Contents

- [Requirements](#requirements)
- [Install the package](#install-the-package)
- [Configure the Xcode project](#configure-the-xcode-project)
- [Configure and initialize the SDK](#configure-and-initialize-the-sdk)
- [Register or update the user](#register-or-update-the-user)
- [Remove the user during logout](#remove-the-user-during-logout)

## Requirements

- `capacitor-rook-sdk` **4.1.1**.
- iOS deployment target **13.0** or later.
- Xcode **26.0** or later according to the Capacitor integration documentation.
- CocoaPods available for the native iOS dependency installation.
- A physical iPhone for HealthKit validation; do not rely on the simulator for integration testing.
- A ROOK `CLIENT_UUID` and `SECRET` registered for the selected environment and bundle identifier.

The package currently embeds native `RookSDK` **4.1.0** through its podspec. Update this reference whenever
the package or podspec changes.

## Install the package

From the Capacitor project root:

```bash
npm install capacitor-rook-sdk@4.1.1
npx cap sync ios
```

If CocoaPods did not resolve during `cap sync`, install the pods explicitly:

```bash
npx pod-install
```

Open the generated iOS workspace, not only the `.xcodeproj`, when CocoaPods is in use.

## Configure the Xcode project

1. Open the iOS workspace in Xcode and select the app target.
2. Under **Build Phases → Link Binary With Libraries**, add `HealthKit.framework` if it is not already
   linked transitively.
3. Under **Signing & Capabilities**, add **HealthKit**.
4. Enable **Background Delivery** inside the HealthKit capability when using background sync.
5. Add **Background Modes** and enable **Background fetch** when using background sync.
6. Add app-specific HealthKit usage descriptions to `Info.plist`:

```xml
<key>NSHealthShareUsageDescription</key>
<string>Explain why the app needs to read the user's health and fitness data.</string>
<key>NSHealthUpdateUsageDescription</key>
<string>Explain why the app needs to write nutrition data to Apple Health.</string>
```

The read description is required for Apple Health reads. The update description is required when the app
requests dietary write permission or writes nutrition data.

## Configure and initialize the SDK

Call `RookConfig.initRook` once from the native-app startup flow. Both background flags are required by
the package's current TypeScript type, even when they are `false`.

```ts
import { Capacitor } from '@capacitor/core';
import { RookConfig } from 'capacitor-rook-sdk';

export async function initializeRook(): Promise<void> {
  if (Capacitor.getPlatform() !== 'ios') return;

  await RookConfig.initRook({
    environment: 'sandbox', // Use 'production' only with production credentials.
    clientUUID: 'CLIENT_UUID',
    secret: 'SECRET',
    // Omit bundleId to use the app's bundle identifier.
    enableBackgroundSync: false,
    enableEventsBackgroundSync: false,
    enableLogs: false,
  });
}
```

Set `bundleId` only when ROOK is configured to use an override. Keep `enableLogs` restricted to debug
builds. The initialization promise resolves after the bridge accepts the configuration; handle rejection
before exposing Apple Health actions.

## Register or update the user

After initialization and login, register the stable identifier from the app's own user system:

```ts
import { RookConfig } from 'capacitor-rook-sdk';

export async function registerRookUser(userId: string): Promise<void> {
  const { result } = await RookConfig.updateUserId({ userId });

  if (!result) {
    throw new Error('ROOK did not register the user');
  }
}
```

Wait for success before requesting permissions or syncing. Calling `updateUserId` with a different value
replaces the stored ID and resets sync state; automatic sync can process the available Apple Health history
again for the new user.

Read the persisted identifier without re-registering it:

```ts
const { userId } = await RookConfig.getUserId();
```

Synchronize the user's timezone explicitly after a meaningful timezone change:

```ts
await RookConfig.syncUserTimeZone();
```

## Remove the user during logout

Call `deleteUserFromRook()` before discarding the local authenticated session. It removes the stored user,
revokes the ROOK-side Apple Health authorization, and disables automatic sync after successful removal.

```ts
export async function logoutFromRook(): Promise<void> {
  const { result } = await RookConfig.deleteUserFromRook();

  if (!result) {
    throw new Error('ROOK did not remove the user');
  }
}
```

Do not use the obsolete `clearUserId()` examples found in older Capacitor documentation.
