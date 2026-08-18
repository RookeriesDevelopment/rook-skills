# Skill template — how to use it

> **This file is scaffolding, not part of the skill. Delete it after you copy the template.**

This folder is the starting skeleton every ROOK SDK skill is built from, per
[`AI-SKILLS-CONVENTIONS.md`](../AI-SKILLS-CONVENTIONS.md). Each team builds its own SDK's skill(s) from this.

## Steps

1. Copy `skill-template/` into the `rook-skills` repo under `skills/`, renamed to your skill:
   `skills/rook-<platform>-<data-source>/` (e.g. `skills/rook-android-health-connect/`).
2. Delete this `README.md`.
3. Fill in every `TODO` and replace every `<placeholder>`:
   - `<platform>` → `android` | `ios` | `flutter` | `react-native` | `capacitor`
   - `<data-source>` → `health-connect` | `samsung-health` | `apple-health` | `api-sources`
   - `<Platform>` / `<Data Source>` → human-readable form for prose.
4. Rename `examples/` snippets to the real extension (`.kt` / `.swift` / `.dart`).
5. Keep it honest against the hard rules in the conventions doc: self-contained, no secrets, correct
   invariants (`client_uuid`, `user_id`, `environment` = SANDBOX/PRODUCTION),
   docs as the single source of truth.

## Structure

```
SKILL.md            # router — short; what/when/invariants + pointers to references
references/         # the detail, loaded on demand
  setup-and-init.md # FOLDED CORE: install, config, environment, user registration
  permissions.md
  sync.md
  background.md
  troubleshooting.md
examples/           # 1–3 runnable golden-path snippets
INSTALL.md          # how a client installs this skill (npx skills + clone fallback)
```
