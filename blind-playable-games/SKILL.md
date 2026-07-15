---
name: blind-playable-games
description: A standalone add-on layer that makes any game project playable by blind children. Applies on top of a new project (e.g. one built with game-dev) or retrofits an existing game; instantiates a docs/accessibility.md contract and enforces two independent audio channels, a self-voicing first-launch narration question, a global narration gesture, screen-reader detection, and icon rules. Use whenever a request mentions blind players, screen readers, or making a game accessible.
---

# Blind-Playable Game Rules

This skill is an ADD-ON LAYER, not a development process: it applies on
top of an existing game project — one just started (with
[`game-dev`](../game-dev/) or any other process) or one already shipped —
and makes that game playable by blind players. The base process must work
without it; this layer only ever adds.

The test behind every rule is the same: **"Can this game be played without
ever looking at the screen — by ear alone?"** Big-and-high-contrast design
serves low-vision players; a blind player needs more: the game itself must
speak. Distilled from building
[Sweet Pasta](https://popy.app/apps/sweet-pasta), an accessible kids'
restaurant game (Flutter + Flame), but the rules are engine-agnostic.

## Rule 1 · Two independent audio channels — never one switch

- **Game sounds** (effects/music, an `Sfx` service) and **narration**
  (TTS, a `Speech` service) are separate services with separate,
  separately-persisted settings.
- Muting the kitchen NEVER silences the narrator; the reverse holds too.
- The HUD speaker button controls **game sounds only**. Narration is
  controlled from settings + the global gesture (Rule 3).
- Game sounds default ON (a sighted child expects a noisy world);
  narration defaults OFF but is asked about OUT LOUD on first launch
  (Rule 2).
- Every effect goes through a single `Sfx.play()` gate — no call site
  checks the setting itself.

## Rule 2 · The blind player's first contact: the game asks, out loud

To a blind child, a visual button — including your speaker icon — does not
exist. Therefore:

- On first launch the game opens a **self-voicing** question screen and
  ASKS ALOUD whether narration should be on. This needs a `speakNow()`
  path in the TTS wrapper that speaks even while narration is off — it is
  the only channel that works before the choice is made.
- The answer geography requires no searching: **top half of the screen =
  yes, bottom half = no** — and the spoken instruction says exactly that,
  word for word.
- The question is asked exactly ONCE (persist a `narrationChoiceMade`
  flag); afterwards the setting lives in settings + the gesture.
- If a system screen reader (VoiceOver/TalkBack — in Flutter,
  `MediaQuery.accessibleNavigation`) is already active, skip the question
  and turn narration on directly.
- The question's spoken text also teaches the gesture ("double-tap the
  screen with two fingers…").

## Rule 3 · A global narration gesture — every screen, findable without sight

- **Double-tap with two fingers** toggles narration; WHERE on the screen
  doesn't matter.
- Mount it ABOVE the navigator (in Flutter: `MaterialApp.builder`) so it
  keeps working while any dialog or full-screen panel is open. Listen
  passively (a raw pointer `Listener`, not a competing gesture recognizer)
  so it never steals taps from buttons or the game.
- BOTH directions are confirmed out loud: the "narration off" sentence is
  itself spoken via `speakNow()` — an inaudible "off" strands a blind
  player with no way back — and it repeats how to turn it back on.

## Rule 4 · A spoken counterpart for every event

- Every game-state event produces a `(banner, spoken)` pair: a short
  high-contrast on-screen banner plus a full TTS sentence. All strings come
  from the localization table; the TTS voice follows the active language.
- ONE central listener speaks announcements (an `announcement` +
  monotonically increasing `announcementId` pair on the game state) —
  widgets never speak on their own, so overlapping speech is impossible.
- Every interactive element carries a semantic label that states status +
  action together ("Game sounds are on. Turn off.").
- Strip emoji before TTS — the narrator must read words, not symbol names.

## Rule 5 · Icons instead of words; never color alone

- Button faces carry no words like "Close"/"Back" — non-readers and
  low-vision players get one bold, single-object icon (✕, ←, gear,
  speaker); the word lives in the semantic label and the narration.
- In on/off icon pairs the difference is never color alone: the off state
  gets a thick diagonal slash + a desaturated body (double coding).
- The game's full icon set is listed in the art brief with generation
  prompts (sound on/off, narration on/off, settings, back, close included).

## Rule 6 · Memorizable touch geography

- The layout is FIXED: the HUD always in the same order at the top, the
  main action strip always at the bottom. A blind player plays by
  memorizing the screen map — elements must never move between sessions.
- Touch targets ≥ 96dp; full-screen choices use giant halves/quadrants
  (no hunting for small targets).
- Time pressure never punishes: a player who explores slowly loses only a
  bonus, never the game.

## Rule 7 · Verification — the eyes-closed run

Run this checklist every release:

1. **Eyes-closed run:** can you complete one full game loop without
   looking, guided only by TTS?
2. Is the first-launch question still understandable with the device
   muted (vibration/visual fallback)?
3. VoiceOver/TalkBack smoke pass: does every element read a meaningful
   label?
4. Widget tests: the first-launch question, the one-time-only flag, the
   gesture toggle (both directions), and channel independence
   (muting sounds must not touch narration).
5. Does TTS speak with the correct locale in every supported language?

## Project-start format

At the start of every new game, create `docs/accessibility.md` from the
template below and fill it in. The design doc references it; in code
review, this table is treated as a contract.

```markdown
# <Game> — Accessibility Contract

Target: low-vision children first-class; blind children playable.

## Audio channels
| Channel | Service | Default | Control points |
|---|---|---|---|
| Game sounds | Sfx | ON | HUD speaker + settings |
| Narration | Speech (TTS) | OFF, asked aloud on first launch | first-launch question + settings + two-finger double tap + screen-reader detection |

## Blind-player flow
- First-launch question: <question text + top/bottom-half choice>
- Global gesture: two-finger double tap → narration on/off (both spoken)
- Screen reader active: question skipped, narration auto-on
- Memorized map: <HUD order / region layout — what is always where>

## Spoken counterparts
- Event → (banner, spoken) table: <game-specific event list>
- Interactive elements without a semantic label: NONE (contract)

## Icons
- Icon set + prompts: docs/art-brief.md §<section>
- Buttons with words on their face: NONE; on/off double-coded (color+shape)

## Eyes-closed run results
- Date / version / findings: <fill in>
```

## How to layer it

This skill has no hard dependency on any base process. Two ways to use it:

- **New project (e.g. with [`game-dev`](../game-dev/)):** when discovery
  puts blind players in scope, instantiate `docs/accessibility.md` while
  the design doc is written; the audio-channel split, the first-launch
  question, and the gesture enter the scaffold from day one. game-dev is
  complete without this layer — add it only when blind playability is a
  goal.
- **Retrofit an existing game:** apply Rules 1→7 in order — split the
  audio channels first, then first contact + gesture, then spoken
  counterparts and icons; write the `docs/accessibility.md` contract and
  close with an eyes-closed run.

While this layer is active, its rules stack on top of the base process's
accessibility principles — where they conflict, this layer wins.
