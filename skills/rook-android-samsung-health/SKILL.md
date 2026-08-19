---
name: rook-android-samsung-health
author: ROOK
description: >
  Integrate the ROOK Android Samsung Health SDK: set up credentials, register users, request
  permissions, and sync Samsung Health summaries and events (steps, calories, sleep, heart rate)
  manually or in the background. Use when a developer is adding ROOK, Samsung Health, or
  wearable/health-data sync to a native Android app.
---

# ROOK Android — Samsung Health SDK

## What this skill does

Guides a developer through integrating the ROOK Samsung Health SDK (`io.tryrook.android:rook-sdk-samsung`)
into a native Android (Kotlin) app: SDK setup and initialization, user registration, checking Samsung
Health availability, requesting permissions, and syncing health data (summaries and events) both
manually and automatically in the background.

## When to use it

Use when the developer mentions ROOK, Samsung Health, health-data sync, wearables, permissions, or
background sync in a native Android (Kotlin) project.

## Invariants — do not get these wrong

- **Identifiers:** use `client_uuid` (the client identifier — **never** `customer_id`) and `user_id`
  (the app end-user whose data is synced).
- **Environment:** the SDK environment is `SHEnvironment.SANDBOX` or `SHEnvironment.PRODUCTION`; each has
  its **own** credentials (package name + secret) registered in the ROOK Portal. Typically SANDBOX for
  debug builds, PRODUCTION for release.
- **Secrets:** never hardcode or print a real `client_uuid` or secret — use placeholders
  `CLIENT_UUID` / `SECRET`.
- **Credentials must be registered first:** the app's `applicationId` (package name) and its secret must
  be registered in the ROOK Portal before `initRook()`, or initialization fails with
  `SHNotAuthorizedException`.
- **Device restrictions:** Samsung Health SDK requires the Samsung Health app v6.29 or later, runs on
  Android 10 (API level 29) or above, and **does not support emulators** — test on a physical device.
  Enable Samsung Health developer mode on the test device.
- **Versions:** `minSdk`, `targetSdk`, and the `rook-sdk-samsung` dependency coordinate/version (plus the
  bundled Samsung `.aar`) live in `references/setup-and-init.md` — keep them in sync with the official
  docs; never invent a version.
- **Initialize once:** call `initRook()` only once per app launch.

## Key API surface

Samsung Health exposes a single entry-point class (unlike the multi-manager Health Connect SDK):

- `RookSamsung` — instance-based entry point (built with an `applicationContext`). Handles
  configuration, `initRook`, `updateUserID`, availability, permissions, manual sync, and background sync.
- `RookSamsungObject` — a singleton alternative to `RookSamsung` that needs no instance, but requires a
  `Context` on every call. Use it in places where you don't hold an instance (e.g. the `Application`
  class or a `BroadcastReceiver`).

## Integration flow

Open the reference for the step you're on:

1. **Setup & initialization** (install, configure, choose environment, `initRook`) → `references/setup-and-init.md`
2. **Register / update the user** (`updateUserID`, login/logout) → `references/setup-and-init.md`
3. **Availability & permissions** (check Samsung Health, request permissions) → `references/permissions.md`
4. **Sync health data — manual** → `references/sync.md`
5. **Sync health data — automatic / background** → `references/background.md`
6. **Troubleshooting** (known exceptions + best practices) → `references/troubleshooting.md`
