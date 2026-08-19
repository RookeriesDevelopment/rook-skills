---
name: rook-android-health-connect
author: ROOK
description: >
  Integrate the ROOK Android Health Connect SDK: set up credentials, register users, request
  permissions, and sync health summaries and events (steps, calories, sleep, heart rate) manually or
  in the background. Use when a developer is adding ROOK, Health Connect, or wearable/health-data sync
  to a native Android app.
---

# ROOK Android — Health Connect SDK

<!-- SKILL.md is the router. Keep it short. Put the detail in references/ so agents load only what they need. -->

## What this skill does

Guides a developer through integrating the ROOK Health Connect SDK (`com.rookmotion.android:rook-sdk`)
into a native Android app: SDK setup and initialization, user registration, checking Health Connect
availability, requesting permissions, and syncing health data (summaries and events) both manually and
automatically in the background.

## When to use it

Use when the developer mentions ROOK, Health Connect, health-data sync, wearables, permissions, or
background sync in a native Android (Kotlin) project.

## Invariants — do not get these wrong

- **Identifiers:** use `client_uuid` (the client identifier — **never** `customer_id`) and `user_id`
  (the app end-user whose data is synced).
- **Environment:** the SDK environment is `RookEnvironment.SANDBOX` or `RookEnvironment.PRODUCTION`;
  each has its **own** credentials (package name + secret) registered in the ROOK Portal. Typically
  SANDBOX for debug builds, PRODUCTION for release.
- **Secrets:** never hardcode or print a real `client_uuid` or secret — use placeholders
  `CLIENT_UUID` / `SECRET`.
- **Credentials must be registered first:** the app's `applicationId` (package name) and its secret must
  be registered in the ROOK Portal before `initRook()`, or initialization fails with
  `HCNotAuthorizedException`.
- **Versions:** `minSdk`, `targetSdk`, and the `rook-sdk` dependency coordinate/version live in
  `references/setup-and-init.md` — keep them in sync with the official docs; never invent a version.
- **Initialize once:** call `initRook()` only once per app launch.

## Key managers

- `RookConfigurationManager` — configuration, initialization, `updateUserID`, login/logout.
- `RookPermissionsManager` — Health Connect availability and permission requests.
- `RookSyncManager` — manual sync of summaries and events.
- `RookBackgroundSyncManager` — automatic/background sync.

## Integration flow

Open the reference for the step you're on:

1. **Setup & initialization** (install, configure, choose environment, `initRook`) → `references/setup-and-init.md`
2. **Register / update the user** (`updateUserID`, login/logout) → `references/setup-and-init.md`
3. **Availability & permissions** (check Health Connect, request permissions) → `references/permissions.md`
4. **Sync health data — manual** → `references/sync.md`
5. **Sync health data — automatic / background** → `references/background.md`
6. **Troubleshooting** (known exceptions + best practices) → `references/troubleshooting.md`
