# Troubleshooting — Capacitor · Health Connect and Samsung Health

## Contents

- [Common failures](#common-failures)
- [Package 4.1.1 bridge caveats](#package-411-bridge-caveats)
- [Provider exception families](#provider-exception-families)
- [Diagnostics](#diagnostics)
- [Best practices](#best-practices)

## Common failures

| Symptom or error | Likely cause | Fix |
| --- | --- | --- |
| `Not implemented for web` | Android-only code ran in a browser. | Guard calls with `Capacitor.getPlatform() === 'android'`. |
| Gradle requires API 29 | Published setup text still mentions API 26, but package 4.1.1 declares `minSdkVersion 29`. | Set the Capacitor Android app to API 29+ or install a package whose Gradle contract supports the older level. |
| Missing Samsung classes / R8 failure | Samsung's `.aar` is absent, misnamed, or incompatible. | Add `android/libs/samsung-health-data-api-1.1.0.aar`, its matching Gradle reference, and Gson. |
| ROOK initialization fails even though Health Connect is ready | The 4.1.1 Android bridge initializes both Health Connect and Samsung Health and rejects if either fails. | Complete Samsung native setup and inspect both provider errors. |
| `NOT_INSTALLED` / `NOT_SUPPORTED` for Health Connect | Health Connect is absent or unsupported. | Offer installation only on supported devices; otherwise use another source. |
| Samsung `OUTDATED`, `DISABLED`, or `NOT_READY` | The Samsung Health app is old, disabled, or has incomplete onboarding. | Ask the user to update, enable, or finish onboarding before permission requests. |
| Samsung access-control / code `2003` | Developer mode is off or production package/signature lacks partnership approval. | Enable developer mode for testing; register the release package and SHA-256 with Samsung for production. |
| `incorrect date format` / date parsing failure | A timestamp or locale date was passed. | Pass `yyyy-MM-dd`, for example `2026-08-26`. |
| Records not found | Provider permission is missing, provider has no data, or the upstream wearable has not synced. | Verify permission, open the provider app, trigger its refresh, and retry without treating empty data as success. |
| Health Connect quota exceeded | Manual/background queries ran too frequently. | Stop extraction for several hours and reduce overlapping calls. |
| Missing user / not initialized | Sync started before initialization and `updateUserId`. | Await initialization, register `user_id`, then request permissions and sync. |
| Authorization/session failure | Environment credentials or package name do not match the ROOK Portal. | Verify the environment-specific `CLIENT_UUID`, `SECRET`, and Android package registration. |

## Package 4.1.1 bridge caveats

The current source contains several TypeScript/native inconsistencies. Account for them when debugging and
prefer a package release that aligns the bridge before production use.

### Health Connect background method names

`RookHealthConnect.enableHealthConnectBackGround()` calls native
`scheduleHealthConnectBackGround`, while the Android plugin registers
`enableHealthConnectBackGround`. The disable path has the same `cancel` versus `disable` mismatch. A runtime
“method not implemented” error comes from this mismatch, not from missing permissions.

### Samsung permission-check parameter

The TypeScript helper sends `{ types: [...] }`, but the Android check implementations read an array named
`permissions`. Requests use `types` correctly; the all/partial check helpers can reject with “Missing
permissions parameter”. Do not convert that rejection into a denied state.

### Boolean result shapes

Several public methods are typed as `Promise<boolean>` while native Android resolves `{ result: boolean }`.
Special-access check methods and Health Connect/Samsung permission checks are affected. Inspect the installed
runtime payload and avoid critical branching on an unverified primitive/object shape.

### Initialization side effects

The 4.1.1 Android implementation attempts to enable its legacy Android step manager after a successful
Health Connect initialization, even when `enableBackgroundSync` is false. If an unexpected step foreground
service or permission request appears, inspect this initialization path and upgrade/fix the bridge rather
than adding duplicate step-counter calls.

### Unsupported shared event values

The shared TypeScript union contains `ecg`, but the Android bridge has no ECG mapping. Samsung targeted
temperature sync is also absent even though Samsung permissions include body temperature. Use the provider
support table in `sync.md`.

## Provider exception families

Capacitor rejects promises with native error messages. Common Health Connect families include:

- `HCNotInitializedException`, `HCUserNotInitializedException`
- `HCMissingConfigurationException`, `HCNotAuthorizedException`, `HCSessionExpiredException`
- `MissingHealthConnectPermissionsException`, `MissingAndroidPermissionsException`
- `HealthConnectNotInstalledException`, `HealthConnectNotSupportedException`
- `HCDateNotValidForSummariesException`, `HCDateNotValidForEventsException`
- `HCRecordsNotFoundException`, `HCQuotaExceededException`, `HCTimeoutException`

Common Samsung Health families include:

- `SHNotInitializedException`, `SHUserNotInitializedException`
- `SHNotAuthorizedException`, `SHSessionExpiredException`
- `MissingSamsungHealthPermissionsException`
- `SamsungHealthNotInstalledException`, `SamsungHealthOutdatedException`,
  `SamsungHealthDisabledException`, `SamsungHealthNotReadyException`, `SamsungHealthNotAllowedException`
- `SHDateNotValidForSummariesException`, `SHDateNotValidForEventsException`
- `SHRecordsNotFoundException`, `SHTimeoutException`

Use the error family to route the user to configuration, user registration, provider setup, permission,
date correction, backoff, or connectivity recovery.

## Diagnostics

Request diagnostics for one Android source at a time and use them only in development:

```ts
const healthConnect = await RookConfig.getDiagnostic({
  androidSource: 'HEALTH_CONNECT',
});

const samsung = await RookConfig.getDiagnostic({
  androidSource: 'SAMSUNG',
});

console.debug(healthConnect.result, samsung.result);
```

The snapshot reports configuration, user identification, permission state, and last background/manual
sync state. Do not log credentials or health data in production.

## Best practices

- Centralize initialization and provider selection in one app-level service.
- Test Health Connect behavior on relevant Android versions and Samsung behavior on a physical device.
- Ask for permissions and background access in context, not automatically at launch.
- Keep Health Connect and Samsung Health as distinct sources unless the backend deliberately handles
  duplication.
- Prefer provider background sync; use manual sync for explicit refresh or targeted recovery.
- Treat the sensor-based `RookStepsCounter` as a third source and do not combine it blindly with provider
  steps.
- Re-run `npx cap sync android` after changing the package or native Android configuration.
