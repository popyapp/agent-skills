---
name: game-dev
description: Accessibility-first kids' game development process for Flutter + Flame. Asks a few discovery questions, then runs a proven end-to-end pipeline - design doc first, deterministic simulation architecture, overflow-guarded tests, an AI art pipeline (Gemini), TTS narration + localization from day one, simulator verification loops, and App Store prep. Use for "let's build a game", "new kids' game", "make my game accessible".
---

# Accessible Kids' Game Dev (Flutter + Flame)

A battle-tested process for taking a children's game from idea to
App-Store-ready — with accessibility (low-vision players first) as a design
constraint, not an afterthought. Distilled from building
[Sweet Pasta](https://popy.app/apps/sweet-pasta), an accessible pasta
restaurant game.

Work in **small waves**: change → tests → simulator build → the user plays
and gives feedback. Never batch a week of work into one unverifiable drop.

## 0 · Discovery (ask first, max ~4 questions)

1. **Concept + genre** — one sentence ("a pasta restaurant sim"). If the
   user can't describe the core loop, propose one.
2. **Audience + accessibility bar** — age range; are low-vision / blind /
   deaf players first-class? (Recommended default: low-vision kids are a
   primary audience.)
3. **Platform + orientation** — tablet/phone, landscape/portrait, iOS/Android.
4. **Art style** — soft clay 3D (AI-generated, recommended), vector, or the
   user's own art. Languages (recommend at least two from day one).

## 1 · Design doc before code: `docs/design.md`

Write and share a design doc covering: a non-negotiable accessibility
principles table, the core loop, a content catalog, an **economy/strategy
dynamic** (make it honestly educational — real-world trade-offs like
cheap-but-slow vs instant-but-pricey, wages, operating costs), an ASCII
screen layout, art direction + logo concept, and an M1→M4 roadmap.

In the same step, open the [`blind-playable-games`](../blind-playable-games/)
skill and instantiate its template as `docs/accessibility.md` — the contract
for two independent audio channels, the self-voicing first-launch narration
question, the global narration gesture, screen-reader detection, and icon
rules. The design doc references that contract.

### Accessibility principles (non-negotiable in every game)

- **Double coding:** no information travels by color alone — every item gets
  a distinct silhouette + color + written (and spoken) name.
- Touch targets ≥ 96dp; body text ≥ 20pt at weight 700+.
- Status indicators are **big faces/icons**, never thin bars.
- Gentle pacing: incentives instead of punishments (customers never storm
  out; a slow serve just trims the tip).
- A `Semantics` label on every interactive element; a speakable sentence for
  every game event.

## 2 · Scaffold

```bash
flutter create <game> --org <your.org> --project-name <game> --platforms ios,android
cd <game> && flutter pub add flame provider flutter_tts shared_preferences
```

Orientation lock (SystemChrome alone is NOT enough):
- iOS: `Info.plist` → `UISupportedInterfaceOrientations(-~ipad)` restricted
  to your orientations + `UIRequiresFullScreen=true`.
- Android: manifest activity `android:screenOrientation="sensorLandscape"`.
- Note: iOS bundle IDs forbid underscores (use `com.org.myGame`); Android
  application IDs allow them.

## 3 · Architecture rules

- **One `GameState extends ChangeNotifier`** owns the whole simulation.
  `tick(double seconds)` is parametric; the constructor takes `seed` and
  `autoTick: false` so tests are DETERMINISTIC. Everything timed (cooking,
  deliveries, patience) advances inside tick.
- **Immediate-mode Flame scene:** one `PositionComponent` reads GameState
  every frame and paints everything — at small-scene scale this makes
  component/state desync bugs impossible. Panels are plain Flutter widgets
  stacked over the `GameWidget`.
- **`l10n.dart` from day one:** an abstract `Strings` class with one
  implementation per language; announcements are `(banner, spoken)` record
  pairs; `AppLanguage { system, xx, yy }`. Catalogs hold no display names —
  all names come from L10n.
- **`speech.dart`:** a flutter_tts wrapper. GameState exposes
  `announcement` + a monotonically increasing `announcementId`; ONE central
  listener in the root screen speaks each event exactly once. Narration is
  opt-in; the TTS voice follows the active language.
- **`settings.dart`:** shared_preferences; text size has exactly TWO options
  (small/big — a binary choice a child can make) applied app-wide through a
  `MediaQuery` textScaler override in `MaterialApp.builder`; plus language
  and narration toggles.
- Background work (cooking and the like) must NOT pause the game: show a
  filling progress line on the card and in the scene; scale durations with
  content complexity.
- Phone + tablet: design tablet-first at fixed sizes, then add one compact
  breakpoint (`MediaQuery.sizeOf(context).height < 500`). Scale card-like
  widgets as whole blocks with `FittedBox` around a fixed design size —
  one wrapper instead of dozens of per-widget breakpoints.

## 4 · Test discipline (after every wave)

- Logic tests drive `tick` directly: scoring, economy, timers, day cycle.
- **Layout overflow tests:** pump real screens at tablet size (1366×1024),
  assert `tester.takeException() == null`; add a big-text variant
  (`textScaleFactorTestValue = 1.22`) AND a landscape-phone variant
  (874×402). These catch real overflows constantly — non-negotiable.
- Pin tests to one language in `setUpAll` so copy edits don't break them.
- Every wave ends with `flutter analyze` clean + all tests green, then a
  simulator check.

## 5 · Simulator verification loop

```bash
xcrun simctl boot <UDID>
flutter build ios --simulator && xcrun simctl install <UDID> build/ios/iphonesimulator/Runner.app
xcrun simctl launch <UDID> <bundle-id>
xcrun simctl io <UDID> screenshot out.png
```

Inspect the screenshot yourself; then let the user play and report. Ship the
next wave only after both.

## 6 · AI art pipeline (Gemini)

1. Write `docs/art-brief.md`: a **style-anchor paragraph** (soft 3D clay
   render, single top-left light, plain pure-white background, no text, 1:1)
   pasted at the top of EVERY prompt, plus a table of per-asset filenames and
   prompts with your palette's hex codes embedded.
2. Generate all assets in ONE chat session ("same exact style as the
   previous images") so the set stays coherent.
3. Process: flood-fill transparency from all four corners (backgrounds are
   often NOT pure white):
```bash
magick in.png -fuzz 6% -fill none -draw 'alpha 1,1 floodfill' \
  -draw 'alpha 2046,1 floodfill' -draw 'alpha 1,2046 floodfill' \
  -draw 'alpha 2046,2046 floodfill' -trim +repage -resize 480x480 \
  -background none -gravity center -extent 512x512 assets/images/out.png
```
4. Verify transparency with a montage on a grey background before wiring in.
5. Character renders often arrive with scene props (a table, a chair) —
   use that as the composed sprite instead of fighting it.
6. App icon: `flutter_launcher_icons` — an opaque icon (`-alpha remove`)
   plus an Android adaptive foreground (transparent emblem at ~60% of a
   2048 canvas).

## 7 · Marketing screenshots without hand-driving

Add a `--dart-define=SHOTS=true` mode: a `debugStageScene()` on GameState
composes a rich moment (several actors in different states), then a timed
script opens each panel on a fixed schedule. Capture with
`xcrun simctl io <UDID> screenshot` at the known timestamps — reproducible
store screenshots in any language (`--dart-define=SHOTS_LANG=en`).

## 8 · Store prep checklist

- `ITSAppUsesNonExemptEncryption=false` in Info.plist.
- Verify `flutter build ios --release` (and later `flutter build ipa`).
- Landing + privacy + support pages; if the game collects nothing, say so
  plainly (and it should collect nothing — it's a kids' game).
- Roadmap template: **M1** core loop end-to-end with placeholder art +
  tests · **M2** art integration + economy depth + TTS foundation ·
  **M3** localization, settings, popup scaling pass · **M4** sound,
  persistence, store screenshots and submission.
