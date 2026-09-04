# ROOK SDK — AI Skills

AI **Agent Skills** that teach your AI coding tool how to integrate the [ROOK](https://tryrook.io) SDKs — so it can
scaffold credential setup, user registration, permissions, and health-data sync for you, correctly and to the current
SDK version.

Every skill is built from the **official ROOK SDK documentation** at
[docs.tryrook.io](https://docs.tryrook.io/docs/category/sdks/) and kept in sync with it, so the guidance your agent
follows matches what's published.

Each skill follows the open [`SKILL.md`](https://github.com/vercel-labs/skills) standard, so the **same skill works
across Claude Code, Cursor, Codex, Antigravity, Cline, and 75+ other agents** — only the install directory differs, and
the installer handles that automatically.

---

## What is a skill?

A skill is a self-contained folder the agent loads **on demand** when your prompt mentions ROOK, the platform, or the
data source. It gives the agent exactly the ROOK-specific knowledge it needs — SDK coordinates, init flow, permission
model, sync APIs, and the common failure modes — without you pasting docs into the chat.

- **Progressive disclosure** — the agent pulls in only the reference file it needs, not the whole doc set.
- **Auto-triggering** — the skill's `description` fires on keywords like *ROOK*, *Health Connect*, *Apple Health*,
  *wearables*, *permissions*, *sync*.
- **Self-contained** — one skill = one data source, on one platform, with auth/config/user-management folded in. Nothing
  else to install.

---

## Available skills

Naming scheme: **`rook-{platform}-{data-source}`**.

| Platform         | Skill                                             | Integrates                                                                                                |
|------------------|---------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| **Android**      | `rook-android-health-connect`                     | Health Connect — summaries & events (steps, calories, sleep, heart rate), manual + background sync        |
| **Android**      | `rook-android-samsung-health`                     | Samsung Health — summaries & events, manual + background sync                                             |
| **Android**      | `rook-android-api-sources`                        | API/third-party sources (Garmin, Oura, Polar, Fitbit, Withings, Whoop, Dexcom) via OAuth                  |
| **iOS**          | `rook-ios-apple-health`                           | Apple Health — summaries & events, manual + background sync                                               |
| **Flutter**      | `rook-flutter-health-connect`                     | Health Connect (Android) — summaries & events, manual + background sync                                   |
| **Flutter**      | `rook-flutter-samsung-health`                     | Samsung Health (Android) — summaries & events, manual + background sync                                   |
| **Flutter**      | `rook-flutter-apple-health`                       | Apple Health / HealthKit (iOS) — summaries & events, manual + background sync                             |
| **Flutter**      | `rook-flutter-api-sources`                        | API/third-party sources (Garmin, Oura, Polar, Fitbit, Withings, Whoop, Dexcom) via OAuth — iOS & Android  |
| **React Native** | `rook-react-native-apple-health`                  | Apple Health / HealthKit (iOS) — summaries, events & current-day data, manual + background sync           |
| **React Native** | `rook-react-native-health-connect-samsung-health` | Health Connect + Samsung Health (Android) — summaries & events, manual + background sync                  |
| **Capacitor**    | `rook-capacitor-apple-health`                     | Apple Health / HealthKit (iOS, Capacitor/Ionic) — summaries & events, manual + background sync            |
| **Capacitor**    | `rook-capacitor-health-connect-samsung-health`    | Health Connect + Samsung Health (Android, Capacitor/Ionic) — summaries & events, manual + background sync |

---

## Install

Replace `rook-{platform}-{data-source}` with a skill name from the table above.

### Option A — `skills` CLI (recommended)

Detects your AI tool and installs into the right directory automatically:

```bash
npx skills add RookeriesDevelopment/rook-skills --skill rook-{platform}-{data-source}
```

Target a specific agent and skip prompts:

```bash
npx skills add RookeriesDevelopment/rook-skills --skill rook-{platform}-{data-source} -a claude-code -y
```

### Option B — clone + copy (no Node.js required)

```bash
git clone --depth 1 https://github.com/RookeriesDevelopment/rook-skills
```

```bash
# Claude / Claude Code:
cp -r rook-skills/skills/rook-{platform}-{data-source} .claude/skills/

# Codex / Antigravity / Cursor:
cp -r rook-skills/skills/rook-{platform}-{data-source} .agents/skills/
```

Both options install the same files. Use global paths (`~/.claude/skills/`, `~/.cursor/skills/`, …) instead of the
project-local ones to make a skill available across all your projects.

Each skill also ships an `INSTALL.md` with these exact commands pre-filled.

---

## Using a skill

**Recommended — invoke the skill explicitly.** Reference it by name with `/` so your agent uses exactly the right
skill from the start (most precise):

> */rook-android-health-connect Add ROOK to my Android app and sync daily steps and sleep.*

**Or ask in natural language** and let the agent load the matching skill automatically:

> *"Set up the ROOK Apple Health SDK and request permissions for heart rate."*

---

## Repository layout

```
rook-skills/
└── skills/                          # installable skills (npx skills auto-discovers here)
    ├── rook-android-health-connect/
    │   ├── SKILL.md                 # router: what it does, when to use, invariants
    │   ├── references/              # setup, permissions, sync, background, troubleshooting
    │   ├── examples/                # copy-paste golden-path snippets (optional)
    │   └── INSTALL.md               # per-tool install commands
    ├── rook-android-samsung-health/
    ├── rook-android-api-sources/
    ├── rook-ios-apple-health/
    ├── rook-flutter-health-connect/
    ├── rook-flutter-samsung-health/
    ├── rook-flutter-apple-health/
    ├── rook-flutter-api-sources/
    ├── rook-react-native-apple-health/
    ├── rook-react-native-health-connect-samsung-health/
    ├── rook-capacitor-apple-health/
    └── rook-capacitor-health-connect-samsung-health/
```

---

<sub>Built and maintained by ROOK. Skills target the current 4.x SDKs (LTS).</sub>
