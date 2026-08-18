---
name: rook-<platform>-<data-source>
author: ROOK
description: >
  Integrate the ROOK <Platform> <Data Source> SDK: set up credentials, register users, request
  permissions, and sync health summaries and events (steps, calories, sleep, heart rate). Use when a
  developer is adding ROOK, <Data Source>, or wearable/health-data sync to a <Platform> app.
---

# ROOK <Platform> — <Data Source> SDK

<!-- SKILL.md is the router. Keep it short. Put the detail in references/ so agents load only what they need. -->

## What this skill does

Guides a developer through integrating the ROOK <Data Source> SDK on <Platform>: SDK setup, user
registration, permissions, and syncing health data (summaries and events).

## When to use it

Use when the developer mentions ROOK, <Data Source>, health-data sync, wearables, permissions, or
background sync in a <Platform> project.

## Invariants — do not get these wrong

- **Identifiers:** use `client_uuid` (never `customer_id`) and `user_id`.
- **Environment:** the SDK environment is `SANDBOX` or `PRODUCTION`; each has its **own** credentials.
  Typically SANDBOX for debug builds, PRODUCTION for release.
- **Secrets:** never hardcode or print real credentials — use placeholders `CLIENT_UUID` / `SECRET`.
- **Versions:** minimum SDK version / dependency coordinate lives in `references/setup-and-init.md`;
  keep it in sync with the official docs — never invent a version.

## Integration flow

Open the reference for the step you're on:

1. **Setup & initialization** (install, configure, choose environment) → `references/setup-and-init.md`
2. **Register / update the user** (`user_id`) → `references/setup-and-init.md`
3. **Availability & permissions** → `references/permissions.md`
4. **Sync health data** (manual) → `references/sync.md`
5. **Sync health data** (automatic) → `references/background.md`
6. **Troubleshooting** (known exceptions + best practices) → `references/troubleshooting.md`

## Examples

See `examples/` for copy-paste golden-path snippets.
