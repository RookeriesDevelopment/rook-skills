# Background steps (optional) — Flutter · Health Connect

> **Optional and not recommended by default.** Only implement this if the developer specifically asks for
> an Android step tracker, or needs a fallback for devices that don't support Health Connect features (e.g.
> background read). For everything else, use Health Connect sync (`references/background.md` /
> `references/sync.md`).
>
> **What it is:** a step tracker built on the Android step-counter sensor — it works **without Health
> Connect installed**, but the **only** data it produces is a **step count uploaded roughly once per
> hour**, and sensor-based accuracy is lower than Health Connect / provider data. Treat Android Steps and
> Health Connect as **separate data sources** (see the best-practices note in `references/troubleshooting.md`).

Prerequisite: the SDK is initialized and a `user_id` is registered (see `references/setup-and-init.md`).
Android only — guard calls with `Platform.isAndroid`.

The background-steps API is exposed through the static `AndroidStepsCounter`.

## Check availability

Confirm the device has the required sensor before offering this feature:

```dart
void isAvailable() async {
  final isAvailable = await AndroidStepsCounter.isStepsCounterAvailable();
}
```

## Permissions

`AndroidStepsCounter` needs:

- **Android permissions** — required. See `references/permissions.md`.
- **Alarm permission** (`SCHEDULE_EXACT_ALARM`) — **optional**; it improves the service's lifetime in
  battery-constrained scenarios. The counter works without it and only schedules an alarm if it's granted.

> **Call `enableStepsCounter` again after the user grants the alarm permission** — otherwise the alarm
> won't be scheduled.

## Customizing the foreground-service notification

The counter runs a foreground service, which requires a permanently displayed notification. Defaults: a
walk icon, title "Steps service", content "Tracking your steps…". To use your own resources, reference them
via manifest `meta-data`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>
        <meta-data
            android:name="io.tryrook.service.notification.STEPS_ICON"
            android:resource="@drawable/my_custom_icon" />
        <meta-data
            android:name="io.tryrook.service.notification.STEPS_TITLE"
            android:resource="@string/my_custom_title" />
        <meta-data
            android:name="io.tryrook.service.notification.STEPS_CONTENT"
            android:resource="@string/my_custom_content" />
    </application>
</manifest>
```

> On Android 13 (SDK 33)+ the user can dismiss this notification without stopping the service; the service
> then appears in the active-apps section (varies by device brand).

## Enable / disable

Start tracking with `enableStepsCounter`, stop with `disableStepsCounter`. Calling enable/disable when the
service is already active/inactive does nothing; confirm the actual state with `isStepsCounterActive`.

```dart
void startStepsTracker() async {
  try {
    await AndroidStepsCounter.enableStepsCounter();

    // Success
  } catch (exception) {
    // Handle error
  }
}

void stopStepsTracker() async {
  try {
    await AndroidStepsCounter.disableStepsCounter();

    // Success
  } catch (exception) {
    // Handle error
  }
}
```

## Sync today's step count

`getTodayStepsCount` retrieves and uploads the current day's step count:

```dart
void getTodaySteps() {
  AndroidStepsCounter.getTodayStepsCount().then((todaySteps) {
    // Success
  }).catchError((exception) {
    // Handle error
  });
}
```

> **Resource-intensive — don't call it too frequently**, or it can degrade the user experience.

## Considerations

- **Auto start after reboot:** once enabled, the foreground service restarts after the user first unlocks
  the device following a restart (varies by brand). `disableStepsCounter` stops this behavior.
- **Force close:** if the user force-closes the app from settings and then restarts the device, the service
  may not be able to restart.
- **Hourly upload isn't exact:** steps upload every hour from when `enableStepsCounter` was called, but the
  exact timing depends on how Android manages device resources.
