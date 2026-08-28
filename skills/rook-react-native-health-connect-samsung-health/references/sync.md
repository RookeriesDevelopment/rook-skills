# Manual sync — React Native · Health Connect and Samsung Health

Use `useRookSync` to upload data, `useRookData` to retrieve supported local models, and `useRookVariables` for
current-day values. Gate `useRookSync` calls with `useRookConfiguration().ready`; `useRookSync` does not return its
own `ready` value. Honor the `ready` values exposed by the data and variable hooks.

## Contents

- [Select a provider](#select-a-provider)
- [Sync summaries](#sync-summaries)
- [Sync events](#sync-events)
- [Read local data](#read-local-data)
- [Read current-day values](#read-current-day-values)
- [Dates and request quota](#dates-and-request-quota)
- [Handle results](#handle-results)

## Select a provider

Use exported enum values:

```tsx
SDKDataSource.HEALTH_CONNECT
SDKDataSource.SAMSUNG_HEALTH
```

Pass one or both values to targeted sync. Local-data and current-day calls accept one source at a time. Do not pass
`SDKDataSource.APPLE_HEALTH` on Android.

## Sync summaries

The no-parameter form attempts both Health Connect and Samsung Health in 4.1.1:

```tsx
import { useRookConfiguration, useRookSync } from 'react-native-rook-sdk';

const { sync } = useRookSync();
const { ready } = useRookConfiguration();

if (ready) {
  sync((result) => {
    console.log(result.healthConnect?.status);
    console.log(result.samsungHealth?.status);
  });
}
```

Prefer a targeted call when the application selected one provider:

```tsx
sync(
  (result) => {
    const hc = result.healthConnect;
    if (!hc?.status) console.warn(hc?.error);
  },
  {
    date: '2026-08-27',
    summary: 'physical', // 'sleep' | 'body' | 'physical'
    sources: [SDKDataSource.HEALTH_CONNECT],
  }
);
```

Select Samsung or both providers when intended:

```tsx
sync(callback, {
  date: '2026-08-27',
  sources: [
    SDKDataSource.HEALTH_CONNECT,
    SDKDataSource.SAMSUNG_HEALTH,
  ],
});
```

Omit `summary` to sync sleep, body, and physical summaries for that date.

## Sync events

Shared events work with either Android provider:

```tsx
await syncEvents({
  date: '2026-08-27',
  event: SharedHealthEvents.ACTIVITY,
  sources: [SDKDataSource.HEALTH_CONNECT],
});
```

Use the provider-specific enum for unique events:

```tsx
await syncEvents({
  date: '2026-08-27',
  event: UniqueHealthConnectEvents.TEMPERATURE,
  sources: [SDKDataSource.HEALTH_CONNECT],
});

await syncEvents({
  date: '2026-08-27',
  event: UniqueSamsungHealthEvents.HYDRATION,
  sources: [SDKDataSource.SAMSUNG_HEALTH],
});
```

Shared values are `ACTIVITY`, `HEART_RATE`, `OXYGENATION`, `BLOOD_PRESSURE`, `BLOOD_GLUCOSE`,
`CALORIES`, `STEPS`, and `BODY_METRICS`. Health Connect adds `TEMPERATURE`, `HYDRATION`, and
`NUTRITION`; Samsung Health adds `HYDRATION` and `NUTRITION`.

Do not combine a provider-specific event with the other provider in `sources`; TypeScript permits the shared union,
but native support remains provider-specific.

## Read local data

```tsx
const {
  getSleepSummary,
  getPhysicalSummary,
  getBodySummary,
  getActivityEvents,
} = useRookData();

const params = {
  date: '2026-08-27',
  source: SDKDataSource.SAMSUNG_HEALTH,
};

const sleep = await getSleepSummary(params);
const physical = await getPhysicalSummary(params);
const body = await getBodySummary(params);
const activities = await getActivityEvents(params);
```

Switch `source` to `HEALTH_CONNECT` for Health Connect. These methods do not accept multiple providers.

## Read current-day values

```tsx
const { getTodaySteps, getTodayCalories, getTodayHeartRate } =
  useRookVariables();

const steps = await getTodaySteps(SDKDataSource.HEALTH_CONNECT);
const calories = await getTodayCalories(SDKDataSource.SAMSUNG_HEALTH);
const heartRate = await getTodayHeartRate(SDKDataSource.HEALTH_CONNECT);
```

`getTodaySteps` returns a string. These provider values are distinct from the optional device-sensor step counter in
`background.md`.

## Dates and request quota

- Use `yyyy-MM-dd` for summary, event, and local-data dates.
- The documented automatic historical window is up to 29 days through yesterday for summaries and through today for
  physical activity; current steps are today only.
- Health Connect summary extraction performs multiple record queries. Avoid repeated full-window sync and redundant
  event extraction.
- When Health Connect reports quota exhaustion, stop retrying for several hours and resume with a targeted request.

## Handle results

- `sync()` is callback-based and returns `void`; awaiting it does not await completion.
- Inspect each requested provider's `status` and `error`. One provider can fail while another succeeds.
- Disable duplicate UI actions while a summary sync callback is pending.
- Catch rejected promises from event, local-data, and current-day calls.
- The 4.1.1 Android `writeNutritionData` path is not a complete public permission flow; do not present it as supported
  until the installed package exposes and requests the required Health Connect write permission.
