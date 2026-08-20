# Troubleshooting — Android · API Sources

`rook-api-sources` returns custom exceptions wrapped in the `Result` of each function. Below are the
ROOK exceptions, their cause 📜, and how to handle them 🔧.

## Known exceptions

| Exception | Cause | Fix |
|---|---|---|
| `ApiHttpRequestException` | An HTTP request failed — usually an error in ROOK servers. | Verify the SDK credentials (`setup-and-init.md`). If correct, read the `httpCode` / `httpMessage` properties and send a report to ROOK support. |
| `ApiNotAuthorizedException` | The operation requires an authorization level your `clientUUID` doesn't have. | Verify the SDK credentials (`setup-and-init.md`). If correct, report to ROOK support — some SDK features need a special authorization level not included in the basic level. |
| `ApiNotInitializedException` | The SDK was not initialized or failed to initialize. | Call `RookApiSources.initRook()` and wait for a successful result. |
| `ApiSessionExpiredException` | The SDK session expired and could not be automatically recovered. | Re-initialize the SDK with a valid Client UUID and secret. Ensure the secret and package name are registered in the ROOK Portal. |
| `ApiTimeoutException` | An HTTP request waited too long — often an unavailable or unstable internet connection. | Check connectivity and try again. |

Quick triage:

- **Authorization / not-authorized errors** → check `CLIENT_UUID` / `SECRET` match the selected
  `environment` (SANDBOX vs PRODUCTION) and that the package name + secret are registered in the ROOK
  Portal for that environment.
- **Not-initialized errors** → initialize once (singleton / DI) before calling any other function.
- **Timeout errors** → connectivity.

## Best practices

### Initialize once

`RookApiSources` uses in-memory storage only. Initialize once per app launch via a ServiceLocator or
dependency injection to avoid multiple initializations.

### Use separate credentials per environment

Each environment (`SANDBOX`, `PRODUCTION`) has its own package name + secret registered in the ROOK
Portal. Use SANDBOX on debug builds and PRODUCTION on release.

### Enable logging only on debug builds

Guard `apiSources.enableLocalLogs()` behind `BuildConfig.DEBUG`.

### Prefer Chrome Custom Tabs over a WebView

To connect a data source the user must open the provider's web page to log in and grant read access.
A WebView is often not enough — some providers (e.g. Polar) may not render correctly or may be
non-functional in a WebView. Replace the WebView flow with **Chrome Custom Tabs**, and use an **App
Link** as the `redirectUrl` so the user returns to your app when the web-based part of the flow
finishes.

- Custom Tabs: <https://developer.chrome.com/docs/android/custom-tabs/guide-get-started>
- App Links: <https://developer.android.com/training/app-links>

## Still stuck

See the official ROOK API Sources docs:
<https://docs.tryrook.io/docs/category/sdks/android/api-sources/>
