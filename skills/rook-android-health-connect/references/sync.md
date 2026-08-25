# Sync health data (manual) — Android · Health Connect

Prerequisite: the SDK is initialized, a `user_id` is registered, and Health Connect permissions are
granted (see `references/setup-and-init.md` and `references/permissions.md`).

> **Prefer automatic sync.** ROOK's automatic/background implementation is faster, easier to integrate,
> and handles everything except permissions and `user_id`. Only build a manual sync process if you need
> custom control. See `references/background.md`.

## Request quota (rate limits)

Health Connect rate-limits API access. Each ROOK data type is assembled from many Health Connect variables
(heart rate, steps, hydration, …), so a single summary triggers multiple Health Connect API calls — it's
possible to hit the limit, especially with repeated extractions in a short window. When syncing manually:

- **Extract summaries once per day.** Summaries cover the previous day, so more than one extraction per day
  is unnecessary.
- **Reuse what you already have.** Physical Events already include Heart Rate (Physical) and Oxygenation
  (Physical) events — don't extract those separately.
- **Sync only what your use case needs.** If you only want a date's summary and not individual events, use
  the `sync` functions on `RookSyncManager`.
- **Back off on quota errors.** If a call returns `HCQuotaExceededException`, stop calling any `sync`
  function for a few hours so the quota can recover.

## RookSyncManager

Use either an instance (recommended as a singleton via DI / ServiceLocator) or the Companion object with a
`context` per call:

```kotlin
// Instance (recommended)
val rookSyncManager = RookSyncManager(context)
rookSyncManager.doSomething()

// Or Companion, passing a context each call
RookSyncManager.doSomething(context)
```

## Data types

There are two kinds of health data, both handled by `RookSyncManager`:

| Health data | Timezone | Oldest retrievable | Latest retrievable |
|-------------|----------|--------------------|--------------------|
| Summary     | UTC      | 29 days ago        | Today (V4)         |
| Event       | UTC      | 29 days ago        | Today              |

Summary types: `SLEEP_SUMMARY`, `PHYSICAL_SUMMARY`, `BODY_SUMMARY`. Dates are selected with `LocalDate`;
a specific type is selected with `SyncType.Summary` / `SyncType.Event`.

## Sync summaries

Sync the **last 29 days** of all three summaries with `sync(enableLogs)`:

```kotlin
rookSyncManager.sync(enableLogs = isDebug).fold(
    { /* Historic summaries sync started */ },
    { /* Handle error */ },
)
```

> If the app goes to the background this synchronization will fail. To keep syncing until it finishes
> (summaries **and** events), use background sync — see `references/background.md`.

Sync all three summaries for **a specific date** with `sync(date)`. It returns an `HCSyncSummariesResult`
holding one result per summary type; you can treat it as a single result (as below) or inspect each:

```kotlin
rookSyncManager.sync(date = localDate).fold(
    { /* All summaries synced successfully */ },
    { /* At least one error occurred */ },
)
```

```kotlin
data class HCSyncSummariesResult(
    val sleepSummary: Result<SyncStatus>,
    val physicalSummary: Result<SyncStatus>,
    val bodySummary: Result<SyncStatus>,
)
```

Sync a **single summary type** for a date with `sync(date, summary)`:

```kotlin
rookSyncManager.sync(date = localDate, summary = summary).fold(
    { /* Summary synced successfully */ },
    { /* Handle error */ },
)
```

## Sync events

Sync a chosen event type for a date with `syncEvents(date, event)`:

```kotlin
rookSyncManager.syncEvents(date = localDate, event = event).fold(
    { /* Event synced successfully */ },
    { /* Handle error */ },
)
```

## Sync current-day events

These upload a small amount of current-day data **and** return the uploaded data, so you can update your UI
immediately without waiting for the webhook. Each returns a `SyncStatusWithData` (`RecordsNotFound` or
`Synced` with the data).

> **Each of these contributes to the Health Connect rate limit — don't call them too frequently.**

```kotlin
// Steps — data is an Int
rookSyncManager.getTodayStepsCount().fold(
    {
        when (it) {
            SyncStatusWithData.RecordsNotFound -> { /* No steps events found */ }
            is SyncStatusWithData.Synced -> {
                val steps: Int = it.data
                // Update UI
            }
        }
    },
    { /* Handle error */ },
)

// Calories — data is an HCCalories
rookSyncManager.getTodayCaloriesCount().fold(
    {
        when (it) {
            SyncStatusWithData.RecordsNotFound -> { /* No calories events found */ }
            is SyncStatusWithData.Synced -> {
                val calories: HCCalories = it.data
                // Update UI
            }
        }
    },
    { /* Handle error */ },
)

// Heart rate — data is an HCHeartRate
rookSyncManager.getTodayHeartRate().fold(
    {
        when (it) {
            SyncStatusWithData.RecordsNotFound -> { /* No heart rate events found */ }
            is SyncStatusWithData.Synced -> {
                val heartRate: HCHeartRate = it.data
                // Update UI
            }
        }
    },
    { /* Handle error */ },
)
```

## Sync and display (get a copy of what was uploaded)

To sync **and** receive a copy of the uploaded summary/event, use the `get*` functions below. Each returns
a `SyncStatusWithData` (`RecordsNotFound`, or `Synced` carrying the data):

| Function                    | `Synced` data type          |
|-----------------------------|-----------------------------|
| `getSleepSummary(date)`     | `List<HCSleepSummary>`      |
| `getPhysicalSummary(date)`  | `HCPhysicalSummary`         |
| `getBodySummary(date)`      | `HCBodySummary`             |
| `getActivityEvents(date)`   | `List<HCActivityEvent>`     |

```kotlin
rookSyncManager.getSleepSummary(localDate).fold(
    {
        when (it) {
            SyncStatusWithData.RecordsNotFound -> { /* No records found */ }
            is SyncStatusWithData.Synced<List<HCSleepSummary>> -> {
                val sleepSummaries = it.data
                // Update UI
            }
        }
    },
    { /* Handle error */ },
)
```

`getPhysicalSummary`, `getBodySummary`, and `getActivityEvents` follow the same shape, differing only in the
`Synced` data type from the table above.

> The returned data classes (`HCSleepSummary`, `HCPhysicalSummary`, `HCBodySummary`, `HCActivityEvent`) are
> large. Fields that Health Connect doesn't support (or that this SDK doesn't process) are always `null`.
> For the full field-by-field definitions, see the docs:
> <https://docs.tryrook.io/docs/rookconnect/sdk/android/healthconnect/fundamentals/usage-sync-healthdata-manually/#sync-and-display>
> For the complete API specification, see the Javadoc:
> <https://javadoc.io/doc/com.rookmotion.android/rook-sdk/latest/com/rookmotion/rook/sdk/package-summary.html>
