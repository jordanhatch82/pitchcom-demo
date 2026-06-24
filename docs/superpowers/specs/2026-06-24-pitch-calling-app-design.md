# Pitch & Play Calling App — Design Spec

- **Date:** 2026-06-24
- **Status:** Approved design (pre-implementation)
- **Topic:** Native iOS + watchOS app for a coach to call pitches, locations, and plays to players' Apple Watches during a game.

---

## 1. Overview

A native **iOS + watchOS** system that lets a coach call **pitches, locations, and plays** from a phone and have them appear instantly on **role-specific Apple Watch displays** worn by players during a live game. Calls travel over a **cloud websocket backend** (not push notifications), so delivery is sub-second and reliable as long as devices have any network. A companion **management layer** (teams, rosters, pitchers, pitch menus, presets, games) is used before/around games.

The build strategy is **"start for my team, architect toward a product"**: ship something usable by a single coaching staff first, but keep the data model and backend multi-tenant-clean so a wider product is not a rewrite.

---

## 2. Goals

- **Reliable, low-latency live calls** — a call must reach the watches within ~1 second, every time, in the ~10–20s window between pitches.
- **Glanceable, role-specific displays** — a player understands the call in under ~2 seconds, in bright sun, with a haptic alert so they know a new call arrived.
- **Fast one-handed coach calling UI** — composing and sending a call takes a couple of taps, eyes mostly on the field.
- **Acknowledgment loop** — the coach can see that players' watches received the call.
- **Team/game management** — teams, rosters, pitchers (with allowed pitch types), presets, and starting/ending games.

## 3. Non-Goals / Out of Scope (v1)

- **Enforcing league legality.** Whether electronic communication is permitted is the **coaches' responsibility**, not the app's.
- **Automatic game-state tracking** (count, outs, baserunners). Too much game-time burden. We keep only a **one-tap "where's the play"** base indicator.
- **Dead-zone (no-internet) transport.** The cloud-websocket model assumes the server is reachable. A local-broker fallback is a documented **future** option, not v1.
- **Audio / earpiece delivery.** v1 is **visual + haptic** only.
- **Full multi-tenant productization** (onboarding, App Store polish, support). We only keep the *seams* clean.

---

## 4. Users & Roles

- **Head coach** — the caller. Composes and sends calls; manages team/game.
- **Assistant coaches** — *future* secondary callers (single active caller in v1 to avoid conflicting signals).
- **Players, grouped by position:**
  - **Battery** — pitcher + catcher
  - **Infield**
  - **Outfield**

Each call carries components; **each watch renders only the slice relevant to its role.**

---

## 5. Architecture

### 5.1 Transport (the critical decision)

- **Cloud websocket server.** Every device (coach phone + each watch) holds a **persistent websocket** to the server. The coach publishes a call; the server fans it out down the open sockets.
- **Path:** `coach phone → network → server → network → watch`.
- **Network availability is separate from brokering.** Devices reach the server over **whatever connection they have** — a watch's own LTE, or a **shared hotspot/wifi the coach provides** at the field.
- **Explicitly NOT push notifications (APNs).** Push can be delayed, coalesced, or dropped — unacceptable for live calling.
- **Assumption:** the server is reachable (the coach's shared connection or watch LTE has at least a trickle of internet). The only unsupported case is a true dead zone where *no one* has signal. This does not affect the data model, so a **local-broker fallback can be added later** without redesign.

### 5.2 Components

1. **watchOS app (player)** — connects to the server, receives **role-filtered** calls, renders the glanceable display, buzzes (haptic) on a new call, and sends an **acknowledgment**.
2. **iOS app — coach mode** — the calling UI + the management hub.
3. **iOS app — player/parent mode** — used to set up and pair the player's watch and join a team/game. (See connectivity note.)
4. **Backend** — accounts, teams, games, rosters, pitchers, pitch menus, presets; realtime fan-out; acknowledgment tracking.

### 5.3 Apple Watch pairing note (constraint)

An Apple Watch pairs to **exactly one iPhone**. The coach's phone **cannot** Bluetooth directly to arbitrary players' watches. Therefore each player's watch reaches the server via **its own paired iPhone and/or its own LTE** — which is exactly why the live path goes through the cloud rather than phone-to-watch locally.

---

## 6. Data Model (high level)

> Proposed entities; field names are illustrative.

- **Team** — `id`, `name`, `level`, `members[]`
- **User** — `id`, `name`, `role` (coach | player), `positions[]`
- **RosterEntry** — `playerId`, `name`, `positions[]`, `bats`/`throws` (L/R)
- **PitcherProfile** — `playerId`, `throws` (L/R), `allowedPitchTypes[]`
- **PitchType** — `code` (e.g. FB/CB/CH/SL), `label`, `color`
- **Preset** — `pitcherId`, `pitchType`, `location` (zone), `label` (e.g. "CB Low")
- **Play** — `code`, `label`, `category` (pitchout | pickoff | bunt-D | positioning | IBB | …), `targetRoles[]`, optional `base`
- **Game** — `id`, `teamId`, `opponent`, `date`, `activePitcherId`, `connectedDevices[]`, `state` (active | ended)
- **Call** (the live message) — `gameId`, `timestamp`, `callerId`, optional `pitch{type, location}`, optional `play{code, base?}`, optional `playAt{base}`, `targetRoles[]`
- **Ack** — `callId`, `deviceId`/`playerId`, `receivedAt`

**Location** = a 9-zone grid (3×3): in/middle/out × high/middle/low.

---

## 7. Displays (role-filtered)

A single call fans out; the watch shows only its role's components.

- **Battery (pitcher/catcher)** — **grid-dominant** layout: the 9-zone grid fills the screen with the target cell highlighted and the **pitch abbreviation inside it**; pitch name on a foot label. A called play appears as an **orange banner** on top (e.g. "PICK · 1B").
- **Infield** — a **baseball diamond** icon with the relevant base lit + a short label (e.g. "PICK / 2ND BASE").
- **Outfield** — **positioning** call: fielder dots pushed to depth/shift + a label (e.g. "NO DOUBLES / PLAY DEEP").

All displays: large high-contrast type, color-coded, **haptic buzz** on arrival, and an idle "Waiting for coach" state when no game is active.

---

## 8. Screens

### 8.1 Coach phone — Management Home (hub)
- Team switcher; large **Start Game** button; "last game" line.
- **Manage** tiles: **Roster, Pitchers, Presets, Schedule.**
- Bottom tabs: Home / Games / Team / Settings.
- Setup flows reached from here: **pitcher pitch menus** (which pitches each pitcher can throw) and the **presets builder** (per pitcher: pitch + location combos).

### 8.2 Coach phone — In-game calling
- **Top bar:** a **Home icon** (always-visible affordance, top-left), active pitcher name, and the **acknowledgment count** ("✓✓ 3 watches").
- **Quasi-tabbed view:** **Pitch / Plays / Play At** — the tabs and Send are shared; only the middle body swaps.
  - **Pitch tab:** **3 presets** across (aligned to the grid columns) that each arm **pitch + location**; a **pitch picker** (pills) to choose the pitch manually; the **9-zone location grid**; a **small, centered Send**. Preset = one-tap shortcut; pills + grid = manual path. Send is explicit (no accidental mis-fires).
  - **Plays tab:** a grid of play calls (Pitchout, Pick 1B/2B/3B, 1st-3rd D, Bunt D, No Doubles, IBB…) + Send.
  - **Play At tab:** a diamond with base buttons (1B/2B/3B/Home) for the one-tap "where's the play"; fielders' diamonds light up to match. + Send.
- **In-game menu sheet** (opened by the **Home icon** or tapping the status bar): game info (opponent, inning, watches connected), **Change pitcher**, **Defense / subs**, **Game settings**, **End game → Home** (red), and **Resume game**. Exit is **deliberate** — End Game is a confirmed second tap so you can't fat-finger out mid-pitch.

### 8.3 Watch (player)
- **Idle** "Waiting for coach" when no active game.
- **Live call display** per role (see §7).
- **Auto flip** between idle and live when the coach starts/ends the game — **no per-player navigation**.
- Sends an **acknowledgment** on receipt (drives the coach's ✓ count).

---

## 9. Key Flows

1. **Setup (pre-season / pre-game):** coach creates team + roster, defines each **pitcher's allowed pitches**, builds **presets**.
2. **Pre-game:** players join the team/game; watches connect to the server; coach taps **Start Game** (watches flip to live).
3. **Live call:** coach composes a call (preset, or pitch + location, or a play, or where's-the-play) → **Send** → server fans out **role-filtered** → watches **buzz + display** → **acks** return → coach sees the ✓ count.
4. **Mid-game changes:** change pitcher / make subs via the in-game sheet.
5. **End game:** coach ends the game → watches return to **idle**.

---

## 10. Reliability & Error Handling

- **Persistent sockets with auto-reconnect.** Per-watch connection state is surfaced to the coach via the **acknowledgment count** — if a watch drops, the count falls and the coach notices.
- **Latest-call-wins.** The watch always shows the most recent call; older calls are superseded.
- **Freshness.** The watch should make it obvious when a call just arrived (haptic + visual change) vs. is stale.
- **Coach connectivity loss** is shown clearly (the coach should know if the server is unreachable rather than silently failing).
- **Deliberate exit** prevents accidental game-end.

---

## 11. Future / Path C (architect toward product)

- Multi-tenant accounts, onboarding, App Store distribution, support.
- **Secondary callers** (assistant coaches) with single-active-caller conflict handling.
- **Local-broker transport fallback** for true dead-zone fields.
- **Audio / earpiece** delivery option (à la PitchCom).
- **Scorekeeping integration** to provide situational context automatically.
- **Sign security** considerations (rotating/obscuring calls).

---

## 12. Open Questions

- **Watch fleet logistics** — coach-provided watches vs. player-owned? Charging, distribution/collection, and "is it set up & paired" each game.
- **Which devices reach the server** — every watch on its own LTE vs. all watches piggybacking on the coach's hotspot. Needs **field testing** to confirm the reliable default.
- **Handedness-aware location** — "in/away" flips between left- and right-handed batters. Decide whether the coach calls **absolute** zones (catcher's perspective) or the UI **mirrors** based on a batter-side toggle.
- **Vocabulary** — exact pitch and play sets, and whether the app targets **baseball, softball, or both** (affects pitch types and defaults).
- **Sport terminology** in presets/menus should be configurable per team.

---

## 13. Mockups

Interactive mockups produced during brainstorming live in
`/.superpowers/brainstorm/<session>/content/`:
`battery-display.html`, `fielder-display.html`, `coach-ui.html`, `coach-ui-v2.html`, `coach-ui-v3.html`, `nav.html`, `nav-v2.html`.
