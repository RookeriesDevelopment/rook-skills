# Manual sync — Capacitor · Health Connect and Samsung Health

Initialize ROOK, register `user_id`, check provider availability, and request the matching permissions
before syncing. Pass dates as `yyyy-MM-dd` and pass `dataSource` explicitly.

## Contents

- [Source selection](#source-selection)
- [Sync summaries](#sync-summaries)
- [Retrieve uploaded summaries](#retrieve-uploaded-summaries)
- [Sync events](#sync-events)
- [Current-day helpers](#current-day-helpers)
- [Dates and request quota](#dates-and-request-quota)
- [Result handling](#result-handling)

## Source selection

| Value | Behavior |
| --- | --- |
| `HEALTH_CONNECT` | Read and upload Health Connect data only. This is the default when omitted. |
| `SAMSUNG` | Read and upload Samsung Health data directly. |
| `ALL` | Attempt both providers and combine operation success/failure for methods that support it. |

Use `ALL` only when the product intentionally ingests both sources and has a deduplication strategy. The
same wearable data can appear in Samsung Health and then be copied into Health Connect.

## Sync summaries

`RookSummaries` supports `sleep`, `physical`, and `body`.

```ts
import { RookSummaries } from 'capacitor-rook-sdk';

// Missing recent summaries from Health Connect.
await RookSummaries.sync({ dataSource: 'HEALTH_CONNECT' });

// Selected Samsung Health summaries for one date.
const samsungResult = await RookSummaries.sync({
  date: '2026-08-26',
  types: ['sleep', 'physical'],
  dataSource: 'SAMSUNG',
});

// Both sources for one date.
await RookSummaries.sync({
  date: '2026-08-26',
  types: ['sleep', 'physical', 'body'],
  dataSource: 'ALL',
});
```

Without `date`, the SDK processes missing summaries in its recent historical window. Prefer completed
days for summaries; same-day sleep, physical, and body data can still be incomplete.

## Retrieve uploaded summaries

The returning helpers accept one provider, not `ALL`:

```ts
const sleep = await RookSummaries.getSleepSummaries({
  date: '2026-08-26',
  dataSource: 'SAMSUNG',
});

const physical = await RookSummaries.getPhysicalSummary({
  date: '2026-08-26',
  dataSource: 'HEALTH_CONNECT',
});

const body = await RookSummaries.getBodySummary({
  date: '2026-08-26',
  dataSource: 'HEALTH_CONNECT',
});
```

`sleep.result` is an array because a date can contain multiple sleep sessions. Physical and body return
one normalized summary object.

`RookSummaries.reSyncFailedSummaries()` retries Health Connect uploads stored as pending by the Android
bridge; it has no provider selector.

## Sync events

```ts
import { RookEvents } from 'capacitor-rook-sdk';

await RookEvents.syncEvents({
  date: '2026-08-26',
  type: 'heart_rate',
  dataSource: 'HEALTH_CONNECT',
});

await RookEvents.syncEvents({
  date: '2026-08-26',
  type: 'activity',
  dataSource: 'SAMSUNG',
});
```

Provider support in the 4.1.1 Android bridge:

| Event value | Health Connect | Samsung Health |
| --- | --- | --- |
| `activity` | Yes | Yes |
| `heart_rate` | Yes | Yes |
| `oxygenation` | Yes | Yes |
| `temperature` | Yes | No targeted bridge mapping |
| `blood_pressure` | Yes | Yes |
| `blood_glucose` | Yes | Yes |
| `calories` | Yes | Yes |
| `hydration` | Yes | Yes |
| `nutrition` | Yes | Yes |
| `steps` | Yes | Yes |
| `body_metrics` | Yes | Yes |
| `ecg` | No Android bridge mapping | No Android bridge mapping |

The shared TypeScript `EventType` includes `ecg`, but package 4.1.1 implements it only in the native iOS
path. Do not use it for this Android skill.

Retrieve and upload activity events from one provider:

```ts
const { result: activities } = await RookEvents.getActivityEvents({
  date: '2026-08-26',
  dataSource: 'SAMSUNG',
});
```

`RookEvents.syncPendingEvents()` retries pending Health Connect event uploads and has no provider selector.

## Current-day helpers

Choose one Android provider with `source`:

```ts
const samsungSteps = await RookEvents.getTodayStepCount({ source: 'SAMSUNG' });
const hcCalories = await RookEvents.getTodayCalories({ source: 'HEALTH_CONNECT' });
const samsungHeartRate = await RookEvents.getTodayHeartRate({ source: 'SAMSUNG' });

console.log(
  samsungSteps.stepCount,
  hcCalories.basal,
  hcCalories.active,
  samsungHeartRate.result,
);
```

Omitting `source` defaults to Health Connect at runtime, but the installed TypeScript type may require it.

## Dates and request quota

- Events support recent historical dates through today; summaries are most reliable on completed days.
- Use a literal calendar date (`yyyy-MM-dd`), not a JavaScript ISO timestamp.
- Health Connect applies request quotas. Summaries query several record types, so repeated or overlapping
  calls can exhaust the quota quickly.
- Sync summaries no more than once daily unless the user explicitly retries.
- If physical activity already contains the data the product needs, avoid immediately syncing the same
  heart-rate and oxygenation events again.
- Back off for several hours after quota errors and prefer provider background sync for the default flow.

## Result handling

- `{ result: true }` means the requested extraction/upload completed; it does not mean every record type
  contained samples.
- Treat provider unavailable, permission missing, no records, invalid date, quota, unregistered user, and
  network errors as separate states.
- Await each operation and avoid concurrent requests for the same provider, date, and type.
