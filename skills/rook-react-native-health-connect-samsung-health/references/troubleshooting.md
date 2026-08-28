# Troubleshooting — React Native · Health Connect and Samsung Health

## Contents

- [Common failures](#common-failures)
- [Package 4.1.1 caveats](#package-411-caveats)
- [Provider checks](#provider-checks)
- [Diagnostics](#diagnostics)
- [Best practices](#best-practices)

## Common failures

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Android build cannot resolve Samsung classes | Missing or mismatched `.aar` | `android/libs`, artifact name, and app Gradle dependency |
| Hook says it is not ready | Called before initialization | Gate shared calls with configuration readiness |
| `ready` never becomes true | Gate/native initialization failed | Credentials, environment, package name, Gradle versions, and logs |
| Health Connect action is unavailable | Provider missing or unsupported | `checkHealthConnectAvailability()` |
| Samsung returns disabled/outdated/not ready | Samsung Health installation state | `checkSamsungAvailability()` and Samsung app |
| Debug Samsung works but release fails | Partner identity mismatch | Release package name and signing SHA-256 |
| Sync reports one source failed | Both sources were requested | Inspect each callback result and target only the intended provider |
| Background work stops | Permission, battery, force-stop, or OEM policy | Provider state, Android settings, and app launch history |
| Release-only serialization/network failure | R8 removed required metadata | Crypto, Retrofit, and coroutine keep rules |

## Package 4.1.1 caveats

- The Android initializer sets the Samsung configuration environment to `SANDBOX` even when the gate receives
  `environment="production"`. Treat production Samsung support as blocked on a package fix; do not claim it is using
  production in 4.1.1.
- `getDiagnosticState(SDKDataSource.SAMSUNG_HEALTH)` calls the native promise without `await` and then passes the
  promise to `JSON.parse`, so the helper fails in 4.1.1. Use native logs/provider checks or upgrade to a fixed package.
- `androidHasAlarmPermissions()` calls the generic Android-permission check natively instead of the exact-alarm check.
  Do not use it as authoritative exact-alarm status in 4.1.1.
- `useRookSync.sync(callback)` without parameters attempts both Health Connect and Samsung Health on Android. Use a
  parameterized call to avoid an expected failure from an unavailable provider.
- The hook's Samsung full/partial checks use the full native permission set, even after a custom subset request.
- `requestWriteNutritionPermission()` is guarded as iOS-only in 4.1.1, while Android `writeNutritionData` has a
  Health Connect implementation. Do not expose Android nutrition writing without a corrected permission flow.
- `useRookSync.sync()` returns `void`; awaiting it does not await its callback.
- Avoid obsolete `scheduleYesterdaySync`, `useRookSummaries`, `useRookEvents`, `requestAllPermissions`, and old
  provider-package imports.

## Provider checks

Before diagnosing extraction, record non-sensitive state:

```tsx
const hcAvailability = await checkHealthConnectAvailability();
const hcPermissions = await healthConnectHasPartialPermissions();
const hcBackground = await checkHealthConnectBackgroundReadStatus();

const samsungAvailability = await checkSamsungAvailability();
const samsungPermissions = samsungAvailability === 'INSTALLED'
  ? await samsungHealthHasPartialPermissions()
  : false;
```

Do not log raw health data, credentials, or personally identifying user IDs.

## Diagnostics

Health Connect diagnostics work through the shared configuration hook:

```tsx
const state = await getDiagnosticState(SDKDataSource.HEALTH_CONNECT);
```

Check `isConfigured`, `userIdentified`, `permissions`, `backgroundSync`, and `manualSync`. For Samsung Health in
4.1.1, use the provider availability/permission/background methods until the missing `await` is fixed.

## Best practices

- Reproduce on a physical device with known records and the exact release/debug signing identity in question.
- Verify `getUserID()` before debugging provider permissions or sync.
- Use one source, one date, and one summary/event type to isolate failures.
- Inspect the merged manifest, not only the library or app manifest in isolation.
- Compare the installed NPM package, native ROOK dependencies, and Samsung `.aar` before comparing apps.
- Keep hooks unconditional and remove `NativeEventEmitter` subscriptions when the component unmounts.
