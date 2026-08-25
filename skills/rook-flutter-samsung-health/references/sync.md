# Sync health data (manual) — Flutter · Samsung Health

Prerequisite: the SDK is initialized, a `user_id` is registered, and Samsung Health permissions are
granted (see `references/setup-and-init.md` and `references/permissions.md`). All calls are **static** on
`RookSamsung` and return a `Future` — use `await` inside `try/catch`.

> **Prefer automatic sync.** ROOK's automatic/background implementation is faster, easier to integrate,
> and handles everything except permissions and `user_id`. Only build a manual sync process if you need
> custom control. See `references/background.md`.

> **No data is not an error you can ignore.** Sync/get calls that find no records for the requested date
> complete by throwing a `RecordsNotFoundException` — handle it in your `catch` block.

## Data types

There are two kinds of health data, **Summaries** and **Events**:

| Health data | Timezone | Oldest retrievable | Latest retrievable |
|-------------|----------|--------------------|--------------------|
| Summary     | UTC      | 29 days ago        | Today (V4)         |
| Event       | UTC      | 29 days ago        | Today              |

- **Summary types** — `SHSummarySyncType`: `sleep`, `physical`, `body`.
- **Event types** — `SHEventSyncType`: `activity`, `bloodGlucose`, `bloodPressure`, `bodyMetrics`,
  `heartRate`, `hydration`, `nutrition`, `oxygenation`, `temperature`, `steps`, `calories`.
- A date is selected with a `DateTime`.

## Sync summaries

`sync` takes three optional named parameters — `{bool? enableLogs, DateTime? date, SHSummarySyncType? summary}`
— and returns `Future<void>`.

Sync the **last 29 days** of all three summaries (`SLEEP_SUMMARY`, `PHYSICAL_SUMMARY`, `BODY_SUMMARY`) with
`sync(enableLogs: ...)`:

```dart
void syncSummariesHistoric() async {
  try {
    await RookSamsung.sync(enableLogs: kDebugMode);

    // Historic summaries sync started
  } catch (error) {
    // Handle error
  }
}
```

Sync all three summaries for **a specific date** with `sync(date: ...)`:

```dart
void syncSummaries() async {
  try {
    await RookSamsung.sync(date: date);

    // Summaries synced successfully
  } catch (error) {
    // Handle error (e.g. RecordsNotFoundException)
  }
}
```

Sync a **single summary type** for a date with `sync(date: ..., summary: ...)`:

```dart
void syncSingleSummary() async {
  try {
    await RookSamsung.sync(date: date, summary: summarySyncType);

    // Summary synced successfully
  } catch (error) {
    // Handle error
  }
}
```

## Sync events

Sync a chosen event type for a date with `syncEvents(DateTime date, SHEventSyncType event)` (returns
`Future<void>`):

```dart
void syncSingleEvent() async {
  try {
    await RookSamsung.syncEvents(date, eventSyncType);

    // Event synced successfully
  } catch (error) {
    // Handle error
  }
}
```

## Sync current-day events

These upload a small amount of current-day data **and** return the uploaded data, so you can update your
UI immediately without waiting for the webhook.

| Function                 | Returns               |
|--------------------------|-----------------------|
| `getTodayStepsCount()`   | `Future<int>`         |
| `getTodayCaloriesCount()`| `Future<DailyCalories>` |
| `getTodayHeartRate()`    | `Future<HeartRate>`   |

```dart
void syncStepsEvents() async {
  try {
    final steps = await RookSamsung.getTodayStepsCount();

    // steps is an int — update UI
  } catch (error) {
    // Handle error (e.g. RecordsNotFoundException)
  }
}

void syncCaloriesEvents() async {
  try {
    final calories = await RookSamsung.getTodayCaloriesCount();

    // calories is a DailyCalories — update UI
  } catch (error) {
    // Handle error
  }
}

void syncHeartRateEvents() async {
  try {
    final heartRate = await RookSamsung.getTodayHeartRate();

    // heartRate is a HeartRate — update UI
  } catch (error) {
    // Handle error
  }
}
```

## Sync and display (get a copy of what was uploaded)

To sync **and** receive a copy of the uploaded summary/event, use the `get*` functions below. Each throws
`RecordsNotFoundException` when there is no data for the date:

| Function                   | Returns                     |
|----------------------------|-----------------------------|
| `getSleepSummary(date)`    | `Future<List<SleepSummary>>` |
| `getPhysicalSummary(date)` | `Future<PhysicalSummary>`   |
| `getBodySummary(date)`     | `Future<BodySummary>`       |
| `getActivityEvents(date)`  | `Future<List<ActivityEvent>>` |

```dart
void example() async {
  final date = DateTime.now();

  try {
    final sleepSummaries = await RookSamsung.getSleepSummary(date);

    // sleepSummaries is a List<SleepSummary> — update UI
  } catch (error) {
    // Handle error (e.g. RecordsNotFoundException)
  }
}
```

`getPhysicalSummary`, `getBodySummary`, and `getActivityEvents` follow the same shape, differing only in the
returned type from the table above.

> The returned model classes (`SleepSummary`, `PhysicalSummary`, `BodySummary`, `ActivityEvent`) are large.
> Properties that Samsung Health doesn't support (or that this SDK doesn't process) are always `null`. For
> the full field-by-field definitions see the docs:
> <https://docs.tryrook.io/docs/rookconnect/sdk/flutter/samsunghealth/fundamentals/usage-sync-healthdata-manually/#sync-and-display>
> For the complete API specification see the pub.dev reference:
> <https://pub.dev/documentation/rook_sdk_samsung_health/latest/rook_sdk_samsung_health/RookSamsung-class.html>
