# Setup & initialization — <Platform> · <Data Source>

<!-- FOLDED CORE: this file carries the shared auth/config/environment/user setup so the skill is self-contained.
     Do NOT link out to a separate "core" skill. -->

## Project configuration

- TODO: Setup Capabilities and PList (IOS) and Manifest/Gradle Config (Android).

## Requirements

- TODO: minimum OS / tooling version (e.g. Android Studio, Xcode, Flutter version).
- TODO: minimum ROOK SDK version and current stable version. Keep in sync with
  `docs/ROOKConnect/SDKs/<Platform>/` — never invent a version.

## Install the dependency

TODO: the dependency declaration for this platform.

- **Android** — `build.gradle` (app module): `implementation("com.rookmotion.android:rook-sdk:<version>")`
- **iOS** — Swift Package Manager or CocoaPods.
- **Flutter** — `pubspec.yaml`.

## Configure & initialize the SDK

TODO: create the configuration manager, set credentials, pick the environment, initialize.

Rules:

- Use placeholders `CLIENT_UUID` and `SECRET` in every snippet — **never** real values.
- `environment` is `SANDBOX` or `PRODUCTION`; each environment has its own `CLIENT_UUID` / `SECRET`.
  Typical pattern: SANDBOX on debug builds, PRODUCTION on release.
- Initialize once per app launch (singleton / DI).

```text
TODO: minimal init snippet using CLIENT_UUID, SECRET, and the SANDBOX/PRODUCTION environment.
```

## Register / update the user

TODO: how to register or update the user with `user_id`.

- Never use `customer_id`.
- TODO: note when the user must be registered relative to permissions/sync.

```text
TODO: user registration snippet.
```
