# Meridian Office Park — "Meridian Move"

> A product brief for the development team. Everything here is fictional.

## The problem

Meridian Office Park operates a large commercial office complex. Our occupational-health surveys keep
telling us the same thing: people sit at their desks for most of the day, work long hours, and their health
pays for it — poor sleep, low activity, rising stress. Tenant companies are asking us what we can do about
it, because healthier employees mean lower turnover and fewer sick days for them.

## The idea

**Meridian Move** — a voluntary wellness app we offer to the people who work in the complex. It quietly
keeps track of each person's activity and rest, and nudges them at the right moments: stand up, take a
walk, wind down. Nothing invasive, nothing they have to babysit. They opt in, connect once, and the app
just works in the background.

We win when an employee opens the app at the end of the day and sees that it noticed their 9,000 steps and
their short night of sleep — without them ever having to press "sync".

## Who will use it

Employees across the complex, on their own phones. They own a mix of devices, and that mix matters:

- **Plenty have no wearable at all — just their phone.** For these people the phone itself is the tracker:
  it can count their steps as they walk around the complex. We can't leave this group out; they may be the
  majority.
- Many have an Android phone paired with some wearable (a Fitbit, a Xiaomi band, whatever) that already
  collects their steps and sleep.
- A large group are Samsung users with Galaxy phones and Galaxy Watches.
- Some wear brands that live mostly in their own cloud apps (Garmin, Oura, Polar).

We don't want to tell anyone to switch devices or buy a tracker. Whatever they already have — even if it's
only their phone — Meridian Move should pick up what it can.

## What the app needs to do (MVP)

1. Let an employee sign in. In real life we have an employee identity system, so each person has an ID — but
   **for this demo, keep the login local**: no backend, no real authentication. A simple screen where the
   person types (or picks) their employee ID and it's stored on the device is enough. That ID is what
   identifies them for their health data.
2. Let them connect their health data in one simple flow, with a clear explanation of what we'll read and
   why, and ask their permission respectfully.
3. Keep their activity and sleep data flowing on its own, in the background, with **no manual refresh** —
   this is the whole point.
4. Show a simple daily dashboard: today's steps, and their most recent sleep and heart-rate summary.

The nudges, the charts, the tenant-company reports — all of that comes later. For the first version we just
need the data to arrive, reliably, for whatever device the employee has.

## The data we care about

Steps and active time, sleep, and heart rate. That's enough to power the first round of wellness nudges.

## Brand & design (final handoff from the UI/UX team)

This has already been through design. The direction below is settled — build to it.

### Brand personality

Calm, credible, quietly encouraging. Meridian Move is a **corporate wellness companion**, not a hardcore
fitness app. It never shouts, never shames. Think "a supportive colleague who reminds you to take a walk,"
not "a coach yelling at you to hit your macros."

### Name & logo

- **App name:** Meridian Move. **Operator:** Meridian Office Park.
- **Logo:** the wordmark "Meridian Move" beside a simple mark — a thin arc (a meridian line) rising over a
  flat horizon with a small filled dot (a midday sun) resting at the arc's peak. It reads as *balance* and
  *midday reset*. Keep the mark single-colour; it should work on both light and dark backgrounds.

### Colour palette

| Role | Name | Hex |
|------|------|-----|
| Primary | Meridian Teal | `#0E6E6E` |
| Accent / calls-to-action / nudges | Sunrise Amber | `#F4A63A` |
| Background | Horizon Sand (warm off-white) | `#F7F5F0` |
| Surface / cards | White | `#FFFFFF` |
| Primary text | Slate | `#2B2F33` |
| Secondary text | Muted Grey | `#8A9299` |
| Positive / on-track | Soft Green | `#3FA34D` |

Teal leads; amber is used sparingly for the one primary action on a screen and for gentle nudges. Lots of
whitespace, soft rounded corners (cards ~16dp radius).

### Typography

A clean humanist sans (Inter, or the platform system font as fallback). Large, friendly headings; generous
line spacing; nothing cramped. Numbers on the dashboard (step count, hours slept) are the visual hero —
show them big.

### Tone of voice

Short, warm, first-person-plural where it helps ("Let's connect your data"). Nudges are invitations, not
orders: "Good time to stretch your legs?" rather than "You've been sitting too long."

### Screens (MVP)

Five screens. Visual mockups (modern, platform-agnostic) are in
[`office-wellness-mockups.html`](office-wellness-mockups.html) — layout intent, not pixel spec.

1. **Welcome / local sign-in** — first launch. Enter the employee ID; stored on the device (no backend).
   One primary action ("Continue") and a privacy reassurance line.
2. **Connect your health data** — choose how the app gets their data, presented as options: *just my phone*
   (steps), *my wearable / phone* (steps, sleep, heart rate), *Samsung Health*, and *cloud brands* (Garmin,
   Oura, Polar…). Only the options the device supports should feel available.
3. **Before we connect** — a short, honest explainer shown *before* the system permission prompt: what we
   read (steps & active time, sleep, heart rate), why, and that they can disconnect anytime. "Allow access"
   / "Not now".
4. **Today (dashboard)** — the home screen. Today's step count as the hero number with a goal, small cards
   for last sleep and heart rate, and one gentle nudge. Simple Today / Settings navigation.
5. **Settings** — the opt-in toggle for automatic background updates, the connected source (with a way to
   change it), a privacy link, and "Disconnect my data".

## What we need from you

- Build **Meridian Move** as a native Android app.
- Use **ROOK** as our health-data platform — we've chosen it so we don't have to build integrations with
  every wearable ourselves. You'll get test (sandbox) access to set things up; we'll hand over the real
  production access when we're ready to launch.
- Support the ways our employees' data can reach us, so nobody is left out:
  1. **phone-only employees** — count their steps using the phone itself, no wearable required,
  2. data from a wearable that already lives on the employee's Android phone via the phone's health hub,
  3. Samsung devices (our Samsung users are a big chunk, and we want their data to be as accurate as
     possible),
  4. cloud-only brands that connect through the employee authorizing their account.
- Set expectations honestly per segment: a phone-only employee will get step counts, while someone with a
  wearable can also get sleep and heart rate. That's fine — the point is that everyone gets *something*.
- Make connecting feel trustworthy: explain the permissions, let people opt in, and let them back out.

You can start with the wearable-on-the-phone path to get an end-to-end version working, then add phone-only
step tracking, Samsung, and the cloud-account paths.

## What success looks like

A Meridian employee downloads Meridian Move, signs in, taps "Connect my health data", approves once, and
never thinks about it again — while their steps and sleep quietly show up on the dashboard day after day. If
we can demo that on a test phone with a sandbox account, we have our MVP.
