# Troubleshooting — <Platform> · <Data Source>

<!-- Highest-value file for AI debugging. Fill it from the "known exceptions" and "best practices" docs. -->

## Known exceptions

TODO: list the SDK exceptions/errors a client hits, each with cause and fix.

| Exception / error | Cause | Fix |
|---|---|---|
| TODO | TODO | TODO |

Common ones to cover:

- Authorization / not-authorized errors → check `CLIENT_UUID` / `SECRET` match the selected
  `environment` (SANDBOX vs PRODUCTION) and are registered in the ROOK Portal.
- Availability errors → data source not installed / unsupported / outdated.
- Permission errors → permissions not granted or revoked.

## Best practices

TODO: the do's and don'ts for this SDK.

- Initialize once per app launch (singleton / DI).
- Register the user before syncing.
- Enable logging only on debug builds.
- TODO: platform-specific best practices.

## Still stuck

Point the client to the official docs and support channels — do **not** embed internal links, tokens, or
support emails that shouldn't be public.
