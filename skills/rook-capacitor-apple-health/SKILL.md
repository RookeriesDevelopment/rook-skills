---
name: rook-capacitor-apple-health
description: >
  Integrate the ROOK Capacitor Apple Health SDK in an iOS app: install and configure
  capacitor-rook-sdk, initialize credentials, register or remove users, request HealthKit permissions,
  and sync summaries and events manually or in the background. Use when a Capacitor or Ionic project
  needs ROOK, Apple Health, HealthKit, iOS health data, permissions, or automatic synchronization.
---

# ROOK Capacitor — Apple Health

Guide a Capacitor app through the native iOS and TypeScript parts of the ROOK Apple Health integration.

## Invariants

- Gate Apple Health calls with `Capacitor.getPlatform() === 'ios'`. The web implementation throws
  `Not implemented for web`.
- Use `client_uuid` and the app end-user's `user_id`; never substitute `customer_id`.
- Use `environment: 'sandbox'` for development and `'production'` for release. Credentials are different
  per environment.
- Use placeholders `CLIENT_UUID` and `SECRET`; never hardcode, print, or commit real credentials.
- Initialize once per native app launch. Register a user and request permissions before syncing.
- Use `yyyy-MM-dd` for every date passed to the Capacitor API.
- Treat Apple Health read authorization and empty data as ambiguous; iOS does not disclose read denial.
- Keep dependency and native SDK versions aligned with `references/setup-and-init.md`; never invent one.

## Public modules

- `RookConfig` — initialization, diagnostics, user registration, logout, and timezone.
- `RookPermissions` — Apple Health read permissions, nutrition write permission, and settings.
- `RookSummaries` — manual sleep, physical, and body summary sync.
- `RookEvents` — manual events and current-day steps, calories, and heart rate.
- `RookAppleHealth` — continuous upload, background summary/event controls, and background errors.

## Integration flow

Open only the reference needed for the current task:

1. Install, configure iOS, initialize, and register/remove the user → `references/setup-and-init.md`
2. Check availability and request HealthKit permissions → `references/permissions.md`
3. Trigger manual summary or event sync → `references/sync.md`
4. Enable continuous or HealthKit background sync → `references/background.md`
5. Diagnose integration and version-specific problems → `references/troubleshooting.md`
