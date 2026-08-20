# Connect data sources — Android · API Sources

This SDK's whole surface is the OAuth connection lifecycle for API-based data sources: get an
authorization URL (so the user can connect), list the sources a user has connected, and revoke
(disconnect) a source. There are no device permissions and no manual/background sync — once a user
authorizes a source, ROOK delivers the data to you.

Every function returns a `Result` (Kotlin `fold`): a success branch and a `throwable` branch. See
`references/troubleshooting.md` for the exceptions.

## Supported data sources

Garmin, Oura, Polar, Fitbit, Withings, Whoop, Dexcom.

## Get a data source authorizer (start the OAuth connection)

To get the authorization URL for a specific data source, call `getDataSourceAuthorizer` with:

- `userID`
- `dataSourceType` — one of: `"Garmin"`, `"Oura"`, `"Polar"`, `"Fitbit"`, `"Withings"`, `"Whoop"`,
  `"Dexcom"`
- `redirectUrl` — optional (default `null`). After the user successfully connects, they are redirected
  to this URL. Prefer an **App Link** here so the user returns to your app (see
  `references/troubleshooting.md` → best practices).

```kotlin
RookApiSources.getDataSourceAuthorizer(userID, dataSource, REDIRECT_URL).fold(
    {
        // Success
    },
    { throwable ->
        // Handle error
    }
)
```

It returns a `DataSourceAuthorizer`:

```kotlin
data class DataSourceAuthorizer(
    val dataSource: String, // The name of the data source.
    val authorized: Boolean, // True if the data source is connected, false otherwise.
    val authorizationUrl: String?, // The URL to authorize the data source
)
```

If the user is **not** authorized, `authorizationUrl` holds the URL to open so the user can log in and
grant read access. If the user is already authorized, `authorizationUrl` is `null`.

Open `authorizationUrl` in **Chrome Custom Tabs** rather than a WebView (some providers, e.g. Polar,
don't render correctly in a WebView) — see `references/troubleshooting.md`.

## List authorized data sources

To get all data sources the current user has connected, call `getAuthorizedDataSourcesV2` with:

- `userID`

```kotlin
RookApiSources.getAuthorizedDataSourcesV2(userID).fold(
    {
        // Success
    },
    { throwable ->
        // Handle error
    }
)
```

It returns a list of `AuthorizedDataSourceV2`:

```kotlin
data class AuthorizedDataSourceV2(
    val name: String, // The name of the data source.
    val authorized: Boolean, // True if the data source is connected, false otherwise.
    val imageUrl: String, // The image URL of the data source.
)
```

`authorized` reflects only the **connection status** — not whether the source is actively sending data
or has granted permissions:

- For **API-based** sources (Fitbit, Garmin, Withings, …), `authorized == true` means the user has
  authorized ROOK to retrieve data through that third-party platform.
- For **SDK-based** sources (Apple Health, Health Connect), `authorized == true` only means the user
  was linked with ROOK via the corresponding SDK `updateUserId` function — it does **not** indicate
  whether the user granted permissions.

## Revoke (disconnect) a data source

To disconnect a data source (revoke authorization), call `revokeDataSource` with:

- `userID`
- `dataSourceType` — one of: `DataSourceType.GARMIN`, `DataSourceType.OURA`, `DataSourceType.POLAR`,
  `DataSourceType.FITBIT`, `DataSourceType.WITHINGS`, `DataSourceType.WHOOP`

```kotlin
RookApiSources.revokeDataSource(userID, DataSourceType.WITHINGS).fold(
    {
        // Success
    },
    { throwable ->
        // Handle error
    }
)
```
