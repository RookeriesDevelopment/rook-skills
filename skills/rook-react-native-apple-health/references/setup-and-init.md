# Setup and initialization — React Native · Apple Health

## About Apple Health

Apple Health is an on-device **store** managed by HealthKit: apps and devices write health samples into it,
and authorized apps read them. ROOK reads the data available in that store; it does not control when an Apple
Watch or another provider writes new samples. A successful ROOK integration therefore depends on HealthKit
configuration, user authorization, source data availability, and the selected sync strategy.

## Contents

- [Requirements](#requirements)
- [Install the package](#install-the-package)
- [Configure the Xcode project](#configure-the-xcode-project)
- [Initialize with RookSyncGate](#initialize-with-rooksyncgate)
- [Register or update the user](#register-or-update-the-user)
- [Remove the user during logout](#remove-the-user-during-logout)

## Requirements

For the documented 4.x line, use:

- `react-native-rook-sdk` 4.1.1; it embeds native `RookSDK` 4.1.0 on iOS.
- React Native 0.70 through 0.77 according to the 4.x documentation. The 4.1.1 source project is built with
  React Native 0.73.6.
- iOS 13 or newer and Xcode 26 or newer according to the current ROOK documentation. A newer React Native
  version may impose a higher deployment target; use the higher requirement.
- CocoaPods and a physical iPhone for realistic HealthKit testing. Simulator data is synthetic and cannot
  reproduce Apple Watch delivery.
- A ROOK `client_uuid`, `secret`, and registered iOS bundle identifier for the selected environment.

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

Install pods and rebuild the native app; a Metro refresh cannot link a new native module:

```bash
npx pod-install
```

## Configure the Xcode project

Open the `.xcworkspace`, select the application target, and configure:

1. Link `HealthKit.framework` if Xcode has not linked it automatically.
2. Add the **HealthKit** capability.
3. Add clear, app-specific purpose strings to `Info.plist`:

```xml
<key>NSHealthShareUsageDescription</key>
<string>Explain why this app reads health and fitness data.</string>
<key>NSHealthUpdateUsageDescription</key>
<string>Explain why this app writes nutrition data.</string>
```

Keep `NSHealthUpdateUsageDescription` when the app calls `writeNutritionData`; HealthKit can terminate an app
that requests a protected capability without its purpose string. Configure background delivery separately in
`background.md` only when automatic sync is required.

## Initialize with RookSyncGate

Wrap the native application once, normally near the root component:

```tsx
import React from 'react';
import { RookSyncGate } from 'react-native-rook-sdk';

export default function App() {
  return (
    <RookSyncGate
      environment="sandbox"
      clientUUID="CLIENT_UUID"
      secret="SECRET"
      bundleId="com.example.app"
      enableLogs={__DEV__}
      enableBackgroundSync={false}
    >
      <AppNavigator />
    </RookSyncGate>
  );
}
```

- Use `environment="production"` only with production credentials and the production bundle ID registered in
  ROOK.
- Supply the actual bundle identifier through build configuration; do not leave a sample value in release code.
- Set `enableBackgroundSync` deliberately. Keep it `false` until native background delivery, user registration,
  and permissions are ready.
- Use `useRookConfiguration().ready` as the shared call gate and honor hook-specific `ready` values where exposed.
  Initialization errors are logged by `RookSyncGate`, so readiness that never becomes true usually indicates
  credentials, bundle ID, native linking, or configuration failure.

React Native build-time environment variables are bundled into the application and are not secret storage.
Follow the application's approved credential-provisioning design and never log the secret.

## Register or update the user

Register the authenticated app user after the hook is ready and before permissions or sync:

```tsx
import { useRookConfiguration } from 'react-native-rook-sdk';

function useRookUser() {
  const { ready, getUserID, updateUserID, syncUserTimeZone } =
    useRookConfiguration();

  const registerUser = async (userID: string) => {
    if (!ready) return false;

    const current = await getUserID();
    if (current !== userID) {
      await updateUserID(userID);
    }

    await syncUserTimeZone();
    return true;
  };

  return { ready, registerUser };
}
```

`updateUserID` replaces the previously stored ID and resets sync state. Do not call it on every render or launch
when the same user is already registered, because automatic sync may upload the historical window again.

## Remove the user during logout

Stop automatic Apple Health sync first, then delete the user from ROOK:

```tsx
import {
  SDKDataSource,
  useRookAppleHealth,
  useRookConfiguration,
} from 'react-native-rook-sdk';

const { disableBackGroundUpdates } = useRookAppleHealth();
const { removeUserFromRook } = useRookConfiguration();

const logoutFromRook = async () => {
  await disableBackGroundUpdates();
  await removeUserFromRook([SDKDataSource.APPLE_HEALTH]);
};
```

On iOS, the hook's `sources` array satisfies the shared TypeScript signature; the native implementation removes
the Apple Health user regardless of the array contents. Removing the user does not revoke HealthKit permission.
If the same person returns, call `updateUserID` and re-establish the intended sync state.
