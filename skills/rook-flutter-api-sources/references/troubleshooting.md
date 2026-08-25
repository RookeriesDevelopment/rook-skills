# Troubleshooting — Flutter · API Sources

Every function returns its result wrapped in a `Future` and throws a custom ROOK exception when an issue
is encountered. Wrap calls in `try` / `catch`. Below are the ROOK exceptions, their cause 📜, and how to
handle them 🔧.

## Known exceptions

| Exception | Cause | Fix |
|---|---|---|
| `ConnectTimeoutException` | An HTTP request waited too long and could not be completed — often an unavailable or unstable internet connection. | Check connectivity and try again. |
| `HttpRequestException` | An HTTP request failed — usually an error in ROOK servers. | Verify the SDK credentials (`setup-and-init.md`). If correct, read the `message` property and send a report to ROOK support. |
| `SDKNotAuthorizedException` | The operation requires an authorization level your `clientUUID` doesn't have. | Verify the SDK credentials (`setup-and-init.md`). If correct, report to ROOK support — some SDK features need a special authorization level not included in the basic level. |
| `SessionExpiredException` | The SDK session expired and could not be automatically recovered. | Re-initialize the SDK with a valid Client UUID and secret. Ensure the secret and `appId` (package name / bundle ID) are registered in the ROOK Portal. |
| `UnknownException` | Unknown exceptions from the OS (Android / iOS) or the Flutter platform. | Read the `message` property and send a report to ROOK support. |

Quick triage:

- **Authorization / not-authorized errors** → check `CLIENT_UUID` / `SECRET` match the selected
  `environment` (SANDBOX vs PRODUCTION) and that the `appId` + secret are registered in the ROOK Portal
  for that environment.
- **Session-expired errors** → re-create the `RookApiSources` instance with valid credentials.
- **Timeout errors** → connectivity.

## Best practices

### Create the instance once

`RookApiSources` uses in-memory storage only. Create one instance per app launch via a singleton or
dependency injection to avoid multiple instances.

### Use separate credentials per environment

Each environment (`SANDBOX`, `PRODUCTION`) has its own `appId` + secret registered in the ROOK Portal.
Use SANDBOX on debug builds and PRODUCTION on release.

### Enable logging only on debug builds

Set `enableLogs` from a debug flag so logging is off in release builds.

### Prefer Custom Tabs over a WebView

To connect a data source the user must open the provider's web page to log in and grant read access. A
WebView is often not enough — some providers (e.g. Polar) may not render correctly or may be
non-functional in a WebView. Replace the WebView flow with **Custom Tabs** (Android) /
**SFSafariViewController** (iOS), and use an **App Link** (Android) / **Universal Link** (iOS) as the
`redirectUrl` so the user returns to your app when the web-based part of the flow finishes.

A single plugin covers both platforms:

- `flutter_custom_tabs` — uses Custom Tabs on Android and SFSafariViewController on iOS:
  <https://pub.dev/packages/flutter_custom_tabs>
- Android App Links (Flutter): <https://docs.flutter.dev/cookbook/navigation/set-up-app-links>
- iOS Universal Links (Flutter): <https://docs.flutter.dev/cookbook/navigation/set-up-universal-links>

## Still stuck

Tell the developer to read the official ROOK Flutter `rook_sdk_core` documentation:
<https://docs.tryrook.io/docs/rookconnect/sdk/flutter/core/api-sources/>
