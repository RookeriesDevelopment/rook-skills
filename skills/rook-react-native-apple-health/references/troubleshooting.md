# Troubleshooting — React Native · Apple Health

## Common failures

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Hook says it is not ready | Called before initialization | Gate shared calls with configuration readiness |
| `ready` never becomes true | Gate initialization failed | Credentials, environment, bundle ID, pods, and native logs |
| Native module is missing | Package was added without a native rebuild | Run `npx pod-install`, clean, and rebuild the iOS app |
| Permission request resolves but reads are empty | No samples or denied read access | Health app data/source, requested types, date, and device |
| Background sync never starts | Missing entitlement, listener, user, or permission | Xcode capabilities, AppDelegate, gate state, and hook status |
| Event sync rejects | Wrong event/source or unsupported OS/type | Apple event enum, permissions, date, and deployment target |
| Nutrition write rejects | Missing write permission or invalid date | Purpose string, permission request, and ISO-8601 timezone |

## Package 4.1.1 caveats

- The published 4.x documentation omits some values present in the 4.1.1 source, including
  `AppleHealthPermission.WORKOUT_ROUTE`, effort-related permission values, and
  `UniqueAppleHealthEvents.ECG`. Prefer exported enums from the installed package.
- `RookSyncGateProviderProps` declares `enableEventsBackgroundSync`, but the 4.1.1 provider does not read or pass
  that prop. Use the single effective `enableBackgroundSync` flag and `useRookAppleHealth` controls.
- Native Apple Health background failures arrive as `ROOK_APPLE_HEALTH_BACKGROUND_ERROR` on the shared
  `ROOK_NOTIFICATION` emitter; the automatic-start state uses `ROOK_BACKGROUND_ENABLED`.
- The exported `Calories` type contains `active` and `basal`; native iOS also returns `total`.
- `useRookConfiguration.removeUserFromRook` requires a sources array in TypeScript, but iOS ignores that array and
  removes the Apple Health user.
- `useRookSync.sync()` is callback-based and returns `void`. Awaiting it does not await completion.
- Do not use older `useRookSummaries`, `useRookEvents`, `requestAllPermissions`, or
  `syncYesterdaySummaries` examples with the unified 4.x API.

## Diagnostics

Read the provider diagnostic state after `ready`:

```tsx
const { getDiagnosticState } = useRookConfiguration();

const state = await getDiagnosticState(SDKDataSource.APPLE_HEALTH);
console.log({
  isConfigured: state.isConfigured,
  userIdentified: state.userIdentified,
  permissions: state.permissions,
  backgroundSync: state.backgroundSync,
  manualSync: state.manualSync,
});
```

Do not log credentials, raw health data, or personally identifying user IDs while diagnosing.

## Best practices

- Reproduce on a physical iPhone with known HealthKit samples and the same signing configuration as the failing
  build.
- Verify `getUserID()` before blaming permission or sync behavior.
- Use one targeted date and one summary/event type to isolate extraction failures.
- Check the exact installed package and native pod versions before comparing results with another app.
- Keep hook calls unconditional and remove `NativeEventEmitter` subscriptions on unmount.
