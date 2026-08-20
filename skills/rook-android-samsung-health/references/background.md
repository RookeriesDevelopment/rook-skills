# Sync health data (automatic) — Android · Samsung Health

Prerequisite: the SDK is initialized and a `user_id` is registered (see `references/setup-and-init.md`).
Background Sync also requires the permissions listed below.

> **This is the recommended way to sync.** Background Sync is faster and easier to integrate than a manual
> process and handles everything **except permissions and `user_id`** for you. Prefer it before building a
> custom manual sync (`references/sync.md`).

## Background Sync

**Background Sync** schedules an automatic sync **every hour** in the background. All functions exist on
both the `RookSamsung` instance (recommended as a singleton via DI / ServiceLocator) and the
`RookSamsungObject` singleton (pass a `Context` on each call). The snippets use an instance named
`rookSamsung`.

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
  granted.

> **Call `schedule` again after the user grants the alarm permission** — otherwise the alarm won't be
> scheduled.

> Background Sync also requires a `user_id`. The first run may do nothing: only once a `user_id` is
> configured **and** the required permissions are granted will a sync happen within the following hour.

### Schedule

`schedule` sets up the hourly sync:

- `enableLogs` — enable logcat logging.
- `cancelAndReschedule` — if `true`, cancels any scheduled or running sync and schedules a new one.

```kotlin
rookSamsung.schedule(enableLogs = isDebug)
```

Also call `schedule` in your `Application.onCreate`. It's good practice to gate it on a user preference:

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // Ask users whether they want background sync, persist their choice,
        // and only schedule if they opted in.
        if (userAllowedBackgroundSync) {
            RookSamsungObject.schedule(this, enableLogs = isDebug)
        }
    }
}
```

### Cancel

```kotlin
rookSamsung.cancel()
```

### Check state (UI only)

Use `isScheduled` (one-shot) or `isScheduledFlow` (experimental, real-time) to reflect state in your UI:

```kotlin
// Latest state
coroutineScope.launch {
    val isScheduled: Boolean = rookSamsung.isScheduled().getOrDefault(defaultValue = false)
    // Update UI
}

// Real-time updates
@OptIn(ExperimentalRookSamsungApi::class)
viewModelScope.launch {
    rookSamsung.isScheduledFlow().collectLatest { isScheduled: Boolean ->
        // Update UI
    }
}
```

> **Don't** use `isScheduled` / `isScheduledFlow` to decide whether to call `schedule`. Doing so can block
> future improvements to Background Sync behavior from being applied. These are for UI purposes only —
> always call `schedule` (it's safe to call again).

### Progress updates (optional)

Only needed if you want to display Background Sync progress in your UI — the sync runs without it.

Register a `BroadcastReceiver` for `RookSamsungObject.ACTION_BACKGROUND_SYNC`. The intent always carries
`EXTRA_BACKGROUND_SYNC_PROGRESS` (convert with `SHBackgroundSyncProgress.fromId(...)`); on `FAILED`, it also
carries `EXTRA_BACKGROUND_SYNC_FAILURE_REASON` with the error message.

```kotlin
// 1. Create the broadcast receiver
private val broadcastReceiver = object : BroadcastReceiver() {
    override fun onReceive(intentContext: Context?, intent: Intent?) {
        val progressId = intent?.getIntExtra(
            RookSamsungObject.EXTRA_BACKGROUND_SYNC_PROGRESS, -1,
        ) ?: -1

        val progress = SHBackgroundSyncProgress.fromId(progressId)

        if (progress == SHBackgroundSyncProgress.FAILED) {
            val message = intent?.getStringExtra(
                RookSamsungObject.EXTRA_BACKGROUND_SYNC_FAILURE_REASON,
            )
        }

        // Update your UI
    }
}

// 2. Register it (e.g. in onCreate)
ContextCompat.registerReceiver(
    context,
    broadcastReceiver,
    IntentFilter(RookSamsungObject.ACTION_BACKGROUND_SYNC),
    ContextCompat.RECEIVER_EXPORTED,
)

// 3. Unregister it (e.g. in onDestroy)
context.unregisterReceiver(broadcastReceiver)
```

A typical progress flow:

1. `STARTED` is broadcast.
2. Sync runs in four stages: `SYNCING_TODAY_EVENTS`, `SYNCING_TODAY_SUMMARIES`,
   `SYNCING_YESTERDAY_SUMMARIES`, `SYNCING_HISTORIC_SUMMARIES_AND_ACTIVITY`.
   - If an unrecoverable error occurs during any stage, `FAILED` is broadcast.
3. `FINISHED` is broadcast on success.

### Execution / skip conditions

Each periodic run checks its requirements and **skips** if any of these is true:

- The device has no internet connection.
- The `user_id` hasn't been configured.
- The user hasn't granted Samsung Health permissions.
- There was an error initializing the SDK.
