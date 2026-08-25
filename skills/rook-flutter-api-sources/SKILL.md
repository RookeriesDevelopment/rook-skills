---
name: rook-flutter-api-sources
author: ROOK
description: >
  Integrate the ROOK Flutter API Sources SDK (rook_sdk_core): set up credentials, initialize the SDK,
  and connect users to API-based data sources (Garmin, Oura, Polar, Fitbit, Withings, Whoop, Dexcom)
  through their OAuth authorization flows, list connected sources, and revoke access. Works on both
  Android and iOS. Use when a developer is adding ROOK, API/third-party wearable connections, or
  non-native health-data sources to a Flutter app.
---

# ROOK Flutter — API Sources SDK

## What this skill does

Guides a developer through integrating the ROOK API Sources SDK on Flutter (`rook_sdk_core`): SDK setup
and initialization, then connecting a user to API-based data sources (Garmin, Oura, Polar, Fitbit,
Withings, Whoop, Dexcom) via their OAuth authorization URLs, listing the sources a user has connected,
and revoking (disconnecting) a source. The same package targets **both Android and iOS**.

## When to use it

Use when the developer mentions ROOK, API Sources, connecting third-party data sources (Garmin, Oura,
Polar, Fitbit, Withings, Whoop, Dexcom), OAuth connection flows, or non-native health-data sources in a
Flutter app.

## What this SDK is (and is not)

`rook_sdk_core`'s API Sources surface **only** manages connections to **API-based** (non-native) data
sources through OAuth flows. It does **not** read on-device sensors, request device health permissions,
or sync/upload health data itself — the data is delivered to you by ROOK once the user has authorized a
source. If you need to read data directly from the device, use the Health Connect (Android) or Apple
Health (iOS) SDK instead.

## Invariants — do not get these wrong

- **Identifiers:** use `clientUUID` / `client_uuid` (never `customer_id`) and `userID` / `user_id`.
- **Environment:** the SDK environment is `RookEnvironment.sandbox` or `RookEnvironment.production`; each
  has its **own** credentials. Typically SANDBOX for debug builds, PRODUCTION for release.
- **Secrets:** never hardcode or print real credentials — use placeholders `CLIENT_UUID` / `SECRET`.
- **`appId`:** the bundle ID (iOS) or package name (Android). It must match what you registered in the
  ROOK Portal for the selected environment, or initialization fails with `SDKNotAuthorizedException`.
- **Credentials must be registered first:** the `appId` and its secret must be registered in the ROOK
  Portal for the selected environment — each environment (SANDBOX, PRODUCTION) needs its own pair.
- **Singleton:** `RookApiSources` uses in-memory storage only — create one instance (singleton / DI).
- **Versions:** minimum SDK / dependency coordinate lives in `references/setup-and-init.md`; keep it in
  sync with the official docs — never invent a version.

## Integration flow

Open the reference for the step you're on:

1. **Setup & initialization** (install, configure, choose environment, initialize) →
   `references/setup-and-init.md`
2. **Connect data sources** (get authorization URL / OAuth flow, list connected sources, revoke) →
   `references/connections.md`
3. **Troubleshooting** (known exceptions + best practices) → `references/troubleshooting.md`
