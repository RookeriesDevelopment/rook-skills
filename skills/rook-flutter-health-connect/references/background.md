# Sync health data (automatic) — Flutter · Health Connect

Prerequisite: the SDK is initialized and a `user_id` is registered (see `references/setup-and-init.md`).
Background Sync also requires the permissions listed below. Android only — guard calls with
`Platform.isAndroid`.

> **This is the recommended way to sync.** Background Sync is faster and easier to integrate than a manual
> process and handles everything **except permissions and `user_id`** for you. Prefer it before building a
> custom manual sync (`references/sync.md`).

## Background Sync

**Background Sync** schedules an automatic sync **every hour** in the background, via the static
`HCRookBackgroundSync`.

You can also start it at initialization by setting `enableBackgroundSync: true` in `RookConfiguration` (see
`references/setup-and-init.md`).

### What each run syncs

Every time it triggers, Background Sync performs, in order:

1. **Today's events** — Activity, Steps, Calories, Body Metrics, Heart Rate, Oxygenation, Temperature,
   Hydration, Nutrition, Blood Pressure, Blood Glucose.
2. **Today's summaries** — Sleep.
3. **Yesterday's summaries** — Sleep, Physical, Body.
4. **Historic data** — Activity events, Sleep / Physical / Body summaries (starting yesterday, for anything
   not yet synced). The range is **29 days by default** and configurable in the ROOK Portal up to 180 days
   (see [Historic data range](#historic-data-range)).

Summaries in steps 2–3 are only re-synced **if at least 4 hours have passed** since the last successful
sync **and** the new data differs from what was previously synced.

> Hydration and Nutrition are synced as a **single event with cumulative values**, not one event per hour.

### Historic data range

The Background Sync historic range is configured in the **ROOK Portal** from 0 (off) to 180 days. When it is
set to **more than 29 days**, Background Sync needs the optional `READ_HEALTH_DATA_HISTORY` permission
(see `references/permissions.md#history-read-permissions`). If it isn't granted, Background Sync **silently
falls back to 29 days** — no error is raised. No code change is needed if the Portal is set to 29 days or
fewer.

### Request quota

Once started, Background Sync attempts to sync all historic data and stops when it finishes or when the
[Health Connect request quota](sync.md#request-quota-rate-limits) is exceeded. When the quota is reached,
all pending syncs are canceled and a recovery timestamp is created; pending syncs resume only at the next
scheduled run.

### Permissions

Background Sync needs:

- **Health Connect permissions** — required. See `references/permissions.md`.
- **Background read permission** — required. See `references/permissions.md`.
- **History read permission** (`READ_HEALTH_DATA_HISTORY`) — **optional**; only required if the ROOK
  Portal historic range is above 29 days. See `references/permissions.md`.
- **Alarm permission** (`SCHEDULE_EXACT_ALARM`) — **optional**; it improves the service's lifetime in
  battery-constrained scenarios. Background Sync works without it and only schedules an alarm if it's
  granted.

> **Call `enableBackground` again after the user grants the alarm permission** — otherwise the alarm won't
> be scheduled.

> Background Sync also requires a `user_id`. The first run may do nothing: only once a `user_id` is
> configured **and** the required permissions are granted will a sync happen within the following hour.

### Enable

`enableBackground` sets up the hourly sync:

- `enableNativeLogs` — enable logcat logging.
- `cancelAndReschedule` — if `true`, cancels any scheduled or currently running sync and schedules a new one.

```dart
void enableBackgroundSync() async {
  try {
    await HCRookBackgroundSync.enableBackground(enableNativeLogs: isDebug);

    // Background sync enabled
  } catch (error) {
    // Handle error
  }
}
```

Also call `enableBackground` in your `main` method. It's good practice to gate it on a user preference:

```dart
void main() {
  // Ensure that the plugin is ready
  WidgetsFlutterBinding.ensureInitialized();

  if (Platform.isAndroid) {
    enableAndroidBackgroundSync();
  }

  runApp(App());
}

void enableAndroidBackgroundSync() async {
  try {
    // Ask users whether they want background sync, persist their choice,
    // and only enable it if they opted in.
    final userAllowedBackgroundSync = await AppPreferences().getUserAllowedBackgroundSync();

    if (userAllowedBackgroundSync) {
      await HCRookBackgroundSync.enableBackground(enableNativeLogs: isDebug);
    }
  } catch (error) {
    // Log
  }
}
```

### Disable

```dart
void disableBackgroundSync() async {
  try {
    await HCRookBackgroundSync.disableBackground();

    // Background sync disabled
  } catch (error) {
    // Handle error
  }
}
```

### Check state (UI only)

Use `isScheduled` (one-shot) or `isScheduledUpdates` (experimental, real-time stream) to reflect state in
your UI:

```dart
// Latest state
void checkBackgroundSyncState() async {
  try {
    final isScheduled = await HCRookBackgroundSync.isScheduled();

    // Update your UI
  } catch (error) {
    // Handle error
  }
}
```

```dart
// Real-time updates
// 1. Create a stream subscription
StreamSubscription<bool>? streamSubscription;

// 2. Listen to the stream
streamSubscription = HCRookBackgroundSync.isScheduledUpdates.listen((isScheduled) {
  // Update UI
});

// 3. Stop listening when you no longer need updates (e.g. on dispose)
streamSubscription?.cancel();
```

> **Don't** use `isScheduled` / `isScheduledUpdates` to decide whether to call `enableBackground`. Doing so
> can block future improvements to Background Sync behavior from being applied. These are for UI purposes
> only — always call `enableBackground` (it's safe to call again).

### Execution / skip conditions

Each periodic run checks its requirements and **skips** if any of these is true:

- The device has no internet connection.
- The `user_id` hasn't been configured.
- The most recent request exceeded the Health Connect request quota.
- Health Connect permissions haven't been granted.
- Background read permission hasn't been granted.
- There was an error initializing the SDK.
