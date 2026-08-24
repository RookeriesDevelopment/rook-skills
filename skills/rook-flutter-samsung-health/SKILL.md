---
name: rook-flutter-samsung-health
author: ROOK
description: >
  Integrate the ROOK Flutter Samsung Health SDK: set up credentials, register users, request
  permissions, and sync Samsung Health summaries and events (steps, calories, sleep, heart rate)
  manually or in the background. Use when a developer is adding ROOK, Samsung Health, or
  wearable/health-data sync to a Flutter app.
---

# ROOK Flutter — Samsung Health SDK

## What this skill does

Guides a developer through integrating the ROOK Samsung Health SDK (`rook_sdk_samsung_health`, plus its
required `rook_sdk_core` dependency) into a Flutter app: SDK setup and initialization, user
registration, checking Samsung Health availability, requesting permissions, and syncing health data
(summaries and events) both manually and automatically in the background.

## When to use it

Use when the developer mentions ROOK, Samsung Health, health-data sync, wearables, permissions, or
background sync in a Flutter project targeting Android.

## Invariants — do not get these wrong

- **Identifiers:** use `client_uuid` (the client identifier — **never** `customer_id`) and `user_id`
  (the app end-user whose data is synced).
- **Environment:** the SDK environment is `RookEnvironment.sandbox` or `RookEnvironment.production`; each
  has its **own** credentials (package name + secret) registered in the ROOK Portal. Typically sandbox
  for debug builds, production for release — gate it with `kDebugMode`.
- **Secrets:** never hardcode or print a real `client_uuid` or secret — use placeholders
  `CLIENT_UUID` / `SECRET`.
- **Credentials must be registered first:** the app's `applicationId` (package name) and its secret must
  be registered in the ROOK Portal before `initRook`, or initialization fails with
  `SDKNotAuthorizedException`.
- **Device restrictions:** the Samsung Health SDK requires the Samsung Health app v6.29 or later, runs on
  Android 10 (API level 29) or above, and **does not support emulators** — test on a physical device.
  Enable Samsung Health developer mode on the test device.
- **Android-only:** this SDK integrates Samsung Health, which exists only on Android. Guard any calls
  with a platform check (`Platform.isAndroid`) in cross-platform apps.
- **Versions:** `minSdk`, `targetSdk`, the `rook_sdk_samsung_health` / `rook_sdk_core` package versions,
  and the bundled Samsung `.aar` live in `references/setup-and-init.md` — keep them in sync with the
  official docs; never invent a version.
- **Initialize once:** call `initRook` only once per app launch.

## Key API surface

The Flutter SDK exposes a single static entry-point class:

- `RookSamsung` — all operations are **static** calls (`RookSamsung.initRook`,
  `RookSamsung.updateUserID`, `RookSamsung.checkSamsungHealthAvailability`,
  `RookSamsung.requestSamsungHealthPermissions`, `RookSamsung.sync`, `RookSamsung.enableBackground`, …).
  There is no instance to construct and no context to pass — call the methods directly.
- `RookConfiguration` — the config object (`clientUUID`, `secret`, `environment`,
  `enableBackgroundSync`) passed to `initRook`.

Most methods return a `Future`; handle both success and error paths (`.then(...).catchError(...)` or
`try/await`).

## Integration flow

Open the reference for the step you're on:

1. **Setup & initialization** (install, configure, choose environment, `initRook`) → `references/setup-and-init.md`
2. **Register / update the user** (`updateUserID`, login/logout) → `references/setup-and-init.md`
3. **Availability & permissions** (check Samsung Health, request permissions) → `references/permissions.md`
4. **Sync health data — manual** → `references/sync.md`
5. **Sync health data — automatic / background** → `references/background.md`
6. **Troubleshooting** (known exceptions + best practices) → `references/troubleshooting.md`
