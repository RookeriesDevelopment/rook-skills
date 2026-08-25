---
name: rook-flutter-apple-health
author: ROOK
description: >
  Integrate the ROOK Flutter Apple Health SDK: set up credentials, register users, request
  permissions, and sync Apple Health (HealthKit) summaries and events (steps, calories, sleep,
  heart rate). Use when a developer is adding ROOK, Apple Health, HealthKit, or wearable/health-data
  sync to a Flutter iOS app.
---

# ROOK Flutter — Apple Health SDK

## What this skill does

Guides a developer through integrating the ROOK Apple Health SDK on Flutter (iOS): SDK setup, user
registration, availability and permissions, and syncing health data (summaries and events) both
manually and automatically in the background.

## When to use it

Use when the developer mentions ROOK, Apple Health, HealthKit, health-data sync, wearables,
permissions, or background sync in a Flutter iOS project.

## Invariants — do not get these wrong

- **Platform:** this SDK is **iOS only**. Apple Health / HealthKit is not available on Android;
  gate any calls behind `Platform.isIOS`.
- **Packages:** two packages work together — `rook_sdk_apple_health` (the data source) and
  `rook_sdk_core` (shared types like `RookConfiguration` and `RookEnvironment`).
- **`AH` prefix:** every Apple Health manager class is prefixed `AH`
  (`AHRookConfigurationManager`, `AHRookHealthPermissionsManager`, `AHRookBackgroundSync`).
- **Identifiers:** use `client_uuid` (never `customer_id`) and `user_id`.
- **Environment:** the SDK environment is `RookEnvironment.sandbox` or `RookEnvironment.production`;
  each has its **own** credentials in the ROOK Portal. Typically SANDBOX for debug builds,
  PRODUCTION for release.
- **Secrets:** never hardcode or print real credentials — use placeholders `CLIENT_UUID` / `SECRET`.
- **Versions:** the SDK versions and Xcode/Dart/Flutter constraints live in
  `references/setup-and-init.md`; keep them in sync with the official docs — never invent a version.

## Integration flow

Open the reference for the step you're on:

1. **Setup & initialization** (install, HealthKit config, choose environment, init) → `references/setup-and-init.md`
2. **Register / update the user** (`user_id`) → `references/setup-and-init.md`
3. **Availability & permissions** → `references/permissions.md`
4. **Sync health data** (manual) → `references/sync.md`
5. **Sync health data** (automatic / background) → `references/background.md`
6. **Troubleshooting** (known exceptions + best practices) → `references/troubleshooting.md`
