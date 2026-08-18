# ROOK SDK AI Skills — Shared Conventions

> **Status:** Proposed standard · **Audience:** SDK teams (Android, iOS, Flutter, React Native, Capacitor)
> **Purpose:** Ship one consistent set of AI "skills" across every ROOK SDK family so our clients get the same
> experience regardless of platform or which AI coding tool they use.

This is an **internal** document. The **skills themselves are for external clients** — see [Hard rules](#hard-rules).

---

## 1. The decision

We ship **AI Agent Skills** (the `SKILL.md` standard) — **not** a giant markdown dump and **not** a loose prompt
collection.

Why skills:

- **Progressive disclosure** — the agent loads only the file it needs, not the whole SDK doc set.
- **Auto-triggering** — the `description` fires when the client's prompt mentions ROOK / the platform / the data source.
- **Tool-agnostic** — the same skill drops into Claude, Codex, Antigravity, Cursor, etc. Only the install
  directory differs (see [§6](#6-distribution--install)).

**Granularity: one skill per data source, per platform.** The shared "core" layer (auth, config, environments,
user management) is **folded into each skill** so every skill is fully self-contained — no cross-skill dependencies,
one thing for the client to install.

---

## 2. Skill inventory

Naming scheme: **`rook-{platform}-{data-source}`**, all lowercase kebab-case.

| Platform | Skills |
|---|---|
| **Android** | `rook-android-health-connect`, `rook-android-samsung-health`, `rook-android-api-sources` |
| **iOS** | `rook-ios-apple-health` |
| **Flutter** | `rook-flutter-health-connect`, `rook-flutter-samsung-health`, `rook-flutter-apple-health` |
| **React Native** | same rule — one skill per data source the family supports *(RN team to confirm sources)* |
| **Capacitor** | same rule — one skill per data source the family supports *(Capacitor team to confirm sources)* |

> There is **no** standalone `core` skill. Flutter's `rook_sdk_core` content is folded into each Flutter skill.

---

## 3. Skill folder structure

Every skill uses the same layout:

```
rook-{platform}-{data-source}/
  SKILL.md            # router: what it does, when to use, decision notes, the naming invariants
  references/
    setup-and-init.md # FOLDED CORE: install (gradle/spm/pubspec), config, environments, user init
    permissions.md    # availability + permission flow
    sync.md           # manual + automatic/background sync
    background.md      # background delivery / steps / calories (if applicable)
    troubleshooting.md # known exceptions + best practices  ← highest value for AI debugging
  examples/
    quickstart.<ext>   # 1–3 copy-paste golden-path snippets
  INSTALL.md          # per-tool install steps + the CTA text used in docs (see §6)
```

Keep `SKILL.md` short (a router + decision guide + invariants). Everything detailed lives in `references/` and is
pulled in on demand.

---

## 4. SKILL.md frontmatter

```yaml
---
name: rook-android-health-connect
author: ROOK
description: >
  Integrate the ROOK Android Health Connect SDK: set up credentials, register users, request permissions,
  and sync health summaries and events (steps, calories, sleep, heart rate). Use when a developer is adding
  ROOK, Health Connect, or wearable/health-data sync to an Android app.
---
```

`author` is always `ROOK`.

Convention for `description` (this is what makes the skill trigger correctly):

- Write it in the **third person**, present tense.
- State **what it does** *and* **when to use it**.
- Include the trigger keywords a client would actually type: `ROOK`, the platform, the data source, plus
  `health data`, `wearables`, `permissions`, `sync`.
- Keep it tight (roughly one to three sentences).

---

## 5. Hard rules

These apply to **every** skill, on **every** platform. They exist because the skills are shipped to **external
clients** and are consumed by autonomous agents.

1. **Self-contained.** No links to internal ROOK repos, playbooks, dashboards, or Jira. No references to other
   skills. A client with only this one folder must be able to complete the integration.
2. **No secrets, ever.** Code samples use placeholders (`CLIENT_UUID`, `SECRET`) — never a real `client_uuid`,
   secret, token, or `.env` value.
3. **Naming invariants must be correct** (agents get these wrong constantly):
   - `client_uuid` — the client identifier, **never** `customer_id`.
   - `user_id` - the identifier if the app end-user whose data will be synced.
   - `environment` — the SDK environment is set to `SANDBOX` or `PRODUCTION` (e.g. `RookEnvironment.SANDBOX`);
     each environment has its own credentials.
4. **Docs are the single source of truth.** Skill content is derived from `docs/ROOKConnect/SDKs/<Platform>/`.
   When the docs change (versions, coordinates, API), the skill is updated in the same change — never hardcode a
   version that will drift.
5. **Runnable snippets.** Examples should compile/run as-is against the current 4.X.X SDK version (LTS).

---

## 6. Distribution & install

**Goal:** a client reading an SDK's Quickstart / Introduction sees a call-to-action and adds the matching skill to
their AI tool in a minute — with **zero setup on our side beyond pushing to a public GitHub repo**.

**Chosen path: a plain public repo, installed with the `skills` CLI ([`npx skills`](https://github.com/vercel-labs/skills)),
with classic clone + copy as the fallback. No plugin marketplace.**

The `skills` CLI (Vercel Labs, open source) is **skill-aware**: it knows the correct install directory for 75+ agents
(Claude Code, Cursor, Codex, Cline, OpenCode, …), understands the `SKILL.md` format, and can install a single skill
out of a multi-skill repo. That means **one command works for every tool** — we don't document a different target
path per agent — and it needs **no special setup from us**: just plain `SKILL.md` folders in a public GitHub repo.

> We chose this over `git`/`degit` copies (which are agent-blind and force a per-tool target path) and over Claude's
> plugin marketplace (which would require authoring a `.claude-plugin/marketplace.json` + a plugin manifest per skill
> + a shared file all five teams edit). `npx skills` gives the same friendly cross-tool install with none of that
> setup, so the marketplace is **not on the roadmap**.

### The V1 approach

All skills live in **one public GitHub repo** (`rook-skills`) under a `skills/` directory, **pushed as-is** — nothing
else on our side. A client installs one skill with a single command; the CLI detects/prompts for their agent and
writes to that agent's skills directory automatically:

```bash
npx skills add RookeriesDevelopment/rook-skills --skill rook-{platform}-{data-source}
```

To target agents explicitly (and skip prompts), add `-a` and `-y`:

```bash
npx skills add RookeriesDevelopment/rook-skills --skill rook-{platform}-{data-source} -a claude-code -a cursor -y
```

**Fallback — classic clone + copy.** Always supported, for clients who can't or prefer not to run `npx`
(no Node.js, locked-down/offline CI, or just preference). Still zero setup on our side — plain git, copy the skill
folder into the agent's skills directory:

```bash
git clone --depth 1 https://github.com/RookeriesDevelopment/rook-skills

# Claude / Claude Code:
cp -r rook-skills/skills/rook-{platform}-{data-source} .claude/skills/
# Codex / Antigravity / Cursor:
cp -r rook-skills/skills/rook-{platform}-{data-source} .agents/skills/
```

Both install paths pull from the **same repo folders**, so they never diverge — document both in each skill's
`INSTALL.md`.

### Repo layout

```
rook-skills/
  skills/                              # `npx skills` auto-discovers skills here
    rook-android-health-connect/
      SKILL.md
      references/ …
      examples/ …
      INSTALL.md                       # the npx skills command + the git fallback
    rook-ios-apple-health/
      …
```

### Docs CTA

Each SDK's Quickstart/Introduction page gets a banner linking to *its* skill — the same one command for all tools:

> ⚡ **Accelerate your integration with AI.** Add the `rook-{platform}-{data-source}` skill to your AI coding tool
> (Claude, Codex, Antigravity, Cursor…) and let it scaffold the integration for you:
> ```bash
> npx skills add RookeriesDevelopment/rook-skills --skill rook-{platform}-{data-source}
> ```

---

## 7. Open items for the teams

- **React Native / Capacitor:** confirm which data sources each family ships, then apply the naming rule.
- **Hosting repo:** confirm `rook-skills` (public GitHub) as the canonical location and who owns it.
- **Update cadence:** agree that a docs PR touching an SDK also updates that SDK's skill in the same PR.

---

*Each team builds its own SDK's skill set in parallel, following this document as the shared standard. Consistency
comes from these conventions — not from one platform going first.*
