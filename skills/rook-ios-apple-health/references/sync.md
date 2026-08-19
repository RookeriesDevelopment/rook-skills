# Sync health data — iOS · Apple Health

This reference covers one-off, manual synchronization through `RookSummaryManager` and
`RookEventsManager`. It intentionally excludes automatic, foreground, and background sync.

## Contents

- [Before syncing](#before-syncing)
- [Sync summaries](#sync-summaries)
- [Sync events](#sync-events)
- [Completion-handler APIs](#completion-handler-apis)
- [Handle results and errors](#handle-results-and-errors)

## Manual sync

Create manager instances where they can be reused. Do not create a new manager for every individual
data type.

```swift
import RookSDK

let summaryManager = RookSummaryManager()
let eventsManager = RookEventsManager()
```

## Before syncing

Complete these steps before calling either manager:

1. Initialize the SDK successfully.
2. Register a stable `user_id` with `updateUserId`.
3. Confirm that HealthKit is available.
4. Request the read permissions required by the summaries or events being synchronized.
5. Ensure that the device has network access so ROOK can upload the extracted Apple Health data.

See [Setup and initialization](setup-and-init.md) and [Availability and permissions](permissions.md)
for the prerequisite flows.

## Sync summaries

Use `RookSummaryManager` for daily sleep, physical, and body summaries.

### Summary types

`SummaryTypeToUpload` supports:

- `.sleep`
- `.physical`
- `.body`

### Methods

| Async method | Behavior |
| --- | --- |
| `sync() async throws -> Bool` | Upload missing sleep, physical, and body summaries from the previous 30 days. |
| `sync(_ date: Date) async throws -> Bool` | Extract and upload all three summary types for one date. |
| `sync(_ date: Date, _ summaryType: [SummaryTypeToUpload]) async throws -> Bool` | Extract and upload only the selected summary types for one date. |
| `getSleepSummary(date:) async throws -> [RookSleepSummary]` | Upload and return the sleep summaries for one date. More than one sleep session may be returned. |
| `getPhysicalSummary(date:) async throws -> RookPhysicalSummary` | Upload and return the physical summary for one date. |
| `getBodySummary(date:) async throws -> RookBodySummary` | Upload and return the body summary for one date. |
| `syncPendingSummaries() async throws -> Bool` | Retry summaries that previously failed to upload. This async overload is deprecated; prefer `sync()` or `sync(_:)`. |

Use `sync(_:)` when all summary types are needed, or pass `SummaryTypeToUpload` values to avoid
extracting data the product does not use.

```swift
func syncSummaries(for date: Date) async {
  do {
    let uploaded = try await summaryManager.sync(
      date,
      [.sleep, .physical, .body]
    )
    debugPrint("Summaries uploaded: \(uploaded)")
  } catch {
    debugPrint("Summary sync failed: \(error.localizedDescription)")
  }
}
```

Use the returning methods when the app needs the normalized result locally as well as uploading it:

```swift
let sleepSummaries = try await summaryManager.getSleepSummary(date: date)
let physicalSummary = try await summaryManager.getPhysicalSummary(date: date)
let bodySummary = try await summaryManager.getBodySummary(date: date)
```

## Sync events

Use `RookEventsManager` for individual or granular health events.

### Event types

`EventTypeToUpload` supports:

| Event type | Data synchronized |
| --- | --- |
| `.activityEvent` | Workouts and activity events |
| `.heartRate` | Heart-rate events |
| `.oxygenation` | Oxygen-saturation events |
| `.temperature` | Temperature events |
| `.bloodPressure` | Blood-pressure events |
| `.bloodGlucose` | Blood-glucose events |
| `.bodyMetrics` | Body-measurement events |
| `.nutrition` | Nutrition events already stored in HealthKit |
| `.hydration` | Hydration events |
| `.calories` | The current day's calorie event |
| `.steps` | The current day's step event |
| `.ecg` | Electrocardiogram events |

The `date` passed to `syncEvents(date:eventType:)` selects the day for historical event types. For
`.steps` and `.calories`, the SDK uses the current day regardless of the supplied date.

### Methods

| Method | Behavior |
| --- | --- |
| `syncEvents() async` | Upload missing events from the previous 30 days. This overload completes without returning a success value. |
| `syncEvents(date:eventType:) async throws -> Bool` | Upload one `EventTypeToUpload` for the selected date. |
| `getActivityEvents(date:) async throws -> [RookActivityEvent]` | Upload and return activity events for one date. |
| `getTodayStepCount() async throws -> Int` | Upload today's step event and return the total step count. |
| `getTodayStepCountBySource() async throws -> StepEvent` | Upload today's step event and return steps grouped by source. |
| `getTodayCalories() async throws -> RookCalories` | Upload today's calorie event and return active, basal, and total calorie data. |
| `getTodayHeartRate() async throws -> RookHeartRate` | Upload and return today's heart-rate data. |
| `writeNutritionEvent(event:) async throws -> Bool` | Write a nutrition event to HealthKit and upload it to ROOK. |
| `syncPendingEvents(completion:)` | Retry events that previously failed to upload; this method is completion-handler only. |

Use `syncEvents(date:eventType:)` for a targeted event sync:

```swift
func syncHeartRateEvents(for date: Date) async {
  do {
    let uploaded = try await eventsManager.syncEvents(
      date: date,
      eventType: .heartRate
    )
    debugPrint("Heart-rate events uploaded: \(uploaded)")
  } catch {
    debugPrint("Heart-rate sync failed: \(error.localizedDescription)")
  }
}
```

Use a returning helper when the app also needs to display the uploaded data:

```swift
let activities = try await eventsManager.getActivityEvents(date: date)
let steps = try await eventsManager.getTodayStepCount()
let calories = try await eventsManager.getTodayCalories()
let heartRate = try await eventsManager.getTodayHeartRate()
```

`writeNutritionEvent(event:)` is different from `.nutrition`: the former creates a new nutrition
sample in HealthKit and uploads it, while `.nutrition` synchronizes nutrition samples already present
for the selected date. Call `requestDietaryWritePermissions` before writing a nutrition event.

## Completion-handler APIs

The managers also provide completion-handler variants. Use them when the surrounding code does not use
Swift concurrency.

```swift
summaryManager.sync(date, summaryType: [.sleep, .physical]) { result in
  switch result {
  case .success(let uploaded):
    debugPrint("Summaries uploaded: \(uploaded)")
  case .failure(let error):
    debugPrint("Summary sync failed: \(error.localizedDescription)")
  }
}

eventsManager.syncEvents(date: date, eventType: .activityEvent) { result in
  switch result {
  case .success(let uploaded):
    debugPrint("Activity events uploaded: \(uploaded)")
  case .failure(let error):
    debugPrint("Activity sync failed: \(error.localizedDescription)")
  }
}
```

The completion-handler forms of the returning helpers use `Result<ReturnType, Error>`, following the
same pattern.

## Handle results and errors

- Treat the returned `Bool` as the result of the extraction-and-upload operation, not as proof that
  HealthKit contains samples for every requested type.
- Handle missing permissions, an unregistered user, empty HealthKit data, a locked device, and network
  failures as distinct outcomes in the product experience.
- Do not interpret empty data as a confirmed permission denial; Apple does not reveal read-denial status.
- Await one manual sync before starting another overlapping sync for the same date and data type.
- Remember that Apple Health is not real time. A successful sync only includes data that providers have
  already written to HealthKit.
