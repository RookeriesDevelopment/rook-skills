---
name: rook-react-native-apple-health
description: >
  Integrate the ROOK React Native SDK with Apple Health: install and configure react-native-rook-sdk,
  initialize RookSyncGate, register or remove users, request HealthKit permissions, and sync summaries,
  events, and current-day health data manually or in the background. Use for ROOK, Apple Health,
  HealthKit, iOS health-data, permission, nutrition, or synchronization work in React Native apps.
---

# ROOK React Native — Apple Health

Guide a React Native app through the JavaScript and native iOS parts of the ROOK Apple Health integration.

## Invariants

- Target the unified `react-native-rook-sdk` 4.x API. Do not mix examples from the legacy
  `react-native-rook-sdk-apple-health` package or the 5.x TurboModule project.
- Wrap the native app once with `RookSyncGate`, and wait for `useRookConfiguration().ready` before shared calls.
  Also honor hook-specific `ready` values where exposed; `useRookSync` itself does not expose one.
- Follow the Rules of Hooks: call ROOK hooks unconditionally at component or custom-hook scope, then branch on
  `Platform.OS` inside effects and handlers.
- Use `client_uuid` and the app end-user's `user_id`; never substitute `customer_id`.
- Use `environment="sandbox"` for development and `"production"` for release. Credentials are different per
  environment and registered bundle ID.
- Use placeholders `CLIENT_UUID`, `SECRET`, and `USER_ID`; never print or commit real credentials. Do not claim
  that `.env` values are secret after React Native bundles them.
- Use `SDKDataSource.APPLE_HEALTH` on iOS and `yyyy-MM-dd` for sync/query dates.
- Treat Apple Health read authorization and empty data as ambiguous; iOS does not disclose read denial.
- Keep dependency and native SDK versions aligned with `references/setup-and-init.md`; never invent a version.

## Public integration surface

- `RookSyncGate` — SDK initialization and optional automatic synchronization.
- `useRookConfiguration` — readiness, diagnostics, user registration, logout, and timezone.
- `useRookPermissions` — Apple Health permissions, nutrition write access, and settings.
- `useRookSync` — manual summary and event upload.
- `useRookData` / `useRookVariables` — local summaries, activities, steps, calories, and heart rate.
- `useRookAppleHealth` — Apple Health automatic/background upload controls.

## Integration flow

Open only the reference needed for the current task:

1. Install, configure Xcode, initialize, and register/remove the user → `references/setup-and-init.md`
2. Check and request Apple Health permissions → `references/permissions.md`
3. Trigger manual sync or read supported local values → `references/sync.md`
4. Configure HealthKit background delivery and automatic sync → `references/background.md`
5. Diagnose integration and version-specific problems → `references/troubleshooting.md`
