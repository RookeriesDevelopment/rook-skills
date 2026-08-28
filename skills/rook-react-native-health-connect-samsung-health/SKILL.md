---
name: rook-react-native-health-connect-samsung-health
description: >
  Integrate the ROOK React Native Android SDK with Health Connect and Samsung Health: install and
  configure react-native-rook-sdk, initialize RookSyncGate, register or remove users, check provider
  availability, request permissions, choose SDKDataSource values, and sync summaries and events manually
  or in the background. Use for ROOK, Health Connect, Samsung Health, Android health-data, permission,
  step-counter, or synchronization work in React Native apps.
---

# ROOK React Native — Health Connect and Samsung Health

Guide a React Native Android app through both supported on-device data sources without mixing their native
configuration, availability, permission, or background workflows.

## Invariants

- Target the unified `react-native-rook-sdk` 4.x API. Do not mix examples from legacy provider packages or the
  separate 5.x TurboModule project.
- Wrap the native app once with `RookSyncGate`, and wait for `useRookConfiguration().ready` before shared calls.
  Also honor hook-specific `ready` values where exposed; `useRookSync` itself does not expose one.
- Follow the Rules of Hooks: call ROOK hooks unconditionally at component or custom-hook scope, then branch on
  `Platform.OS` inside effects and handlers.
- Use `client_uuid` and the app end-user's `user_id`; never substitute `customer_id`.
- Use `environment="sandbox"` for development and `"production"` for release, with credentials and package name
  registered for that environment.
- Use placeholders `CLIENT_UUID`, `SECRET`, and `USER_ID`; never log or commit real credentials. Do not describe
  `.env` values embedded in a React Native bundle as secret.
- Use `SDKDataSource.HEALTH_CONNECT` or `SDKDataSource.SAMSUNG_HEALTH` explicitly in targeted sync and query calls.
- Use `yyyy-MM-dd` dates and stay within the provider's supported historical range.
- Keep package, Android SDK, Kotlin, and Samsung `.aar` versions aligned with `references/setup-and-init.md`.

## Public integration surface

- `RookSyncGate` — shared initialization and optional automatic synchronization.
- `useRookConfiguration` — readiness, diagnostics, user registration, logout, and timezone.
- `useRookPermissions` — Health Connect, Samsung Health, Android service, alarm, battery, and OEM permissions.
- `useRookSync` — provider-selectable manual summary and event upload.
- `useRookData` / `useRookVariables` — local summaries, activities, steps, calories, and heart rate.
- `useRookHealthConnect` / `useRookSamsungHealth` — provider background controls.
- `useRookAndroidStepCounter` — optional device-sensor step counter, separate from provider data.

## Integration flow

Open only the reference needed for the current task:

1. Install, configure Android/providers, initialize, and register/remove the user →
   `references/setup-and-init.md`
2. Check provider availability and request permissions → `references/permissions.md`
3. Sync summaries/events manually or read supported local values → `references/sync.md`
4. Configure provider background sync or the optional device step counter → `references/background.md`
5. Diagnose integration, provider, and package-version problems → `references/troubleshooting.md`
