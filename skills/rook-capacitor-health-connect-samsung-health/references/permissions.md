# Availability and permissions — Capacitor · Health Connect and Samsung Health

## Contents

- [Recommended flow](#recommended-flow)
- [Health Connect](#health-connect)
- [Samsung Health](#samsung-health)
- [Android service permissions](#android-service-permissions)
- [Background reliability settings](#background-reliability-settings)

## Recommended flow

1. Initialize ROOK.
2. Check the chosen provider's availability.
3. Explain the requested data and ask from a user-initiated screen.
4. Check existing permission state and request only what the product needs.
5. Register permission-event listeners before requests that complete through Android activities.
6. Request optional background reliability settings only after the user opts into background behavior.

## Health Connect

### Check availability

```ts
import { RookHealthConnect } from 'capacitor-rook-sdk';

const { result } = await RookHealthConnect.checkAvailability();
```

| Result | Action |
| --- | --- |
| `INSTALLED` | Continue to permission checks. |
| `NOT_INSTALLED` | Ask the user to install Health Connect on supported Android versions. |
| `NOT_SUPPORTED` | Hide the Health Connect path and offer another supported source. |

### Declared read permissions

The plugin manifest declares these Health Connect reads and merges them into the application:

```text
READ_SLEEP, READ_STEPS, READ_DISTANCE, READ_FLOORS_CLIMBED,
READ_ELEVATION_GAINED, READ_OXYGEN_SATURATION, READ_VO2_MAX,
READ_TOTAL_CALORIES_BURNED, READ_ACTIVE_CALORIES_BURNED,
READ_HEART_RATE, READ_RESTING_HEART_RATE, READ_HEART_RATE_VARIABILITY,
READ_EXERCISE, READ_SPEED, READ_WEIGHT, READ_HEIGHT, READ_BLOOD_GLUCOSE,
READ_BLOOD_PRESSURE, READ_HYDRATION, READ_BODY_TEMPERATURE,
READ_RESPIRATORY_RATE, READ_NUTRITION, READ_MENSTRUATION, READ_POWER
```

Use Android manifest merge rules when the app must remove unused record permissions. Keep the resulting
manifest aligned with the Play Console Health apps declaration.

### Check and request

```ts
import { RookPermissions } from 'capacitor-rook-sdk';

const hasAll = await RookPermissions.checkHealthConnectPermissions();
const hasSome = await RookPermissions.checkHealthConnectPermissionsPartially();

if (!hasSome.result) {
  const request = await RookPermissions.requestHealthConnectPermissions();
  console.log(request.result); // REQUEST_SENT or ALREADY_GRANTED
}
```

Package 4.1.1 has inconsistent boolean result shapes; see `troubleshooting.md` before depending on
`hasAll` as a primitive boolean.

Listen for the final permission activity result and remove the listener with its owning lifecycle:

```ts
const healthConnectListener = await RookPermissions.addListener(
  'io.tryrook.permissions.healthConnect',
  ({ result }: { result: boolean }) => {
    console.log('Health Connect permissions:', result);
  },
);

// Later:
await healthConnectListener.remove();
```

Open Health Connect settings or revoke access when the product exposes those actions:

```ts
await RookPermissions.openHealthConnectSettings();
await RookPermissions.revokeHealthConnectPermissions();
```

Revocation may become visible only after the Android app process closes.

### Background read

Health Connect background sync needs a compatible provider version and
`READ_HEALTH_DATA_IN_BACKGROUND`. Inspect the current state with:

```ts
const { result } = await RookHealthConnect.checkBackgroundReadStatus();
```

Possible values:

- `UNAVAILABLE` — this Health Connect version/device cannot grant background read.
- `PERMISSION_NOT_GRANTED` — supported but not granted.
- `PERMISSION_GRANTED` — background read is available.

The Health Connect permission request includes background read when supported. Re-check this status after
the user returns from the permission activity.

## Samsung Health

### Check availability

```ts
import { RookSamsungHealth } from 'capacitor-rook-sdk';

const { result } = await RookSamsungHealth.checkSamsungHealthAvailability();
```

| Result | Action |
| --- | --- |
| `INSTALLED` | Continue to permissions. |
| `NOT_INSTALLED` | Ask the user to install Samsung Health. |
| `OUTDATED` | Ask the user to update to Samsung Health 6.29 or later. |
| `DISABLED` | Ask the user to enable Samsung Health. |
| `NOT_READY` | Ask the user to finish Samsung Health onboarding and accept its terms. |

### Select permissions

Request only types registered in the Samsung partnership and needed by visible app behavior:

```ts
import type { SamsungPermissionType } from 'capacitor-rook-sdk';

const samsungPermissions: SamsungPermissionType[] = [
  'ACTIVITY_SUMMARY',
  'EXERCISE',
  'HEART_RATE',
  'SLEEP',
  'STEPS',
];
```

The complete 4.1.1 union is:

```text
ACTIVITY_SUMMARY, BLOOD_GLUCOSE, BLOOD_OXYGEN, BLOOD_PRESSURE,
BODY_COMPOSITION, EXERCISE, EXERCISE_LOCATION, FLOORS_CLIMBED,
HEART_RATE, NUTRITION, SLEEP, SLEEP_APNEA, STEPS, WATER_INTAKE,
BODY_TEMPERATURE
```

Request the selected set:

```ts
const permissionRequest = await RookPermissions.requestSamsungHealthPermissions({
  types: samsungPermissions,
});

console.log(permissionRequest.result);
```

The public wrapper also exposes all/partial check methods:

```ts
await RookPermissions.checkSamsungHealthPermission({ types: samsungPermissions });
await RookPermissions.checkSamsungHealthPermissionPartially({ types: samsungPermissions });
```

Package 4.1.1 contains a native parameter-name mismatch in those two check methods. See
`troubleshooting.md`; do not silently treat rejection as denied permission.

## Android service permissions

Background work and the optional step counter use Android runtime permissions such as activity recognition
and notifications. Check and request them separately from provider data permissions:

```ts
const androidPermissions = await RookPermissions.checkAndroidPermissions();

if (!androidPermissions.result) {
  const request = await RookPermissions.requestAndroidPermissions();
  console.log(request.result); // REQUEST_SENT or ALREADY_GRANTED
}
```

Register the Android permission listener before requesting:

```ts
const androidListener = await RookPermissions.addListener(
  'io.tryrook.permissions.android',
  ({ result }: { result: boolean }) => {
    console.log('Android service permissions:', result);
  },
);
```

Remove it when the owning component/service is destroyed.

## Background reliability settings

These are optional special-access flows. Ask only after explaining why background sync or the step counter
needs more reliable execution.

### Exact alarms

```ts
const hasExactAlarm = await RookPermissions.checkExactAlarmPermissions();

if (!hasExactAlarm) {
  const request = await RookPermissions.requestExactAlarmPermissions();
  console.log(request.result);
}
```

On Android 12+, the request opens system settings. Re-check when the app resumes; Android does not provide
a direct callback. Android 11 and lower report access as available.

### Battery optimization exemption

```ts
const isExempt = await RookPermissions.checkBatteryOptimizationsDisabled();

if (!isExempt) {
  await RookPermissions.requestDisableBatteryOptimizations();
}
```

Re-check on resume. The app may need a Play Console justification for this request.

### OEM auto-start

```ts
if (await RookPermissions.requiresOemAutoStartSetup()) {
  await RookPermissions.openOemAutoStartSetup();
}
```

`true` means a known OEM settings screen exists, not that auto-start is currently blocked. Treat this as a
last-resort troubleshooting step because Android cannot reliably read the user's choice.
