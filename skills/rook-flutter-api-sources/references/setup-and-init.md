# Setup & initialization — Flutter · API Sources

## Requirements

- **Flutter** — a Flutter project targeting Android and/or iOS.
- **ROOK SDK version** — `rook_sdk_core`, latest stable: **4.1.1**. Keep this in sync with
  `docs/ROOKConnect/SDKs/Flutter/rook_sdk_core/` — never invent a version.

## Install the dependency

Add the package to your project:

```text
flutter pub add rook_sdk_core
```

## Configure & initialize the SDK

All API Sources functions live in `RookApiSources`. Create an instance with your credentials:

- `clientUUID`
- `secret`
- `appId` — the bundle ID (iOS) or package name (Android).
- `environment` — `RookEnvironment.sandbox` or `RookEnvironment.production`.
- `enableLogs` — enable logging only on debug builds.

`RookApiSources` has **in-memory storage only**. To avoid multiple instances, use the singleton pattern
with dependency injection — create one instance per app launch.

Rules:

- Use placeholders `CLIENT_UUID` and `SECRET` in every snippet — **never** real values.
- `environment` is `RookEnvironment.sandbox` or `RookEnvironment.production`; each environment has its
  own `CLIENT_UUID` / `SECRET`. Typical pattern: SANDBOX on debug builds, PRODUCTION on release.
- Enable logging only on debug builds.

```dart
// Available environments:
//
// * SANDBOX    ➞ Use this during your app development process.
// * PRODUCTION ➞ Use this ONLY when your app is published to the store.
final RookEnvironment rookEnvironment =
    isDebug ? RookEnvironment.sandbox : RookEnvironment.production;

// Enable logging only on debug builds
final bool enableLogs = isDebug;

final RookApiSources rookApiSources = RookApiSources(
  clientUUID: CLIENT_UUID,
  secret: SECRET,
  appId: "YOUR BUNDLE ID (iOS) OR PACKAGE NAME (Android)",
  environment: rookEnvironment,
  enableLogs: enableLogs,
);
```

### Critical requirement — register credentials first

You must register your `appId` (bundle ID / package name) and its corresponding secret in the ROOK
Portal **before** creating the `RookApiSources` instance. Failure to register these credentials causes
initialization to fail with an `SDKNotAuthorizedException`.

The ROOK Portal keeps independent configurations for Sandbox and Production. **Each environment requires
its own unique pair of `appId` and secret.** Because the same Flutter package runs on both platforms,
register the Android package name and the iOS bundle ID that your app uses.

If you are upgrading from a previous version, you **must** re-initialize the SDK with this authentication
flow.

## About the user

This SDK has no separate "register user" step. You pass the `userID` (`user_id`) directly to each
connection function (see `references/connections.md`) — never use `customer_id`.
