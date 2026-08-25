# Sync health data (automatic) — Flutter · Samsung Health

Prerequisite: the SDK is initialized and a `user_id` is registered (see `references/setup-and-init.md`).
Background Sync also requires the permissions listed below. All calls are **static** on `RookSamsung` and
return a `Future` — use `await` inside `try/catch`. Samsung Health is Android-only, so gate background
sync with `Platform.isAndroid`.

> **This is the recommended way to sync.** Background Sync is faster and easier to integrate than a manual
> process and handles everything **except permissions and `user_id`** for you. Prefer it before building a
> custom manual sync (`references/sync.md`).

## Background Sync

**Background Sync** schedules an automatic sync **every hour** in the background.

### What each run syncs

Every time it triggers, Background Sync performs, in order:

1. **Today's events** — Activity, Steps, Calories, Body Metrics, Heart Rate, Oxygenation, Temperature,
   Hydration, Nutrition, Blood Pressure, Blood Glucose.
2. **Today's summaries** — Sleep.
3. **Yesterday's summaries** — Sleep, Physical, Body.
4. **Historic data** — Activity events, Sleep / Physical / Body summaries (up to 29 days back, starting
   yesterday, for anything not yet synced).

Summaries in steps 2–3 are only re-synced **if at least 4 hours have passed** since the last successful
sync **and** the new data differs from what was previously synced.

> Ask users to enable **Continuous HR measurement** on their Galaxy Watch to improve the accuracy of the
> readings.

### Permissions

Background Sync needs:

- **Samsung Health permissions** — required. See `references/permissions.md`.
- **Alarm permission** (`SCHEDULE_EXACT_ALARM`) — **optional**; it improves the service's lifetime in
  battery-constrained scenarios. Background Sync works without it and only schedules an alarm if it's
  granted. See the optimization permissions in `references/permissions.md`.

> **Call `enableBackground` again after the user grants the alarm permission** — otherwise the alarm won't
> be scheduled.

> Background Sync also requires a `user_id`. The first run may do nothing: only once a `user_id` is
> configured **and** the required permissions are granted will a sync happen within the following hour.

### Enable

`enableBackground({required bool enableNativeLogs, bool cancelAndReschedule = false})` schedules the hourly
sync:

- `enableNativeLogs` — enable logcat logging.
- `cancelAndReschedule` — if `true`, cancels any scheduled or running sync and schedules a new one
  (defaults to `false`).

```dart
void enableBackgroundSync() async {
  try {
    await RookSamsung.enableBackground(enableNativeLogs: kDebugMode);

    // Background sync enabled
  } catch (error) {
    // Handle error
  }
}
```

Also call `enableBackground` from `main`, gated on the user's stored preference. Guard it with a platform
check so it only runs on Android:

```dart
void main() {
  // Ensure the plugin is ready
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
      await RookSamsung.enableBackground(enableNativeLogs: kDebugMode);
    }
  } catch (error) {
    // Log
  }
}
```

> Setting `enableBackgroundSync: true` in `RookConfiguration` at init time also starts Background Sync (see
> `references/setup-and-init.md`). The recommended pattern is to leave it `false` and call
> `enableBackground` explicitly once the user opts in.

### Disable

```dart
void disableBackgroundSync() async {
  try {
    await RookSamsung.disableBackground();

    // Background sync disabled
  } catch (error) {
    // Handle error
  }
}
```

### Check state (UI only)

Use `isScheduled` (one-shot, `Future<bool>`) or the `isScheduledUpdates` stream (`Stream<bool>`,
experimental, real-time) to reflect state in your UI:

```dart
// Latest state
void checkBackgroundSyncState() async {
  try {
    final isScheduled = await RookSamsung.isScheduled();

    // isScheduled is a bool — update your UI
  } catch (error) {
    // Handle error
  }
}
```

```dart
// Real-time updates
StreamSubscription<bool>? streamSubscription;

void listenBackgroundSyncState() {
  streamSubscription = RookSamsung.isScheduledUpdates.listen((isScheduled) {
    // Update UI
  });
}

// Stop listening when done (e.g. in dispose)
void dispose() {
  streamSubscription?.cancel();
}
```

> **Don't** use `isScheduled` / `isScheduledUpdates` to decide whether to call `enableBackground`. Doing so
> can block future improvements to Background Sync behavior from being applied. These are for UI purposes
> only — always call `enableBackground` (it's safe to call again).

### Execution / skip conditions

Each periodic run checks its requirements and **skips** if any of these is true:

- The device has no internet connection.
- The `user_id` hasn't been configured.
- The user hasn't granted Samsung Health permissions.
- There was an error initializing the SDK.
