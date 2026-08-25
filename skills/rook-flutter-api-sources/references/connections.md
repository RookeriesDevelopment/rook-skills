# Connect data sources — Flutter · API Sources

This SDK's whole surface is the OAuth connection lifecycle for API-based data sources: get an
authorization URL (so the user can connect), list the sources a user has connected, and revoke
(disconnect) a source. There are no device permissions and no manual/background sync — once a user
authorizes a source, ROOK delivers the data to you.

Every function returns a `Future` that completes with a value or throws a ROOK exception. Wrap calls in
`try` / `catch`. See `references/troubleshooting.md` for the exceptions.

## Supported data sources

Garmin, Oura, Polar, Fitbit, Withings, Whoop, Dexcom.

## Get a data source authorizer (start the OAuth connection)

To get the authorization URL for a specific data source, call `getDataSourceAuthorizer` with:

- `userID`
- `dataSource` — one of: `"Garmin"`, `"Oura"`, `"Polar"`, `"Fitbit"`, `"Withings"`, `"Whoop"`,
  `"Dexcom"`
- `redirectUrl` — optional (default `null`). After the user successfully connects, they are redirected
  to this URL. Prefer an **App Link** (Android) / **Universal Link** (iOS) here so the user returns to
  your app (see `references/troubleshooting.md` → best practices).

```dart
try {
  final authorizer = await rookApiSources.getDataSourceAuthorizer(
    userID: userID,
    dataSource: "Fitbit",
    redirectUrl: null,
  );

  // Success
} catch (error) {
  // Handle error
}
```

It returns a `DataSourceAuthorizer`:

```dart
class DataSourceAuthorizer {
  String dataSource;       // The name of the data source.
  bool authorized;         // True if the data source is connected, false otherwise.
  String? authorizationUrl; // The URL to authorize the data source.
}
```

If the user is **not** authorized, `authorizationUrl` holds the URL to open so the user can log in and
grant read access. If the user is already authorized, `authorizationUrl` is `null`.

Open `authorizationUrl` in **Custom Tabs / SFSafariViewController** rather than a WebView (some
providers, e.g. Polar, don't render correctly in a WebView) — see `references/troubleshooting.md`.

## List authorized data sources

To get all data sources the current user has connected, call `getAuthorizedDataSourcesV2` with:

- `userID`

```dart
try {
  final dataSources = await rookApiSources.getAuthorizedDataSourcesV2(
    userID: userID,
  );

  // Success
} catch (error) {
  // Handle error
}
```

It returns a list of `AuthorizedDataSourceV2`:

```dart
class AuthorizedDataSourceV2 {
  String name;      // The name of the data source.
  bool authorized;  // True if the data source is connected, false otherwise.
  String imageUrl;  // The image URL of the data source.
}
```

`authorized` reflects only the **connection status** — not whether the source is actively sending data
or has granted permissions:

- For **API-based** sources (Fitbit, Garmin, Withings, …), `authorized == true` means the user has
  authorized ROOK to retrieve data through that third-party platform.
- For **SDK-based** sources (Apple Health, Health Connect), `authorized == true` only means the user was
  created via the SDK's `updateUserId` function — the user is linked with ROOK via SDK, but it does
  **not** indicate whether the user granted permissions.

## Revoke (disconnect) a data source

To disconnect a data source (revoke authorization), call `revokeDataSource` with:

- `userID`
- `dataSource` — a `DataSourceType`: one of `DataSourceType.garmin`, `DataSourceType.oura`,
  `DataSourceType.polar`, `DataSourceType.fitbit`, `DataSourceType.withings`, `DataSourceType.whoop`.

```dart
try {
  await rookApiSources.revokeDataSource(
    userID: userID,
    dataSource: DataSourceType.withings,
  );

  // Success
} catch (error) {
  // Handle error
}
```
