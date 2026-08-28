# Automatic sync — React Native · Health Connect and Samsung Health

Treat provider background upload and device-sensor step counting as separate capabilities. Provider sync reads health
stores; the optional step counter collects from Android sensors and may run a foreground service.

## Contents

- [Before enabling services](#before-enabling-services)
- [Enable automatic sync at initialization](#enable-automatic-sync-at-initialization)
- [Control Health Connect sync](#control-health-connect-sync)
- [Control Samsung Health sync](#control-samsung-health-sync)
- [Use the device step counter](#use-the-device-step-counter)
- [Customize the foreground notification](#customize-the-foreground-notification)
- [Listen for service status](#listen-for-service-status)
- [Logout](#logout)
- [Constraints](#constraints)

## Before enabling services

For each selected provider:

1. Confirm `ready` and a registered user.
2. Confirm provider availability.
3. Request and recheck provider permissions.
4. For Health Connect, inspect `checkHealthConnectBackgroundReadStatus()` when background read is required.
5. Enable only after an explicit product/user decision.

## Enable automatic sync at initialization

```tsx
<RookSyncGate
  environment="sandbox"
  clientUUID="CLIENT_UUID"
  secret="SECRET"
  packageName="com.example.app"
  enableLogs={__DEV__}
  enableBackgroundSync={true}
>
  <AppNavigator />
</RookSyncGate>
```

In 4.1.1, the gate attempts Samsung Health sync when Samsung is installed, then Health Connect native background
sync when partial Health Connect permission exists. The gate repeats its startup checks when the app becomes active.

Keep the flag `false` until user registration and permission onboarding are complete, or manage it through application
state. A permanently true flag can retry services after logout unless the app updates that state.

## Control Health Connect sync

```tsx
const {
  ready,
  scheduleBackgroundSync,
  cancelBackgroundSync,
  isBackgroundSyncEnabled,
} = useRookHealthConnect();

const enableHealthConnectSync = async () => {
  if (!ready) return false;

  const availability = await checkHealthConnectAvailability();
  const hasSome = await healthConnectHasPartialPermissions();
  if (availability !== 'INSTALLED' || !hasSome) return false;

  await scheduleBackgroundSync();
  return isBackgroundSyncEnabled();
};
```

Use `cancelBackgroundSync()` to stop the scheduled Health Connect work. Background read behavior varies by Android
and Health Connect version; retain a foreground/manual recovery path.

## Control Samsung Health sync

```tsx
const {
  ready,
  enableSamsungSync,
  disableSamsungSync,
  isSamsungSyncEnabled,
} = useRookSamsungHealth();

const enableSamsungBackground = async () => {
  if (!ready) return false;
  if ((await checkSamsungAvailability()) !== 'INSTALLED') return false;
  if (!(await samsungHealthHasPartialPermissions())) return false;

  await enableSamsungSync();
  return isSamsungSyncEnabled();
};
```

Samsung Health background sync does not use the Health Connect scheduler. Control it with the Samsung hook.

## Use the device step counter

Package 4.1.1 exports `useRookAndroidStepCounter` for the current sensor-based counter:

```tsx
const {
  ready,
  isStepsCounterAvailable,
  isStepsCounterActive,
  enableStepsCounter,
  disableStepsCounter,
  getTodayStepsCount,
} = useRookAndroidStepCounter();

if (ready && (await isStepsCounterAvailable())) {
  await enableStepsCounter();
  const active = await isStepsCounterActive();
  const steps = active ? await getTodayStepsCount() : 0;
}
```

Request Android sensor/background permissions from `permissions.md` before enabling it. The older
`useRookAndroidBackgroundSteps` hook remains exported for compatibility and returns step count as a string; do not
confuse either sensor value with Health Connect or Samsung Health steps.

## Customize the foreground notification

For the sensor-based step foreground service, add application metadata pointing to app resources:

```xml
<application ...>
  <meta-data
    android:name="io.tryrook.service.notification.SYNC_ICON"
    android:resource="@drawable/my_custom_icon" />
  <meta-data
    android:name="io.tryrook.service.notification.SYNC_TITLE"
    android:resource="@string/my_custom_title" />
  <meta-data
    android:name="io.tryrook.service.notification.SYNC_CONTENT"
    android:resource="@string/my_custom_content" />
</application>
```

## Listen for service status

Subscribe once to `ROOK_NOTIFICATION` through `NativeEventEmitter(getRookModule())`. Handle
`ROOK_BACKGROUND_ENABLED` to refresh operational UI, and remove the subscription on unmount. The notice does not
prove that a specific record reached ROOK.

## Logout

```tsx
await cancelBackgroundSync();
await disableSamsungSync();
await disableStepsCounter();
await removeUserFromRook([
  SDKDataSource.HEALTH_CONNECT,
  SDKDataSource.SAMSUNG_HEALTH,
]);
```

Also change the root gate to `enableBackgroundSync={false}`.

## Constraints

- Android WorkManager, Health Connect, Samsung Health, battery policy, and OEM process management determine exact
  execution timing.
- A force stop prevents background work until the user launches the app again.
- Sensor-based steps may require a persistent notification and can restart after reboot; exact timing is not guaranteed.
- Exact alarms, battery optimization exemption, and OEM auto-start settings require clear user benefit and consent.
- Test production signing/package registration and background behavior on representative physical devices.
