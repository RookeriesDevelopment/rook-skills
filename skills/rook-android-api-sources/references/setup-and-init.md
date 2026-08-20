# Setup & initialization — Android · API Sources

## Requirements

- **Android Studio** — Narwhal 4 Feature Drop | 2025.1.4 or higher is recommended.
- **SDK levels** — in your `build.gradle` (app module):

  ```groovy
  minSdk 26
  targetSdk 36
  ```

- **ROOK SDK version** — latest stable: **4.0.1**. Keep this in sync with
  `docs/ROOKConnect/SDKs/Android/ApiSources/` — never invent a version.

## Install the dependency

In your **build.gradle** (app module) add the dependency, replacing `version` with the current stable
version (see above):

```groovy
implementation("io.tryrook.android:rook-api-sources:version")
```

## Configure & initialize the SDK

All SDK functions live in `RookApiSources`. Create an instance and initialize it with:

- `clientUUID`
- `secret`
- `environment`
- `packageName` — optional; if not provided the SDK reads it from `Context`.

`RookApiSources` has **in-memory storage only**. To avoid multiple initializations, use the singleton
pattern with a ServiceLocator or dependency injection — initialize once per app launch.

Rules:

- Use placeholders `CLIENT_UUID` and `SECRET` in every snippet — **never** real values.
- `environment` is `ApiEnvironment.SANDBOX` or `ApiEnvironment.PRODUCTION`; each environment has its own
  `CLIENT_UUID` / `SECRET`. Typical pattern: SANDBOX on debug builds, PRODUCTION on release.
- Enable logging only on debug builds.

```kotlin
val apiSources = RookApiSources(context)

// Available environments:
//
// * SANDBOX    ➞ Use this during your app development process.
// * PRODUCTION ➞ Use this ONLY when your app is published to the PlayStore.
val apiEnvironment = if (BuildConfig.DEBUG) ApiEnvironment.SANDBOX
else ApiEnvironment.PRODUCTION

// Enable logging only on debug builds
if (BuildConfig.DEBUG) {
    apiSources.enableLocalLogs()
}

val configuration = ApiConfiguration(
    clientUUID = CLIENT_UUID,
    secret = SECRET,
    environment = apiEnvironment,
)

apiSources.initRook(configuration)
```

### Critical requirement — register credentials first

You must register your `applicationId` (package name) and its corresponding secret in the ROOK Portal
**before** initializing the SDK. Failure to register these credentials causes initialization to fail
with an `ApiNotAuthorizedException`.

The ROOK Portal keeps independent configurations for Sandbox and Production. **Each environment requires
its own unique pair of package name and secret.**

If you are upgrading from a previous version, you **must** re-initialize the SDK with this
authentication flow.

## About the user

This SDK has no separate "register user" step. You pass the `userID` (`user_id`) directly to each
connection function (see `references/connections.md`) — never use `customer_id`.
