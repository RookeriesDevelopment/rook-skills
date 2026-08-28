# Manual sync — React Native · Apple Health

Use `useRookSync` to upload data, `useRookData` to retrieve supported local models, and `useRookVariables` for
current-day values. Gate `useRookSync` calls with `useRookConfiguration().ready`; `useRookSync` does not return its
own `ready` value. Honor the `ready` values exposed by the data and variable hooks, and use
`SDKDataSource.APPLE_HEALTH` only on iOS.

## Contents

- [Sync summaries](#sync-summaries)
- [Sync events](#sync-events)
- [Read local data](#read-local-data)
- [Read current-day values](#read-current-day-values)
- [Write nutrition](#write-nutrition)
- [Handle results](#handle-results)

## Sync summaries

The no-parameter form uploads the SDK's default historical window:

```tsx
import { useRookConfiguration, useRookSync } from 'react-native-rook-sdk';

const { sync } = useRookSync();
const { ready } = useRookConfiguration();

if (ready) {
  sync((result) => {
    const apple = result.appleHealth;
    if (!apple?.status) {
      console.warn('Apple Health sync did not complete', apple?.error);
    }
  });
}
```

Use parameters to target a date and optionally one summary pillar:

```tsx
import { SDKDataSource, useRookSync } from 'react-native-rook-sdk';

sync(
  (result) => {
    console.log(result.appleHealth?.status);
  },
  {
    date: '2026-08-27',
    summary: 'sleep', // 'sleep' | 'body' | 'physical'
    sources: [SDKDataSource.APPLE_HEALTH],
  }
);
```

Omit `summary` to sync sleep, body, and physical summaries for the selected date. Dates use `yyyy-MM-dd`.

## Sync events

Use shared events or Apple-specific events:

```tsx
import {
  SDKDataSource,
  SharedHealthEvents,
  UniqueAppleHealthEvents,
  useRookSync,
} from 'react-native-rook-sdk';

const { syncEvents } = useRookSync();

await syncEvents({
  date: '2026-08-27',
  event: SharedHealthEvents.ACTIVITY,
  sources: [SDKDataSource.APPLE_HEALTH],
});

await syncEvents({
  date: '2026-08-27',
  event: UniqueAppleHealthEvents.ECG,
  sources: [SDKDataSource.APPLE_HEALTH],
});
```

Shared values are `ACTIVITY`, `HEART_RATE`, `OXYGENATION`, `BLOOD_PRESSURE`, `BLOOD_GLUCOSE`,
`CALORIES`, `STEPS`, and `BODY_METRICS`. Apple-specific values are `TEMPERATURE` and `ECG` in the 4.1.1
source. Request the corresponding HealthKit types before syncing.

## Read local data

`useRookData` retrieves data without using the upload callback:

```tsx
const {
  getSleepSummary,
  getPhysicalSummary,
  getBodySummary,
  getActivityEvents,
} = useRookData();

const params = {
  date: '2026-08-27',
  source: SDKDataSource.APPLE_HEALTH,
};

const sleep = await getSleepSummary(params);
const physical = await getPhysicalSummary(params);
const body = await getBodySummary(params);
const activities = await getActivityEvents(params);
```

Treat empty results as either no matching data or unavailable read access; HealthKit does not distinguish them.

## Read current-day values

```tsx
const { getTodaySteps, getTodayCalories, getTodayHeartRate } =
  useRookVariables();

const steps = await getTodaySteps(SDKDataSource.APPLE_HEALTH);
const calories = await getTodayCalories(SDKDataSource.APPLE_HEALTH);
const heartRate = await getTodayHeartRate(SDKDataSource.APPLE_HEALTH);
```

`steps` is a string. Calories include active and basal values; native iOS also supplies a total value even though the
4.1.1 exported `Calories` type declares only active and basal.

## Write nutrition

Request write permission first, then pass an ISO-8601 timestamp with a timezone:

```tsx
import {
  MealDataType,
  SDKDataSource,
  useRookData,
  useRookPermissions,
} from 'react-native-rook-sdk';

const { requestWriteNutritionPermission } = useRookPermissions();
const { writeNutritionData } = useRookData();

await requestWriteNutritionPermission();
await writeNutritionData(
  {
    name: 'Lunch',
    date: '2026-08-28T13:00:00-06:00',
    mealType: MealDataType.LUNCH,
    calories: 620,
    protein: 32,
    totalCarbohydrates: 70,
    totalFat: 22,
  },
  SDKDataSource.APPLE_HEALTH
);
```

## Handle results

- Disable the sync action while a callback-based summary sync is running; `sync()` returns `void`.
- Inspect `result.appleHealth.status` and `result.appleHealth.error`; callback completion alone is not success.
- Catch rejected promises from event, local-data, variable, and nutrition calls.
- Avoid rapid repeated full-window syncs. Prefer one targeted date or pillar when the use case permits.
