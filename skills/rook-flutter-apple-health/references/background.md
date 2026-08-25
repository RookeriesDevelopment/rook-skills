# Sync health data (automatic) — Flutter · Apple Health

Prerequisite: the SDK is initialized, a `user_id` is registered, and Apple Health permissions are
requested (see `references/setup-and-init.md` and `references/permissions.md`). Apple Health is
iOS-only, so gate every call with `Platform.isIOS`.

> **This is the recommended way to sync.** Automatic sync is faster and easier to integrate than a
> manual process and handles everything **except permissions and `user_id`** for you. Prefer it before
> building a custom manual sync (`references/sync.md`).

There are two independent components you can enable — **combine both** for the best experience:

- **Background Sync** (`AHRookBackgroundSync`) — syncs roughly every hour while the app is closed / in
  the background, up to **14 days into the past**.
- **Continuous Upload** (`AHRookContinuousUpload`) — syncs every time the user opens the app.

> The Xcode **HealthKit → Background delivery** and **Background Modes → Background fetch** capabilities
> must be enabled for background sync to run — see `references/setup-and-init.md`.

## Background Sync

`enableBackground({required bool enableNativeLogs})` → `Future<void>` schedules the background sync:

```dart
void enableBackgroundSync() async {
  try {
    await AHRookBackgroundSync.enableBackground(enableNativeLogs: kDebugMode);

    // Success — data will start syncing in the background
  } catch (error) {
    // Handle error
  }
}
```

Also call `enableBackground` from `main`, gated on the user's stored preference and a platform check so
it only runs on iOS:

```dart
void main() {
  // Ensure the plugin is ready
  WidgetsFlutterBinding.ensureInitialized();

  if (Platform.isIOS) {
    enableIOSBackgroundSync();
  }

  runApp(App());
}

void enableIOSBackgroundSync() async {
  try {
    // Ask users whether they want background sync, persist their choice,
    // and only enable it if they opted in.
    final userAllowedBackgroundSync = await AppPreferences().getUserAllowedBackgroundSync();

    if (userAllowedBackgroundSync) {
      await AHRookBackgroundSync.enableBackground(enableNativeLogs: kDebugMode);
    }
  } catch (error) {
    // Log
  }
}
```

> Setting `enableBackgroundSync: true` in `RookConfiguration` at init time also starts Background Sync
> (see `references/setup-and-init.md`). The recommended pattern is to leave it `false` and call
> `enableBackground` explicitly once the user opts in.

> **iOS locks HealthKit while the device is locked.** For security, iOS encrypts HealthKit storage when
> the user locks the device, so the app may be unable to read Apple Health data while running in the
> background. This is expected iOS behavior — see Apple's documentation.

### Disable / check state

```dart
// Disable background sync
await AHRookBackgroundSync.disableBackground();

// Check whether background sync is currently scheduled (Future<bool>) — for UI
final isScheduled = await AHRookBackgroundSync.isScheduled();
```

### Background errors

Errors raised by the Background Sync component are delivered through the
`AHRookHelpers.backgroundErrorsUpdates` stream (`Stream<Exception>`):

```dart
// 1. Create a stream subscription
StreamSubscription<Exception>? streamSubscription;

// 2. Listen to the stream
streamSubscription = AHRookHelpers.backgroundErrorsUpdates.listen((backgroundError) {
  // Process error
});

// 3. Stop listening when done (e.g. in dispose)
streamSubscription?.cancel();
```

## Continuous Upload

`enableContinuousUpload({required bool enableNativeLogs})` → `Future<void>` syncs health data every time
the user opens the app:

```dart
void enableContinuousUpload() async {
  try {
    await AHRookContinuousUpload.enableContinuousUpload(enableNativeLogs: kDebugMode);

    // Success — data syncs on each app open (try restarting the app if you don't see it)
  } catch (error) {
    // Handle error
  }
}
```

### Disable / check state

```dart
// Disable continuous upload
await AHRookContinuousUpload.disableContinuousUpload();

// Check whether continuous upload is enabled (Future<bool>) — for UI
final isEnabled = await AHRookContinuousUpload.isContinuousUploadEnabled();
```
