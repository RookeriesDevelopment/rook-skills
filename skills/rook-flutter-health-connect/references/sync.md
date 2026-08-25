# Sync health data (manual) — Flutter · Health Connect

Prerequisite: the SDK is initialized, a `user_id` is registered, and Health Connect permissions are
granted (see `references/setup-and-init.md` and `references/permissions.md`). Android only — guard calls
with `Platform.isAndroid`.

> **Prefer automatic sync.** ROOK's automatic/background implementation is faster, easier to integrate,
> and handles everything except permissions and `user_id`. Only build a manual sync process if you need
> custom control. See `references/background.md`.

The manual sync API is exposed through the static `HCRookSyncManager`.

## Request quota (rate limits)

Health Connect rate-limits API access. Each ROOK data type is assembled from many Health Connect variables
(heart rate, steps, hydration, …), so a single summary triggers multiple Health Connect API calls — it's
possible to hit the limit, especially with repeated extractions in a short window. When syncing manually:

- **Extract summaries once per day.** Summaries cover the previous day, so more than one extraction per day
  is unnecessary.
- **Reuse what you already have.** Physical Events already include Heart Rate (Physical) and Oxygenation
  (Physical) events — don't extract those separately.
- **Sync only what your use case needs.** If you only want a date's summary and not individual events, use
  the `sync` functions on `HCRookSyncManager`.
- **Back off on quota errors.** If a call throws `HealthConnectQuotaExceededException`, stop calling any
  `sync` function for a few hours so the quota can recover.

## Data types

There are two kinds of health data, both handled by `HCRookSyncManager`:

| Health data | Timezone | Oldest retrievable | Latest retrievable |
|-------------|----------|--------------------|--------------------|
| Summary     | UTC      | 29 days ago        | Today (V4)         |
| Event       | UTC      | 29 days ago        | Today              |

Summary types: `SLEEP_SUMMARY`, `PHYSICAL_SUMMARY`, `BODY_SUMMARY`. Dates are selected with a `DateTime`; a
specific type is selected with `HCSummarySyncType` (summaries) or `HCEventSyncType` (events).

## Sync summaries

Sync the **last 29 days** of all three summaries with `sync(enableLogs:)`:

```dart
void syncSummariesHistoric() async {
  try {
    await HCRookSyncManager.sync(enableLogs: isDebug);

    // Historic summaries sync started
  } catch (error) {
    // Handle error
  }
}
```

> If the app goes to the background this synchronization will fail. To keep syncing until it finishes
> (summaries **and** events), use background sync — see `references/background.md`.

Sync all three summaries for **a specific date** with `sync(date:)`:

```dart
void syncSummaries() async {
  try {
    await HCRookSyncManager.sync(date: date);

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
    await HCRookSyncManager.sync(date: date, summary: summarySyncType);

    // Summary synced successfully
  } catch (error) {
    // Handle error
  }
}
```

## Sync events

Sync a chosen event type for a date with `syncEvents(date, event)`:

```dart
void syncSingleEvent() async {
  try {
    await HCRookSyncManager.syncEvents(date, eventSyncType);

    // Event synced successfully
  } catch (error) {
    // Handle error
  }
}
```

## Sync current-day events

These upload a small amount of current-day data **and** return the uploaded data, so you can update your UI
immediately without waiting for the webhook.

> **Each of these contributes to the Health Connect rate limit — don't call them too frequently.**

```dart
// Steps
void syncStepsEvents() async {
  try {
    final steps = await HCRookSyncManager.getTodayStepsCount();

    // Update UI
  } catch (error) {
    // Handle error
  }
}

// Calories
void syncCaloriesEvents() async {
  try {
    final calories = await HCRookSyncManager.getTodayCaloriesCount();

    // Update UI
  } catch (error) {
    // Handle error
  }
}

// Heart rate
void syncHeartRateEvents() async {
  try {
    final heartRate = await HCRookSyncManager.getTodayHeartRate();

    // Update UI
  } catch (error) {
    // Handle error
  }
}
```

## Sync and display (get a copy of what was uploaded)

To sync **and** receive a copy of the uploaded summary/event, use the `get*` functions below. Each syncs the
data for the given `date` and returns it so you can update your UI right away:

| Function                   | Returns               |
|----------------------------|-----------------------|
| `getSleepSummary(date)`    | `List<SleepSummary>`  |
| `getPhysicalSummary(date)` | `PhysicalSummary`     |
| `getBodySummary(date)`     | `BodySummary`         |
| `getActivityEvents(date)`  | `List<ActivityEvent>` |

```dart
void example() async {
  final date = DateTime.now();

  try {
    final sleepSummaries = await HCRookSyncManager.getSleepSummary(date);

    // Update UI
  } catch (error) {
    // Handle error
  }
}
```

`getPhysicalSummary`, `getBodySummary`, and `getActivityEvents` follow the same shape, differing only in the
returned type from the table above.

> The returned classes (`SleepSummary`, `PhysicalSummary`, `BodySummary`, `ActivityEvent`) are large. Fields
> that Health Connect doesn't support (or that this SDK doesn't process) are always `null`. For the full
> field-by-field definitions, see the docs:
> <https://docs.tryrook.io/docs/rookconnect/sdk/flutter/health-connect/fundamentals/usage-sync-healthdata-manually/#sync-and-display>
> For the complete API reference, see pub.dev:
> <https://pub.dev/documentation/rook_sdk_health_connect/latest/rook_sdk_health_connect/>
