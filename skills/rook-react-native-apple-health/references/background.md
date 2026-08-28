# Automatic sync — React Native · Apple Health

Apple Health automatic sync requires both native HealthKit background delivery and a JavaScript-side decision to
enable ROOK synchronization.

## Contents

- [Configure native background delivery](#configure-native-background-delivery)
- [Enable automatic sync at initialization](#enable-automatic-sync-at-initialization)
- [Control sync manually](#control-sync-manually)
- [Listen for automatic-sync status](#listen-for-automatic-sync-status)
- [Logout](#logout)
- [Constraints](#constraints)

## Configure native background delivery

In Xcode, select the application target and:

1. Add the **HealthKit** capability.
2. Enable **Background Delivery** under HealthKit.
3. Add **Background Modes** and enable **Background fetch**.

Register ROOK's native listeners during app launch. For an Objective-C++ `AppDelegate.mm`:

```objc
#import "AppDelegate.h"
#import "RookSDK/RookSDK-Swift.h"

- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  [[RookBackGroundSync shared] setBackListeners];
  return [super application:application
      didFinishLaunchingWithOptions:launchOptions];
}
```

Preserve the React Native template's existing initialization and insert only the listener call before returning.
For a Swift app delegate, call `RookBackGroundSync.shared.setBackListeners()`.

## Enable automatic sync at initialization

Set `enableBackgroundSync` on the root gate only after the application intends to use automatic sync:

```tsx
<RookSyncGate
  environment="sandbox"
  clientUUID="CLIENT_UUID"
  secret="SECRET"
  bundleId="com.example.app"
  enableLogs={__DEV__}
  enableBackgroundSync={true}
>
  <AppNavigator />
</RookSyncGate>
```

On iOS, the 4.1.1 gate verifies a stored user and inferred Apple Health permission state, enables native sync, and
starts a foreground manual sync. It retries the automatic-start path when the app returns to the active state.

Do not enable this flag before user registration and permission onboarding. If the flag is fixed to `true` from the
first render, initialization can finish before those prerequisites exist; either manage it as application state or use
the manual hook after onboarding.

## Control sync manually

Use `useRookAppleHealth` when the product provides an explicit automatic-sync toggle:

```tsx
import { useRookAppleHealth } from 'react-native-rook-sdk';

const {
  ready,
  enableBackGroundUpdates,
  disableBackGroundUpdates,
  isBackgroundUpdatesEnabled,
} = useRookAppleHealth();

const enableAppleHealthSync = async () => {
  if (!ready) return;
  await enableBackGroundUpdates();
  return isBackgroundUpdatesEnabled();
};
```

In 4.1.1, `enableBackGroundUpdates()` calls the native configuration manager's global `enableSync()`. The status
method reports that global sync state; there are no separate React Native controls for summary and event background
delivery.

## Listen for automatic-sync status

The gate sends `ROOK_BACKGROUND_ENABLED` through the shared `ROOK_NOTIFICATION` channel. Subscribe once and clean
up the listener:

```tsx
import { useEffect } from 'react';
import { NativeEventEmitter } from 'react-native';
import { getRookModule } from 'react-native-rook-sdk';

useEffect(() => {
  const emitter = new NativeEventEmitter(getRookModule());
  const subscription = emitter.addListener('ROOK_NOTIFICATION', (notice) => {
    if (notice.type === 'ROOK_BACKGROUND_ENABLED') {
      console.log(notice.value, notice.message);
    }

    if (notice.type === 'ROOK_APPLE_HEALTH_BACKGROUND_ERROR') {
      console.error(notice.code, notice.message);
    }
  });

  return () => subscription.remove();
}, []);
```

`ROOK_BACKGROUND_ENABLED` reports gate startup state. Native HealthKit background failures use
`ROOK_APPLE_HEALTH_BACKGROUND_ERROR`. Treat either notification as operational status, not as proof that a
particular sample was uploaded.

## Logout

Disable automatic sync before calling `removeUserFromRook`:

```tsx
await disableBackGroundUpdates();
await removeUserFromRook([SDKDataSource.APPLE_HEALTH]);
```

If the root gate still receives `enableBackgroundSync={true}`, it can attempt to start services again on an app-state
change. Update the application's gate state as part of logout.

## Constraints

- HealthKit decides when background delivery runs; React Native timers are not a substitute.
- Force-quitting the app, device power state, and iOS resource policy affect delivery timing.
- New Apple Watch samples first need to reach the paired iPhone's HealthKit store.
- Background delivery requires a physical device, correct entitlements, valid signing, permissions, and a registered
  ROOK user.
- Combine background delivery with a targeted manual retry path for user-visible recovery.
