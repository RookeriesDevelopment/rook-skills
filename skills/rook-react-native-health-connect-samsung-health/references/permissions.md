# Availability and permissions — React Native · Health Connect and Samsung Health

Call `useRookPermissions()` once at component scope, wait for `ready`, and keep Health Connect, Samsung Health,
and optional Android sensor permissions in separate user flows.

## Contents

- [Recommended flow](#recommended-flow)
- [Health Connect availability](#health-connect-availability)
- [Health Connect permissions](#health-connect-permissions)
- [Health Connect background read](#health-connect-background-read)
- [Samsung Health availability](#samsung-health-availability)
- [Samsung Health permissions](#samsung-health-permissions)
- [Android sensor permissions](#android-sensor-permissions)
- [Background reliability settings](#background-reliability-settings)
- [Permission notifications](#permission-notifications)

## Recommended flow

1. Register the ROOK user.
2. Check the selected provider's availability.
3. Explain the requested health categories in product UI.
4. Check current permission state.
5. Request permissions from a user action.
6. Recheck state after the native dialog closes or a permission notification arrives.
7. Enable the provider's background service only if the user opted in.

## Health Connect availability

```tsx
const {
  ready,
  checkHealthConnectAvailability,
} = useRookPermissions();

const availability = ready
  ? await checkHealthConnectAvailability()
  : 'NOT_SUPPORTED';
```

Handle `INSTALLED`, `NOT_INSTALLED`, and `NOT_SUPPORTED`. Do not show a permission action until availability is
`INSTALLED`.

## Health Connect permissions

The library manifest declares these read categories in 4.1.1:

- Sleep, steps, distance, floors climbed, elevation gained.
- Oxygen saturation, VO2 max, total and active calories.
- Heart rate, resting heart rate, heart-rate variability.
- Exercise, speed, and power.
- Weight, height, blood glucose, and blood pressure.
- Hydration, body temperature, respiratory rate, nutrition, and menstruation.

Check all or partial permission state and request the permissions present in the merged manifest:

```tsx
const {
  healthConnectHasPermissions,
  healthConnectHasPartialPermissions,
  requestHealthConnectPermissions,
  openHealthConnectSettings,
  revokeHealthConnectPermissions,
} = useRookPermissions();

const hasAll = await healthConnectHasPermissions();
const hasSome = await healthConnectHasPartialPermissions();

if (!hasAll) {
  const request = await requestHealthConnectPermissions();
  // 'REQUEST_SENT' or 'ALREADY_GRANTED'
  console.log(request, hasSome);
}
```

The request result describes whether the request was sent, not which categories the user granted. Recheck afterward.
Use `openHealthConnectSettings()` for later changes. `revokeHealthConnectPermissions()` can appear deferred until
the app process stops.

## Health Connect background read

```tsx
const { checkHealthConnectBackgroundReadStatus } = useRookPermissions();

const background = await checkHealthConnectBackgroundReadStatus();
```

The result is `UNAVAILABLE`, `PERMISSION_NOT_GRANTED`, or `PERMISSION_GRANTED`. Background read availability
depends on the Android/Health Connect version. Continue to support foreground synchronization when unavailable.

## Samsung Health availability

```tsx
const { checkSamsungAvailability } = useRookPermissions();

const availability = await checkSamsungAvailability();
```

Handle `INSTALLED`, `NOT_INSTALLED`, `OUTDATED`, `DISABLED`, and `NOT_READY`. Do not request Samsung permissions
unless the result is `INSTALLED`.

## Samsung Health permissions

The 4.1.1 package exports:

- `ACTIVITY_SUMMARY`
- `BLOOD_GLUCOSE`
- `BLOOD_OXYGEN`
- `BLOOD_PRESSURE`
- `BODY_COMPOSITION`
- `EXERCISE`
- `EXERCISE_LOCATION`
- `FLOORS_CLIMBED`
- `HEART_RATE`
- `NUTRITION`
- `SLEEP`
- `SLEEP_APNEA`
- `STEPS`
- `WATER_INTAKE`

Request only types approved in Samsung's partner configuration:

```tsx
import {
  SamsungHealthPermission,
  useRookPermissions,
} from 'react-native-rook-sdk';

const {
  samsungHealthHasPermissions,
  samsungHealthHasPartialPermissions,
  requestSamsungHealthPermissions,
} = useRookPermissions();

const hasSome = await samsungHealthHasPartialPermissions();
const result = await requestSamsungHealthPermissions([
  SamsungHealthPermission.SLEEP,
  SamsungHealthPermission.STEPS,
  SamsungHealthPermission.HEART_RATE,
]);
```

Calling `requestSamsungHealthPermissions()` without an array requests every enum value. The two Samsung check
helpers evaluate the SDK's full native permission set rather than the custom request array, so a deliberately limited
integration may need the partial check plus app-maintained onboarding state.

## Android sensor permissions

These permissions are for the optional accelerometer/device step service, not ordinary Health Connect background
read:

```tsx
const {
  androidHasBackgroundPermissions,
  shouldRequestAndroidBackgroundPermissions,
  requestAndroidBackgroundPermissions,
} = useRookPermissions();

if (
  !(await androidHasBackgroundPermissions()) &&
  (await shouldRequestAndroidBackgroundPermissions())
) {
  await requestAndroidBackgroundPermissions();
}
```

The service can require activity recognition and notification/foreground-service permissions and displays a persistent
notification where Android requires it. Ask only when the user enables device-sensor step tracking.

## Background reliability settings

Version 4.1.1 exposes exact-alarm, battery-optimization, and OEM auto-start helpers:

```tsx
const {
  androidHasAlarmPermissions,
  requestAndroidAlarmPermissions,
  requestDisableBatteryOptimizations,
  requiresOemAutoStartSetup,
  openOemAutoStartSetup,
} = useRookPermissions();

if (!(await androidHasAlarmPermissions())) {
  await requestAndroidAlarmPermissions();
}

await requestDisableBatteryOptimizations();

if (await requiresOemAutoStartSetup()) {
  await openOemAutoStartSetup();
}
```

Request these settings only for a documented background use case and explain the battery impact. See
`troubleshooting.md` before relying on the 4.1.1 alarm-status helper.

## Permission notifications

Subscribe to `ROOK_NOTIFICATION` with `NativeEventEmitter(getRookModule())` and remove the subscription on unmount.
Relevant Android types include:

- `ROOK_BACKGROUND_ANDROID_PERMISSIONS`
- `ROOK_BACKGROUND_ANDROID_PERMISSIONS_DIALOG_DISPLAYED`
- `ROOK_HEALTH_CONNECT_PERMISSIONS`
- `ROOK_HEALTH_CONNECT_PERMISSIONS_PARTIALLY_GRANTED`
- `ROOK_SAMSUNG_HEALTH_PERMISSIONS`
- `ROOK_SAMSUNG_HEALTH_PERMISSIONS_PARTIALLY_GRANTED`
- `ROOK_BACKGROUND_ENABLED`

Use notifications to refresh application state; do not treat them as proof that health data was uploaded.
