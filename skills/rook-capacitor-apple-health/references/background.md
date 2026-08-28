# Automatic sync — Capacitor · Apple Health

ROOK exposes continuous upload and HealthKit background delivery:

- `RookAppleHealth.enableAppleHealthSync()` enables foreground/continuous upload and both background
  categories through the native configuration manager.
- The granular background methods independently control summaries and events.

Neither mode replaces SDK initialization, user registration, or HealthKit authorization.

## Contents

- [Configure native background delivery](#configure-native-background-delivery)
- [Choose initial state](#choose-initial-state)
- [Enable all automatic sync](#enable-all-automatic-sync)
- [Control summaries and events separately](#control-summaries-and-events-separately)
- [Listen for background errors](#listen-for-background-errors)
- [Constraints](#constraints)

## Configure native background delivery

Enable **HealthKit → Background Delivery** and **Background Modes → Background fetch** in Xcode. Then
register the native HealthKit observers on every app launch in the Capacitor app delegate:

```swift
import Capacitor
import RookSDK
import UIKit

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
  var window: UIWindow?

  func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    RookBackGroundSync.shared.setBackListeners()
    return true
  }
}
```

Keep `setBackListeners()` in the native launch path even when the user turns sync on later in JavaScript.

## Choose initial state

The initialization flags enable summary and event background sync at configuration time:

```ts
await RookConfig.initRook({
  environment: 'sandbox',
  clientUUID: 'CLIENT_UUID',
  secret: 'SECRET',
  enableBackgroundSync: true,
  enableEventsBackgroundSync: true,
  enableLogs: false,
});
```

Prefer storing the user's opt-in and enabling the categories explicitly after login and permissions.
Use the explicit disable methods when turning off a previously enabled category; a `false` initialization
flag is not a runtime disable operation.

## Enable all automatic sync

After user registration and permission request:

```ts
import { Capacitor } from '@capacitor/core';
import { RookAppleHealth } from 'capacitor-rook-sdk';

if (Capacitor.getPlatform() === 'ios') {
  await RookAppleHealth.enableAppleHealthSync();
}
```

This enables continuous upload when the app opens and turns on background summaries and events. Disable
all three behaviors with:

```ts
await RookAppleHealth.disableAppleHealthSync();
```

`isAppleHealthSyncEnable()` reports the foreground/continuous state, not the two granular background
states.

## Control summaries and events separately

```ts
await RookAppleHealth.enableBackGroundUpdates();
await RookAppleHealth.enableBackGroundEventsUpdates();

const summaries = await RookAppleHealth.isBackGroundUpdatesEnable();
const events = await RookAppleHealth.isBackGroundEventsUpdatesEnable();

console.log(summaries.result, events.result);
```

Turn off only the desired category:

```ts
await RookAppleHealth.disableBackGroundUpdates();
await RookAppleHealth.disableBackGroundEventsUpdates();
```

## Listen for background errors

The public method is intentionally spelled `starListening()` in package 4.1.1. Register the Capacitor
listener and start the native notification bridge:

```ts
const listener = await RookAppleHealth.addListener(
  'io.tryrook.background.appleHealth.errors',
  ({ result }: { result: string }) => {
    console.error('Apple Health background error:', result);
  },
);

await RookAppleHealth.starListening();

// When the owning screen/service is disposed:
await listener.remove();
await RookAppleHealth.stopListening();
```

Do not register duplicate listeners on every render. Keep one listener in an app-level service or clean it
up with the owning lifecycle.

## Constraints

- iOS decides when background delivery runs; it is not real time and has no guaranteed interval.
- HealthKit reads may wait until the user unlocks the device.
- Network state, Low Power Mode, force-quitting, and system resource limits can delay uploads.
- Background sync processes only authorized data already written to Apple Health.
- Reconfirm the user's automatic-sync choice after changing `user_id`, because changing users resets sync
  state and may sync history again.
- `deleteUserFromRook()` disables automatic sync after successful user removal.
