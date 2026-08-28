# Manual sync — Capacitor · Apple Health

Use manual sync only when the product needs an explicit refresh or a targeted date/type. Initialize the
SDK, register a `user_id`, request the relevant HealthKit permissions, and confirm network access first.

All dates use `yyyy-MM-dd`. On iOS, omit `dataSource`; it is an Android-only selector and is ignored by the
Apple Health bridge.

## Contents

- [Sync summaries](#sync-summaries)
- [Sync events](#sync-events)
- [Current-day helpers](#current-day-helpers)
- [Write nutrition](#write-nutrition)
- [Handling results](#handling-results)

## Sync summaries

`RookSummaries` handles `sleep`, `physical`, and `body` summaries.

```ts
import { RookSummaries } from 'capacitor-rook-sdk';

// Upload missing summaries from the recent historical window.
await RookSummaries.sync({});

// Upload every summary type for one date.
await RookSummaries.sync({ date: '2026-08-26' });

// Upload selected summary types for one date.
const { result } = await RookSummaries.sync({
  date: '2026-08-26',
  types: ['sleep', 'physical'],
});
```

Use the returning helpers when the app also needs the normalized object locally:

```ts
const sleep = await RookSummaries.getSleepSummaries({ date: '2026-08-26' });
const physical = await RookSummaries.getPhysicalSummary({ date: '2026-08-26' });
const body = await RookSummaries.getBodySummary({ date: '2026-08-26' });

console.log(sleep.result, physical.result, body.result);
```

More than one sleep session can be returned for a date. Retry locally pending summary uploads with:

```ts
await RookSummaries.reSyncFailedSummaries();
```

## Sync events

`RookEvents.syncEvents` uploads one event type for one date:

```ts
import { RookEvents } from 'capacitor-rook-sdk';

await RookEvents.syncEvents({
  date: '2026-08-26',
  type: 'heart_rate',
});
```

Supported Apple Health event values are:

```text
activity, heart_rate, oxygenation, temperature, blood_pressure, blood_glucose,
calories, hydration, nutrition, steps, body_metrics, ecg
```

For `steps` and `calories`, the native Apple Health SDK uses the current day even though the bridge still
requires a `date` property.

Retrieve and upload activity events for a date:

```ts
const { result: activities } = await RookEvents.getActivityEvents({
  date: '2026-08-26',
});
```

Retry locally pending event uploads with:

```ts
await RookEvents.syncPendingEvents();
```

## Current-day helpers

These helpers upload current-day data and return a value suitable for the UI:

```ts
const { stepCount } = await RookEvents.getTodayStepCount();
const { basal, active } = await RookEvents.getTodayCalories();
const { result: heartRate } = await RookEvents.getTodayHeartRate();
```

## Write nutrition

`writeNutritionEvent` writes a new nutrition sample to HealthKit and uploads it. This differs from syncing
`type: 'nutrition'`, which reads nutrition already stored in HealthKit. Request dietary write permission
first and construct the event with the installed package's `NutritionEvent` type.

```ts
await RookPermissions.requestDietaryWritePermissions();
await RookEvents.writeNutritionEvent({ event: nutritionEvent });
```

## Handling results

- A `{ result: true }` response confirms the operation completed; it does not prove that every requested
  type contained data.
- Treat no records, missing permissions, locked HealthKit, invalid date, missing user, and network failure
  as different product states.
- Await a request before starting another request for the same date and data type.
- Remember that providers decide when their data reaches Apple Health; manual sync is not real time.
