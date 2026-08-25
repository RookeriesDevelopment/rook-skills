# Sync health data (manual) — Flutter · Apple Health

Prerequisite: the SDK is initialized, a `user_id` is registered, and Apple Health permissions are
requested (see `references/setup-and-init.md` and `references/permissions.md`).

> **Prefer automatic sync.** ROOK's automatic/background implementation is faster, easier to integrate,
> and handles everything except permissions and `user_id`. Only build a manual sync process if you need
> custom control. See `references/background.md`.

All manual-sync functions live on the **`AHRookSyncManager`** class and are `async` (await them inside a
`try`/`catch`).

## Data types

There are two kinds of health data, **Summaries** and **Events**:

| Health data | Timezone | Oldest retrievable | Latest retrievable |
|-------------|----------|--------------------|--------------------|
| Summary     | UTC      | N/A                | Today (V4)         |
| Event       | UTC      | N/A                | Today              |

- **Summary types** — `AHSummarySyncType`: `sleep`, `physical`, `body`.
- **Event types** — `AHEventSyncType`: `activity`, `bloodGlucose`, `bloodPressure`, `bodyMetrics`,
  `heartRate`, `nutrition`, `oxygenation`, `temperature`, `steps`, `calories`, `ecg`.
- A date is selected with a `DateTime`.

## Sync summaries

Sync the **last 29 days** of all three summaries (sleep, physical, body), **not including today**, with
`sync(enableLogs:)` → `Future<void>`:

```dart
void syncSummariesHistoric() async {
  try {
    await AHRookSyncManager.sync(enableLogs: isDebug);

    // Historic summaries sync started
  } catch (error) {
    // Handle error
  }
}
```

> **This call fails if the app goes to the background** while it runs. If you need the sync to continue
> to completion, use Background Sync instead (it syncs both summaries **and** events) — see
> `references/background.md`.

Sync all three summaries for **a specific date** with `sync(date:)`:

```dart
void syncSummaries() async {
  try {
    await AHRookSyncManager.sync(date: date);

    // Summaries synced successfully
  } catch (error) {
    // Handle error
  }
}
```

Sync a **single summary type** for a date with `sync(date:, summary:)`:

```dart
void syncSingleSummary() async {
  try {
    await AHRookSyncManager.sync(date: date, summary: AHSummarySyncType.sleep);

    // Summary synced successfully
  } catch (error) {
    // Handle error
  }
}
```

## Sync events

Sync a chosen event type for a date with `syncEvents(date, event)` → `Future<void>`:

```dart
void syncSingleEvent() async {
  try {
    await AHRookSyncManager.syncEvents(date, AHEventSyncType.steps);

    // Event synced successfully
  } catch (error) {
    // Handle error
  }
}
```

## Sync current-day events

These upload a small amount of current-day data **and** return the uploaded value, so you can update
your UI immediately without waiting for the webhook.

```dart
// Steps → Future<int>
void syncStepsEvents() async {
  try {
    final int steps = await AHRookSyncManager.getTodayStepsCount();

    // Update UI with steps
  } catch (error) {
    // Handle error
  }
}

// Calories → Future<DailyCalories>
void syncCaloriesEvents() async {
  try {
    final calories = await AHRookSyncManager.getTodayCaloriesCount();

    // Update UI with calories
  } catch (error) {
    // Handle error
  }
}

// Heart rate → Future<HeartRate>
void syncHeartRateEvents() async {
  try {
    final heartRate = await AHRookSyncManager.getTodayHeartRate();

    // Update UI with heart rate
  } catch (error) {
    // Handle error
  }
}
```

## Sync and display (get a copy of what was uploaded)

To sync **and** receive a copy of the uploaded summary/event, use the `get*` functions below. Each
syncs the data for the given date and returns it so you can render it right away:

| Function                   | Returns                    |
|----------------------------|----------------------------|
| `getSleepSummary(date)`    | `List<SleepSummary>`       |
| `getPhysicalSummary(date)` | `PhysicalSummary`          |
| `getBodySummary(date)`     | `BodySummary`              |
| `getActivityEvents(date)`  | `List<ActivityEvent>`      |

```dart
void example() async {
  final date = DateTime.now();

  try {
    final sleepSummaries = await AHRookSyncManager.getSleepSummary(date);

    // Update UI
  } catch (error) {
    // Handle error
  }
}
```

`getPhysicalSummary`, `getBodySummary`, and `getActivityEvents` follow the same shape, differing only in
the returned type from the table above.

> The returned objects carry many fields; properties that Apple Health doesn't provide (or that this SDK
> doesn't process) are `null`. For the full field-by-field definitions, see the API references:
> <https://pub.dev/documentation/rook_sdk_apple_health/latest/rook_sdk_apple_health/AHRookSyncManager-class.html>
> (methods) and <https://pub.dev/documentation/rook_sdk_core/latest/rook_sdk_core/> (data types).
