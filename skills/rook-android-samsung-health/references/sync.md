# Sync health data (manual) — Android · Samsung Health

Prerequisite: the SDK is initialized, a `user_id` is registered, and Samsung Health permissions are
granted (see `references/setup-and-init.md` and `references/permissions.md`).

> **Prefer automatic sync.** ROOK's automatic/background implementation is faster, easier to integrate,
> and handles everything except permissions and `user_id`. Only build a manual sync process if you need
> custom control. See `references/background.md`.

## RookSamsung instance vs. RookSamsungObject

All the functions below exist on both the `RookSamsung` instance (recommended as a singleton via DI /
ServiceLocator) and the `RookSamsungObject` singleton (pass a `Context` on each call). The snippets use an
instance named `rookSamsung`.

## Data types

There are two kinds of health data, **Summaries** and **Events**:

| Health data | Timezone | Oldest retrievable | Latest retrievable |
|-------------|----------|--------------------|--------------------|
| Summary     | UTC      | 29 days ago        | Today (V4)         |
| Event       | UTC      | 29 days ago        | Today              |

Summary types: `SLEEP_SUMMARY`, `PHYSICAL_SUMMARY`, `BODY_SUMMARY`. A date is selected with `LocalDate`; a
specific type is selected with `SHSyncType.Summary` / `SHSyncType.Event`.

## Sync summaries

Sync the **last 29 days** of all three summaries with `sync(enableLogs)`:

```kotlin
rookSamsung.sync(enableLogs = isDebug).fold(
    { /* Historic summaries sync started */ },
    { /* Handle error */ },
)
```

Sync all three summaries for **a specific date** with `sync(date)`. It returns an `SHSyncSummariesResult`
holding one result per summary type; you can treat it as a single result (as below) or inspect each:

```kotlin
rookSamsung.sync(date = localDate).fold(
    { /* All summaries synced successfully */ },
    { /* At least one error occurred */ },
)
```

```kotlin
data class SHSyncSummariesResult(
    val sleepSummary: Result<SHSyncStatus>,
    val physicalSummary: Result<SHSyncStatus>,
    val bodySummary: Result<SHSyncStatus>,
)
```

Sync a **single summary type** for a date with `sync(date, summary)`:

```kotlin
rookSamsung.sync(date = localDate, summary = summary).fold(
    { /* Summary synced successfully */ },
    { /* Handle error */ },
)
```

## Sync events

Sync a chosen event type for a date with `syncEvents(date, event)`:

```kotlin
rookSamsung.syncEvents(date = localDate, event = event).fold(
    { /* Event synced successfully */ },
    { /* Handle error */ },
)
```

## Sync current-day events

These upload a small amount of current-day data **and** return the uploaded data, so you can update your UI
immediately without waiting for the webhook. Each returns a `SHSyncStatusWithData` (`RecordsNotFound`, or
`Synced` carrying the data).

```kotlin
// Steps — data is an Int
rookSamsung.getTodayStepsCount().fold(
    {
        when (it) {
            SHSyncStatusWithData.RecordsNotFound -> { /* No steps events found */ }
            is SHSyncStatusWithData.Synced -> {
                val steps: Int = it.data
                // Update UI
            }
        }
    },
    { /* Handle error */ },
)

// Calories — data is an SHCalories
rookSamsung.getTodayCaloriesCount().fold(
    {
        when (it) {
            SHSyncStatusWithData.RecordsNotFound -> { /* No calories events found */ }
            is SHSyncStatusWithData.Synced -> {
                val dailyCalories: SHCalories = it.data
                // Update UI
            }
        }
    },
    { /* Handle error */ },
)

// Heart rate — data is an SHHeartRate
rookSamsung.getTodayHeartRate().fold(
    {
        when (it) {
            SHSyncStatusWithData.RecordsNotFound -> { /* No heart rate events found */ }
            is SHSyncStatusWithData.Synced -> {
                val heartRate: SHHeartRate = it.data
                // Update UI
            }
        }
    },
    { /* Handle error */ },
)
```

## Sync and display (get a copy of what was uploaded)

To sync **and** receive a copy of the uploaded summary/event, use the `get*` functions below. Each returns
a `SHSyncStatusWithData` (`RecordsNotFound`, or `Synced` carrying the data):

| Function                    | `Synced` data type          |
|-----------------------------|-----------------------------|
| `getSleepSummary(date)`     | `List<SHSleepSummary>`      |
| `getPhysicalSummary(date)`  | `SHPhysicalSummary`         |
| `getBodySummary(date)`      | `SHBodySummary`             |
| `getActivityEvents(date)`   | `List<SHActivityEvent>`     |

```kotlin
rookSamsung.getSleepSummary(localDate).fold(
    {
        when (it) {
            SHSyncStatusWithData.RecordsNotFound -> { /* No records found */ }
            is SHSyncStatusWithData.Synced<List<SHSleepSummary>> -> {
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

> The returned data classes (`SHSleepSummary`, `SHPhysicalSummary`, `SHBodySummary`, `SHActivityEvent`) are
> large. Properties that Samsung Health doesn't support (or that this SDK doesn't process) are always
> `null`. For the full field-by-field definitions, see the docs:
> <https://docs.tryrook.io/docs/rookconnect/sdk/android/samsunghealth/fundamentals/usage-sync-healthdata-manually/#sync-and-display>
> For the complete API specification, see the Javadoc:
> <https://javadoc.io/doc/io.tryrook.android/rook-sdk-samsung/latest/io/tryrook/sdk/samsung/package-summary.html>
