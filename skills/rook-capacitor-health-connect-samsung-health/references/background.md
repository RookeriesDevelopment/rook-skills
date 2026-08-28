# Automatic sync — Capacitor · Health Connect and Samsung Health

Health Connect and Samsung Health have independent hourly background schedulers. Ask the user which
provider(s) to enable, persist that choice, and re-apply it after initialization on later app launches.

## Contents

- [Before scheduling](#before-scheduling)
- [Initialization flag](#initialization-flag)
- [Health Connect background sync](#health-connect-background-sync)
- [Samsung Health background sync](#samsung-health-background-sync)
- [Optional ROOK step counter](#optional-rook-step-counter)
- [Constraints](#constraints)

## Before scheduling

Complete these prerequisites:

1. Initialize ROOK and register `user_id`.
2. Confirm the selected provider is available.
3. Obtain the selected provider's data permissions.
4. For Health Connect, confirm `checkBackgroundReadStatus().result === 'PERMISSION_GRANTED'`.
5. Obtain the user's explicit opt-in to background upload.
6. Optionally request exact alarm, battery optimization, and OEM auto-start settings from
   `permissions.md`.

## Initialization flag

On Android, `enableBackgroundSync: true` attempts to schedule both Health Connect and Samsung Health after
native initialization, but only when Health Connect background read is granted. This combined behavior is
not appropriate when the user selected only one source.

Prefer initializing with `enableBackgroundSync: false` and calling the provider-specific methods below.
`enableEventsBackgroundSync` is Apple Health-specific in the current Android bridge.

## Health Connect background sync

The intended public Capacitor API is:

```ts
import { RookHealthConnect } from 'capacitor-rook-sdk';

const backgroundRead = await RookHealthConnect.checkBackgroundReadStatus();

if (backgroundRead.result === 'PERMISSION_GRANTED') {
  await RookHealthConnect.enableHealthConnectBackGround();
}

const status = await RookHealthConnect.isBackGroundUpdatesEnable();
console.log('Health Connect scheduled:', status.result);

// When the user turns it off:
await RookHealthConnect.disableHealthConnectBackGround();
```

Package 4.1.1 contains a method-name mismatch between this helper and the native Android bridge. Read
`troubleshooting.md` before shipping this path and use a package release where the bridge names align.

### Each Health Connect run

The hourly scheduler processes:

1. Today's activity, steps, calories, body metrics, heart rate, oxygenation, temperature, hydration,
   nutrition, blood pressure, and blood glucose events.
2. Today's sleep summary when eligible.
3. Yesterday's sleep, physical, and body summaries when eligible.
4. Missing activity events and sleep/physical/body summaries from up to 29 days back.

Summary reprocessing is limited by native SDK timing and content checks. Health Connect quota can pause
historical work until a later run.

The scheduler skips a run when required state is missing, including network, user ID, provider permissions,
background read, valid initialization, or quota recovery.

## Samsung Health background sync

```ts
import { RookSamsungHealth } from 'capacitor-rook-sdk';

await RookSamsungHealth.enableBackGroundUpdates();

const status = await RookSamsungHealth.isBackGroundUpdatesEnable();
console.log('Samsung Health scheduled:', status.result);

// When the user turns it off:
await RookSamsungHealth.disableBackGroundUpdates();
```

The Samsung scheduler follows the same high-level hourly sequence: today's events, eligible today/yesterday
summaries, then missing history up to 29 days. It requires Samsung Health permissions but does not use the
Health Connect background-read permission.

The scheduler can skip when there is no network, no registered user, missing Samsung permissions, or an
initialization error. Android power management may delay execution beyond one hour.

## Optional ROOK step counter

`RookStepsCounter` is a separate Android sensor-based source. It is not Health Connect or Samsung Health,
and should not be enabled by default when provider step data already meets the product's needs.

```ts
import { RookStepsCounter } from 'capacitor-rook-sdk';

const available = await RookStepsCounter.isRookStepsCounterAvailable();

if (available.result) {
  await RookStepsCounter.enableRookStepsCounter();

  const active = await RookStepsCounter.isRookStepsCounterActive();
  const today = await RookStepsCounter.getRookTodayStepsCount();

  console.log(active.result, today.stepCount);
}
```

Turn it off with:

```ts
await RookStepsCounter.disableRookStepsCounter();
```

Do not use the deprecated `RookHealthConnect.enableBackgroundAndroidSteps`,
`disableBackgroundAndroidSteps`, `isBackgroundAndroidStepsActive`, or
`syncTodayAndroidStepsCount` methods in new code.

The step counter requires Android activity/service permissions and may display a foreground-service
notification. Exact alarm access and battery/OEM settings can improve reliability but remain optional and
must be user-initiated.

## Constraints

- “Every hour” is a target cadence, not an exact guarantee; Android and OEM power management control
  execution.
- Low battery/storage, no network, force-stop, Doze, and vendor auto-start restrictions can delay work.
- Health Connect quotas can pause background and manual extraction.
- Direct Samsung Health does not work on emulators and requires Samsung Health 6.29+ on a ready device.
- Data may duplicate when both providers contain the same underlying wearable samples. Enable `ALL` or both
  schedulers only with an intentional source/deduplication policy.
- Cancel both provider schedulers and the optional step counter when the user revokes the corresponding
  opt-in. `deleteUserFromRook()` removes the user association but the app should still update its saved
  background preferences.
