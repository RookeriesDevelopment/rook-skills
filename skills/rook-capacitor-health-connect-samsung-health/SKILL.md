---
name: rook-capacitor-health-connect-samsung-health
description: >
  Integrate the ROOK Capacitor Android SDK with Health Connect and Samsung Health: install and configure
  capacitor-rook-sdk, initialize credentials, register or remove users, check provider availability,
  request permissions, choose HEALTH_CONNECT, SAMSUNG, or ALL, and sync health summaries and events
  manually or in the background. Use for ROOK health-data work in Capacitor or Ionic Android apps.
---

# ROOK Capacitor — Health Connect and Samsung Health

Guide a Capacitor Android app through both supported on-device data sources without mixing provider-specific
requirements or permission flows.

## Invariants

- Gate these calls with `Capacitor.getPlatform() === 'android'`; the web implementation is unavailable.
- Use the app end-user's `user_id` and the ROOK `client_uuid`; never substitute `customer_id`.
- Use `'sandbox'` for development and `'production'` for release, with credentials registered separately
  for each environment and package name.
- Use placeholders `CLIENT_UUID` and `SECRET`; never log or commit real credentials.
- Initialize once. Register the user and obtain the selected provider's permissions before syncing.
- Pass `dataSource: 'HEALTH_CONNECT'`, `'SAMSUNG'`, or `'ALL'` explicitly for manual sync. Omission defaults
  to Health Connect.
- Use `yyyy-MM-dd` dates and stay within the provider's supported historical window.
- Keep the Capacitor package, Android SDK, Samsung `.aar`, SDK levels, and permission declarations aligned
  with `references/setup-and-init.md`; never invent versions.

## Public modules

- `RookConfig` — shared initialization, diagnostics, user registration, logout, and timezone.
- `RookPermissions` — Health Connect, Android service, Samsung Health, alarm, battery, and OEM permissions.
- `RookHealthConnect` — Health Connect availability and background controls.
- `RookSamsungHealth` — Samsung Health availability and background controls.
- `RookSummaries` / `RookEvents` — provider-selectable manual sync.
- `RookStepsCounter` — optional Android sensor-based step counter; keep separate from provider data.

## Integration flow

Open only the reference needed for the current task:

1. Install, configure Android/provider access, initialize, and register/remove user →
   `references/setup-and-init.md`
2. Check provider availability and request permissions → `references/permissions.md`
3. Sync summaries and events manually → `references/sync.md`
4. Configure background provider sync or the optional step counter → `references/background.md`
5. Diagnose integration, provider, and package-version problems → `references/troubleshooting.md`
