# Webcam Gesture Fighter (v2)

A two-player fighting game for the browser, controlled **only by a single laptop webcam**. There is no keyboard, mouse, controller or touch combat. Players punch toward each other, and hand tracking (MediaPipe) turns each punch into HP damage.

The spec is in [`brd.md`](brd.md). The whole game is in `index.html`. MediaPipe is loaded from a CDN, so you need an internet connection.

## Running it

The camera only works on a secure origin, so serve the folder locally:

```bash
python3 -m http.server 8000
```

Then open <http://localhost:8000> in **Chrome or Edge** (Safari works but is slower).

Opening `index.html` directly (`file://`) usually works in Chrome, but a local server is more reliable.

On load the game automatically:
1. loads the MediaPipe hand and pose models
2. asks for camera permission, so click **Allow**

If you blocked the camera: click the camera or 🔒 icon in the address bar, allow the camera, then press **RETRY CAMERA**. On macOS, also check *System Settings → Privacy & Security → Camera*.

Click anywhere once to enable sound, because browsers block audio until you interact with the page.

## Positioning

The preview is mirrored, like a mirror.

| | Stands on | Attacks toward |
|---|---|---|
| **Player 2** (magenta) | **LEFT** half of the screen | → RIGHT |
| **Player 1** (cyan) | **RIGHT** half of the screen | ← LEFT |

- Stand about 1–2 m from the laptop, side by side, each in your own half. Your upper body and hands must be visible.
- Use good front lighting and a calm background.
- On the **GET READY** screen, move your hands. Each player shows **✓ DETECTED** when found. Practice punches there show `PUNCH ✓ 92%` but do no damage.
- The match starts automatically once both players have been detected for 1.5 s: **STARTING… → 3 → 2 → 1 → FIGHT!**

## How to attack

**Punch or karate-strike sideways toward your opponent**: fast, mostly horizontal, then pull your hand back.

| Move | Gesture | Damage |
|---|---|---|
| **Punch** | Fast strike toward the opponent (P1 ←, P2 →) | **17** |
| Heavy strike *(advanced)* | Very fast and long strike toward the opponent | 25 |
| Special *(advanced)* | Both hands strike toward the opponent together | 30 (2.5 s cooldown) |
| Uppercut *(off by default)* | Fast upward strike | 20 |

Rules, from the BRD:
- Everyone starts on **100 HP**. HP never drops below 0. Six punches is a K.O.
- **600 ms cooldown** after every strike, and **you must pull your hand back** (neutral position) before you can strike again. One punch always equals one hit.
- A strike must go in your attack direction. Moving your hand away from the opponent does nothing.
- **Range:** if you stand too far from your opponent, you'll see **TOO FAR!** and the strike does nothing.
- If the opponent isn't detected, you'll see **NO TARGET**.
- Rounds last 60 s. If time runs out, the player with more HP wins.
- **Rematch:** after a K.O., both players raise a hand above their head, or someone clicks **REMATCH**. The game returns to the positioning screen.

These movements are designed **not** to count as attacks:
- waving (detected from fingers-up hand orientation and back-and-forth motion)
- slow movement
- raising your hand
- walking or moving your body (measured relative to the pose)
- striking in the wrong direction
- camera exposure jumps
- hands of people in the background, when both players' bodies are tracked

## How detection works

```
webcam → MediaPipe HandLandmarker (+ PoseLandmarker) → display-space coords (mirrored)
→ player identification (pose wrists → body centre → x<50% rule, sticky per hand track)
→ per-hand position history → velocity / direction / distance / acceleration
→ gates + confidence → punch state machine → hit validation (range, opponent) → damage → FX
```

- **Confidence (≥ 0.70 to register):** a weighted mean of six scores:
  - direction (also checks the hand is oriented toward the opponent)
  - speed and distance
  - acceleration (slow → fast, still before the strike, no oscillation)
  - hand-tracking score
  - player region
  - frame-difference motion around the hand's path (a secondary signal only)
- **Hard gates:** all of these must pass:
  - the hand has been tracked for 3 or more frames
  - it moves toward the opponent (mostly horizontal)
  - peak speed ≥ 0.6 screen-widths/s
  - distance ≥ 0.06 screen widths
  - it isn't waving
- **State machine per player:**

  ```
  READY → HAND MOVING TOWARD OPPONENT → ATTACK DETECTED → HIT CONFIRMED → COOLDOWN → WAIT FOR HAND RESET → READY
  ```

  After ATTACK DETECTED, the strike is observed for up to 90 ms so its full length can be measured (punch vs heavy vs special). The hit is then confirmed.
- **Speed:** CV runs at up to 30 fps. Rendering runs at 60 fps, and landmarks are smoothed between CV frames.

## On-screen tools (mouse only, never combat)

Bottom toolbar:
- **⚙ SETTINGS**:
  - gesture sensitivity
  - body tracking on/off
  - advanced attacks
  - uppercut
  - overlay, debug, sound

  Settings are saved in the browser.
- **CV OVERLAY**: shows hand skeletons (21 landmarks), velocity arrows, pose skeletons, and a panel per player with *hand detected · direction · velocity · confidence · action*.
- **DEBUG**:
  - the six confidence components as bars, for the live candidate or the last attack
  - which gates failed
  - body source (pose or estimated)
  - CV fps and ms, and the GPU/CPU delegate
  - frame-difference %, exposure filter status
  - active thresholds
- **SOUND**, **FULLSCREEN**

## Tuning and troubleshooting

| Symptom | Fix |
|---|---|
| Punches not registering | Open DEBUG and read *gates failed*. Strike **sideways**, faster and longer, and pull back after each hit. Raise **sensitivity** in Settings. |
| Random hits | Lower sensitivity. Keep hands relaxed between strikes. Check the background for moving people. |
| A hand gets assigned to the wrong player | Keep body tracking on. Stand clearly on your own side of the centre line. |
| "TOO FAR!" | Move closer to the centre line. |
| Low CV fps | Close other tabs, or turn off body tracking in Settings. |
| "Computer vision failed to load" | Check your internet connection. MediaPipe comes from `cdn.jsdelivr.net` and `storage.googleapis.com`. |

All thresholds are in the `CFG` object at the top of the script (`damage`, `cooldown`, `conf`, `punch.*`, `reach`, `weights`, …).

**Console hooks:**
- `WGF.players`, `WGF.tracks`, `WGF.state` show live state.
- `WGF.simulate()` plus `WGF.feed(hands, bodies)` inject synthetic CV results (display coordinates) for testing without a camera.
