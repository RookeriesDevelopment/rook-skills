# Troubleshooting — Capacitor · Apple Health

## Common failures

| Symptom or error | Likely cause | Fix |
| --- | --- | --- |
| `Not implemented for web` | A native-only method ran in a browser/PWA path. | Guard every call with `Capacitor.getPlatform() === 'ios'`. |
| HealthKit UI never appears | HealthKit capability or usage descriptions are missing, or the type was already decided. | Add the capability and `Info.plist` keys; direct the user to Apple Health settings for prior decisions. |
| No data after a successful permission request | Apple hides read denial, the store is empty, the provider has not synced, or the device is locked. | Do not equate success with granted reads; check provider data and retry after unlock. |
| `incorrect date format should be yyyy-MM-dd` | A date/time or locale-formatted string was passed. | Pass a calendar date such as `2026-08-26`. |
| Missing/not initialized user | Sync ran before `updateUserId` succeeded. | Initialize, register `user_id`, request permissions, then sync. |
| Authorization or session error | Credentials, environment, or bundle identifier do not match the ROOK Portal. | Verify the environment-specific `CLIENT_UUID`, `SECRET`, and bundle ID registration. |
| Background sync never runs | Native observers or Xcode background capabilities are missing. | Add `setBackListeners()` at launch, HealthKit Background Delivery, and Background fetch. |
| Background error listener is silent | `starListening()` was not called or the listener was removed. | Add the listener once, call the package's current `starListening()` spelling, and retain its handle. |
| CocoaPods build/link failure | Pods were not installed after adding/updating the plugin. | Run `npx cap sync ios` and, if needed, `npx pod-install`; open the workspace. |

## Package 4.1.1 bridge caveats

The package's TypeScript declarations and native iOS return payloads are not fully aligned:

- `requestAppleHealthPermissions()` is typed as `Promise<boolean>`, while the native bridge resolves a
  payload containing `result`.
- `enableAppleHealthSync()` and related state methods have similar boolean/object inconsistencies.
- `writeNutritionEvent()` is declared as `BoolResult`, but the native success payload is not currently a
  normal boolean result.

Avoid building critical control flow around those mismatched payload shapes. Await the operation, catch
rejection, and verify the installed package's runtime payload before consuming it. Upgrade to a release
that aligns the bridge when available.

## Diagnostics

Use diagnostics only in development:

```ts
const diagnostic = await RookConfig.getDiagnostic();

console.debug({
  configured: diagnostic.result.isConfigured,
  userIdentified: diagnostic.result.userIdentified,
  permissions: diagnostic.result.permissions,
  background: diagnostic.result.backgroundSync,
  manual: diagnostic.result.manualSync,
});
```

Do not log credentials, user health data, or diagnostic payloads containing sensitive context in
production.

## Best practices

- Initialize once and centralize ROOK calls in one app-level service.
- Test HealthKit and background behavior on a physical iPhone.
- Request the smallest permission set from a user-initiated flow.
- Await syncs and avoid overlapping the same date/type.
- Treat Apple Health as eventually available, not real time.
- Use automatic sync for the default integration and manual sync for explicit retry or targeted refresh.
