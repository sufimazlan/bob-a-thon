# Business Requirements Document (BRD)

# Webcam Gesture Fighter

**Document Type:** Priority BRD
**Project Type:** Two-Player Computer-Vision Fighting Game
**Platform:** Web Browser
**Input:** Single Laptop Webcam Only
**Status:** Development Specification
**Version:** 2.0

---

## 1. Executive Summary

**Webcam Gesture Fighter** is a browser-based, two-player fighting game controlled entirely through a single laptop webcam.

Two players stand in front of the webcam:

```text
              LAPTOP WEBCAM
                   │
                   ▼

┌─────────────────────────────────────────────┐
│                                             │
│   PLAYER 2                  PLAYER 1       │
│     LEFT                     RIGHT         │
│                                             │
│    👤  ────►       ◄────  👤               │
│                                             │
│   Attack →             ← Attack             │
│                                             │
└─────────────────────────────────────────────┘
```

Player 1 occupies the **right side** of the camera image.

Player 2 occupies the **left side** of the camera image.

The game uses computer vision to detect each player's hand/body movement and determine whether the player is performing an attack.

### Example

Player 1 performs a punch/karate movement toward the **left**:

```text
PLAYER 1
   👤
   │
   └──────► LEFT

          HIT!

PLAYER 2 HP
100 → 83
```

Player 2 then performs a punch/karate movement toward the **right**:

```text
PLAYER 2
   👤
   │
   └──────► RIGHT

          HIT!

PLAYER 1 HP
100 → 82
```

The entire interaction must happen through the webcam.

**No keyboard.**

**No mouse controls during combat.**

**No game controller.**

**No touch controls.**

---

# 2. Core Gameplay Concept

The game is designed around one simple principle:

> **Move your hands toward your opponent to attack them.**

Each player has:

* 100 HP
* A physical position in front of the webcam
* A detected hand/body position
* A detected attack direction
* An attack cooldown
* A gesture confidence score

The computer vision system continuously analyzes the webcam feed.

When a valid attack gesture is detected, the game determines:

1. Which player performed the gesture
2. Which direction the hand moved
3. Whether the movement was fast enough
4. Whether the movement was directed toward the opponent
5. Whether the attack is within the allowed attack range
6. Whether the gesture has sufficient confidence
7. Whether the player's attack cooldown has expired

If all conditions are satisfied, damage is applied to the opponent.

---

# 3. Player Position

## Player 1

Player 1 must stand on the **RIGHT side** of the webcam.

```text
┌──────────────────────────────────────────┐
│                         PLAYER 1         │
│                            👤            │
│                            │             │
│                            │             │
│                            │             │
└──────────────────────────────────────────┘
```

Player 1 attacks toward the **LEFT**.

```text
PLAYER 1
   👤
   └──────────────► LEFT
```

## Player 2

Player 2 must stand on the **LEFT side** of the webcam.

```text
┌──────────────────────────────────────────┐
│       PLAYER 2                           │
│          👤                              │
│          │                               │
│          │                               │
│          │                               │
└──────────────────────────────────────────┘
```

Player 2 attacks toward the **RIGHT**.

```text
PLAYER 2
   👤
   └──────────────► RIGHT
```

---

# 4. Camera Layout

The webcam image is divided into two player regions.

```text
┌─────────────────────────────────────────────┐
│                                             │
│              │                              │
│   PLAYER 2   │          PLAYER 1            │
│              │                              │
│      👤      │                 👤           │
│              │                              │
│              │                              │
│              │                              │
└─────────────────────────────────────────────┘
       LEFT                 RIGHT
```

### Camera Requirements

* Use laptop webcam.
* Request camera permission automatically.
* Target resolution: 1280 × 720.
* No audio required.
* Use `navigator.mediaDevices.getUserMedia()`.
* Mirror the preview if necessary for intuitive interaction.
* Computer vision coordinates must remain consistent after mirroring.

---

# 5. Computer Vision Architecture

The game must use **computer vision**, not keyboard simulation.

## Primary CV Technology

Use:

**MediaPipe Hands**

or an equivalent browser-compatible hand-tracking solution.

Optional:

**MediaPipe Pose**

may be used to improve player-body tracking and determine player boundaries.

Frame-difference detection can be used as a supporting signal, but must NOT be the primary method for determining whether a punch occurred.

### Reason

Simple frame difference cannot reliably distinguish:

```text
punch
```

from:

```text
walking
waving
moving body
adjusting clothes
camera exposure changes
```

Hand landmark tracking provides significantly more useful information.

---

# 6. Hand Detection

The system should detect the following landmarks for each player:

```text
        WRIST
          │
      ┌───┴───┐
      │       │
   THUMB    INDEX
      │       │
   MIDDLE   RING
      │       │
      └── PINKY
```

Track:

* Wrist
* Thumb
* Index finger
* Middle finger
* Ring finger
* Pinky

For each detected hand, record:

```javascript
{
    x,
    y,
    timestamp,
    player,
    handedness
}
```

The system should maintain a short history of hand positions.

Example:

```text
t0: x = 0.72
t1: x = 0.68
t2: x = 0.61
t3: x = 0.52
```

This indicates strong movement toward the LEFT.

---

# 7. Attack Detection

An attack is not simply:

> "A hand moved."

The system must detect a **rapid directional hand movement toward the opponent**.

## Player 1

Player 1 is on the RIGHT.

Valid attack direction:

```text
RIGHT → LEFT
```

Example:

```text
Before

        👤
         \
          ✋


During punch

        👤
      ───✋
       ←
```

The hand's X coordinate should move significantly toward the LEFT within a short period.

---

## Player 2

Player 2 is on the LEFT.

Valid attack direction:

```text
LEFT → RIGHT
```

Example:

```text
Before

   👤
  /
 ✋


During punch

   👤
    ───✋
        →
```

The hand's X coordinate should move significantly toward the RIGHT.

---

# 8. Punch Recognition Algorithm

For each player's tracked hand:

### Step 1 — Track position

Store the previous positions:

```text
P0
P1
P2
P3
P4
```

### Step 2 — Calculate velocity

```text
velocityX = currentX - previousX
```

### Step 3 — Determine direction

Player 1:

```text
velocityX < negative_threshold
```

Player 2:

```text
velocityX > positive_threshold
```

### Step 4 — Determine speed

The movement must exceed a minimum velocity threshold.

Slow movement:

```text
✋ ───────────────►
```

should NOT automatically cause damage.

Fast movement:

```text
✋ ───►
```

should be interpreted as an attack.

### Step 5 — Determine acceleration

A punch should ideally have:

```text
slow → fast → stop/retract
```

movement characteristics.

This helps distinguish a punch from simply holding the hand in a particular position.

---

# 9. Attack Confidence

Every detected attack should receive a confidence score.

Example:

```text
Direction:       0.95
Speed:           0.88
Acceleration:    0.91
Hand tracking:   0.97
Player region:   0.99
--------------------------------
Overall:         0.94
```

Minimum attack confidence:

```text
0.70
```

If:

```text
confidence < 0.70
```

do not register the attack.

---

# 10. Attack Types

The initial version should focus on **one reliable attack** before adding complicated moves.

## Attack 1 — Punch / Karate Strike

Both players can perform the same fundamental attack.

### Player 1

```text
RIGHT → LEFT
```

Damage:

```text
17 HP
```

Example:

```text
Player 2:

100 HP
   ↓
83 HP
```

### Player 2

```text
LEFT → RIGHT
```

Damage:

```text
17 HP
```

Example:

```text
Player 1:

100 HP
   ↓
83 HP
```

---

# 11. Optional Attack Types

After basic punch detection works reliably, support additional attacks.

## Heavy Strike

Requirements:

* Higher hand velocity
* Larger movement distance
* High confidence

Damage:

```text
25 HP
```

---

## Uppercut

Detect strong upward hand movement.

Direction:

```text
↑
```

Damage:

```text
20 HP
```

---

## Special Attack

Detect both hands moving forward simultaneously.

Player 1:

```text
        👤
       / \
    ✋     ✋
       ← ←
```

Player 2:

```text
        👤
       / \
    ✋     ✋
       → →
```

Damage:

```text
30 HP
```

Special attacks should have a longer cooldown.

---

# 12. No Keyboard Interaction

This requirement is strict.

## Forbidden

The game must NOT use:

* Keyboard attack buttons
* Keyboard movement
* WASD
* Arrow keys
* Space bar
* Enter key for attacks
* Game controller
* Mouse click for combat

The webcam is the gameplay input.

Keyboard may only be used for normal browser functions such as:

* refreshing the page
* browser developer tools
* exiting fullscreen

There must be **no keyboard fallback combat mode**.

---

# 13. Hit Detection

A successful attack requires all of the following:

```text
Valid player
      +
Valid hand detected
      +
Correct attack direction
      +
Sufficient movement speed
      +
Sufficient movement distance
      +
Sufficient confidence
      +
Attack cooldown expired
      +
Opponent exists
      =
SUCCESSFUL HIT
```

---

# 14. Attack Direction Validation

## Player 1

Player 1 is on the right.

Therefore:

```text
Expected direction = LEFT
```

If Player 1 moves their hand:

```text
→ RIGHT
```

do NOT damage Player 2.

---

## Player 2

Player 2 is on the left.

Therefore:

```text
Expected direction = RIGHT
```

If Player 2 moves their hand:

```text
← LEFT
```

do NOT damage Player 1.

---

# 15. Attack Range

The game should use the players' relative camera positions.

Example:

```text
PLAYER 2                  PLAYER 1

   👤                         👤
   │                          │
   ✋ ───────────────►        │
            ATTACK
```

If the players are too far apart, the attack should not register.

However, the game does not need physical collision between actual hands.

Instead, calculate a virtual combat distance based on:

* Player body center
* Hand position
* Camera coordinates

This creates a virtual fighting arena.

---

# 16. Damage System

Starting HP:

```text
100
```

Default attack damage:

```text
17
```

Example:

```text
PLAYER 1                  PLAYER 2

HP: 100                   HP: 100

   👤                         👤
    \                       /
     ✋ ────────►          

                           HIT!

PLAYER 2 HP:
100 → 83
```

Second successful attack:

```text
83 → 66
```

Third:

```text
66 → 49
```

Fourth:

```text
49 → 32
```

Fifth:

```text
32 → 15
```

Sixth:

```text
15 → 0
```

Player 1 wins.

---

# 17. HP Rules

HP must never fall below zero.

```javascript
hp = Math.max(0, hp - damage);
```

When HP reaches zero:

```text
GAME OVER
```

The losing player cannot perform additional attacks.

---

# 18. Attack Cooldown

A single physical punch must not generate multiple attacks.

Example:

```text
One punch
   ↓
Detection
   ↓
17 damage
   ↓
Cooldown
   ↓
Wait for hand reset
```

Default cooldown:

```text
600 ms
```

Additionally, the system should require the hand to return to a neutral position before another punch can be registered.

This prevents:

```text
ONE PUNCH

17
34
51
68
85
100
```

from occurring due to multiple camera frames.

---

# 19. Punch State Machine

Each player should have a punch state.

```text
READY
  ↓
HAND MOVING TOWARD OPPONENT
  ↓
ATTACK DETECTED
  ↓
HIT CONFIRMED
  ↓
COOLDOWN
  ↓
WAIT FOR HAND RESET
  ↓
READY
```

This is preferable to simply checking:

```javascript
if (velocityX > threshold)
```

on every frame.

---

# 20. Player Detection

The system must determine which detected hand belongs to which player.

Basic rule:

```text
x < 50%
    → Player 2

x >= 50%
    → Player 1
```

However, hand tracking should use the player's tracked body/pose where possible.

The system must avoid accidentally assigning a hand crossing the center line to the wrong player.

---

# 21. Center Boundary

Display a virtual center boundary:

```text
┌─────────────────────────────────────────────┐
│                                             │
│         PLAYER 2 │ PLAYER 1                │
│                  │                          │
│        👤        │        👤                │
│                  │                          │
│                  │                          │
└─────────────────────────────────────────────┘
                   ↑
              CENTER LINE
```

Players should remain primarily within their respective zones.

---

# 22. Game Interface

The live webcam should remain visible.

Example:

```text
┌──────────────────────────────────────────────┐
│                                              │
│ P2                         TIMER        P1    │
│ ██████████░░░░              60       ███████ │
│ 83 HP                                  100 HP│
│                                              │
├──────────────────────────────────────────────┤
│                                              │
│       👤                  │        👤         │
│       P2                  │        P1         │
│                           │                  │
│             CAMERA FEED                     │
│                                              │
│                           │                  │
├──────────────────────────────────────────────┤
│                                              │
│     "PUNCH!"                    "READY"      │
│                                              │
└──────────────────────────────────────────────┘
```

---

# 23. Visual Attack Feedback

When Player 1 successfully hits Player 2:

```text
PLAYER 1
   👤
    \
     ✋ ─────►

                 💥

             -17 HP
```

Display:

```text
-17
```

near Player 2.

Player 2 should also receive:

* Screen shake
* Hit flash
* Impact particles
* Damage number
* Short hit sound

---

# 24. Gesture Visualization

During development, display computer-vision information over the webcam.

For each hand:

```text
        ●────●
       /      \
      ●        ●
       \      /
        ●────●
           │
          WRIST
```

Display:

```text
PLAYER 1
Hand detected
Direction: ←
Velocity: 0.82
Confidence: 94%
Action: PUNCH
```

This debug information can later be hidden for normal gameplay.

---

# 25. Camera Processing Pipeline

```text
WEBCAM
  ↓
VIDEO FRAME
  ↓
MEDIA PIPE HAND / POSE DETECTION
  ↓
PLAYER IDENTIFICATION
  ↓
HAND LANDMARK EXTRACTION
  ↓
POSITION HISTORY
  ↓
VELOCITY CALCULATION
  ↓
DIRECTION DETECTION
  ↓
ACCELERATION ANALYSIS
  ↓
GESTURE CLASSIFICATION
  ↓
CONFIDENCE CHECK
  ↓
ATTACK VALIDATION
  ↓
HIT DETECTION
  ↓
DAMAGE
  ↓
HP UPDATE
  ↓
VISUAL / AUDIO EFFECT
```

---

# 26. Frame Difference

Frame difference may still be used as a **secondary motion signal**.

It can help answer:

```text
"Did something move quickly?"
```

But hand landmarks should answer:

```text
"Which hand moved?"
"Which player?"
"Which direction?"
"How far?"
"How fast?"
```

Therefore:

**Primary detection:**

```text
Hand landmarks
```

**Secondary signal:**

```text
Frame difference / motion analysis
```

---

# 27. Performance Requirements

Target:

```text
60 FPS rendering
```

Computer vision does not necessarily need to run at 60 FPS.

For example:

```text
Rendering:
60 FPS

Computer vision:
20–30 FPS
```

Interpolate or smooth tracking data between CV updates.

Target gesture-to-hit latency:

```text
< 150 ms
```

---

# 28. False Positive Prevention

The system must avoid treating normal movement as attacks.

The following should NOT cause damage:

```text
waving
slow hand movement
scratching head
walking
moving body
adjusting clothes
raising hand slowly
camera exposure changes
background movement
```

An attack should require a combination of:

```text
direction
+
speed
+
distance
+
acceleration
+
player region
+
confidence
+
attack state
```

---

# 29. Neutral Position

After an attack, the player should return their hand toward a neutral position.

Example:

```text
READY

      ✋
       \
        👤
```

Attack:

```text
      👤────✋
           ←
```

Reset:

```text
      ✋
       \
        👤
```

The next attack cannot begin until the hand has sufficiently reset.

---

# 30. Game States

```text
LOADING
   ↓
CAMERA_PERMISSION
   ↓
PLAYER_POSITIONING
   ↓
READY
   ↓
COUNTDOWN
   ↓
FIGHTING
   ↓
GAME_OVER
   ↓
REMATCH
```

---

# 31. Player Positioning Screen

Before starting the match, show instructions:

```text
GET READY

PLAYER 2                  PLAYER 1

   👤                         👤
 LEFT SIDE                 RIGHT SIDE

Stay on your side of the camera.

Move your hands to test detection.

PLAYER 2: ✓ DETECTED
PLAYER 1: ✓ DETECTED

          STARTING...
```

The game should not start until both players are detected.

---

# 32. Countdown

Display:

```text
3
```

then:

```text
2
```

then:

```text
1
```

then:

```text
FIGHT!
```

Combat input should only be processed during:

```text
FIGHTING
```

---

# 33. Game Over

When one player's HP reaches zero:

```text
┌──────────────────────────────────────────┐
│                                          │
│                 K.O.!                    │
│                                          │
│             PLAYER 1 WINS                │
│                                          │
│       PLAYER 2     0 HP                  │
│       PLAYER 1     17 HP                 │
│                                          │
│             REMATCH                      │
│                                          │
└──────────────────────────────────────────┘
```

The rematch should return to the player-position detection screen.

---

# 34. Audio

Use Web Audio API or lightweight audio files.

Sounds:

* Countdown
* Fight
* Punch
* Hit
* Heavy hit
* Special
* KO

When a punch connects:

```text
PUNCH
   ↓
HIT
   ↓
-17 HP
```

---

# 35. Technology Stack

| Component               | Technology                |
| ----------------------- | ------------------------- |
| Language                | JavaScript                |
| UI                      | HTML5 + CSS3              |
| Rendering               | Canvas 2D                 |
| Camera                  | `getUserMedia()`          |
| Hand Detection          | MediaPipe Hands           |
| Optional Body Detection | MediaPipe Pose            |
| Game Loop               | `requestAnimationFrame()` |
| Audio                   | Web Audio API             |
| Backend                 | None                      |
| Input                   | Webcam only               |
| Deployment              | Client-side               |

---

# 36. External Dependencies

Unlike the original specification, computer vision should NOT be implemented solely with raw frame differences.

Recommended:

```text
MediaPipe Hands
```

because the game fundamentally depends on recognizing hand movement.

The application may load the computer-vision library from a CDN.

If a completely offline single-file version is required later, the required CV assets can be bundled separately.

---

# 37. Development Priority

## Phase 1 — Webcam

Implement:

* Camera permission
* Webcam feed
* 1280×720
* Mirrored display
* Center line

---

## Phase 2 — Hand Detection

Implement:

* MediaPipe Hands
* Hand landmarks
* Hand tracking
* Left/right player assignment

---

## Phase 3 — Movement Tracking

Implement:

* Hand position history
* Velocity
* Direction
* Acceleration
* Movement distance

---

## Phase 4 — Punch Recognition

Implement ONLY:

```text
Player 1:
RIGHT → LEFT

Player 2:
LEFT → RIGHT
```

Do not implement special moves yet.

---

## Phase 5 — Combat

Implement:

```text
100 HP
17 damage
600 ms cooldown
```

Example:

```text
P1 attacks
    ↓
P2 HP: 100 → 83
```

---

## Phase 6 — Feedback

Add:

* Damage numbers
* Hit animation
* Screen shake
* Sound
* HP animation
* Attack indicator

---

## Phase 7 — Reliability

Tune:

* Gesture thresholds
* Confidence
* Attack cooldown
* Hand reset
* False positives
* Player boundaries

---

## Phase 8 — Advanced Attacks

Only after basic punch detection is reliable:

* Heavy strike
* Uppercut
* Two-hand special
* Dodge
* Block

---

# 38. Important Product Principle

The game should feel like:

> **"I physically punch toward the other player, and the other player's HP goes down."**

It should NOT feel like:

> "I am moving around in front of a camera and the game randomly interprets my movement."

Therefore, gesture recognition must prioritize **directional intent**.

---

# 39. Example Complete Match

Initial state:

```text
PLAYER 2 HP: 100
PLAYER 1 HP: 100
```

Player 1 on right performs:

```text
RIGHT → LEFT
```

Computer vision detects:

```text
Player: 1
Direction: LEFT
Velocity: HIGH
Confidence: 94%
Gesture: PUNCH
```

Game:

```text
PLAYER 2 HP
100 → 83
```

Player 2 performs:

```text
LEFT → RIGHT
```

CV detects:

```text
Player: 2
Direction: RIGHT
Velocity: HIGH
Confidence: 91%
Gesture: PUNCH
```

Game:

```text
PLAYER 1 HP
100 → 83
```

The fight continues until one player's HP reaches zero.

---

# 40. Out of Scope

The following are NOT required for version 1:

* Online multiplayer
* Two webcams
* Keyboard combat
* Gamepad
* Mobile support
* Accounts
* Leaderboards
* Multiplayer networking
* VR
* External game engine
* AI opponent

---

# 41. Final Requirements Checklist

### Camera

* [ ] Single laptop webcam
* [ ] 1280×720 target
* [ ] Camera permission
* [ ] Live camera feed
* [ ] Mirrored preview
* [ ] Center boundary

### Player Detection

* [ ] Player 2 = LEFT
* [ ] Player 1 = RIGHT
* [ ] Both players detected before match
* [ ] Hand tracking
* [ ] Player association

### Combat

* [ ] Player 1 attacks LEFT
* [ ] Player 2 attacks RIGHT
* [ ] Punch detection
* [ ] Direction detection
* [ ] Velocity detection
* [ ] Confidence detection
* [ ] Attack cooldown
* [ ] Hand reset requirement
* [ ] 100 HP
* [ ] 17 damage per successful punch

### Input

* [ ] Webcam only
* [ ] No keyboard combat
* [ ] No mouse combat
* [ ] No controller

### Feedback

* [ ] HP bars
* [ ] Damage numbers
* [ ] Punch animation
* [ ] Hit effect
* [ ] Sound
* [ ] K.O. screen
* [ ] Rematch

### Computer Vision

* [ ] MediaPipe Hands
* [ ] Hand landmarks
* [ ] Position history
* [ ] Velocity
* [ ] Direction
* [ ] Acceleration
* [ ] Gesture confidence
* [ ] False-positive filtering

---

# 42. Core Game Rule

The fundamental rule of the entire project is:

```text
                    WEBCAM

       PLAYER 2                 PLAYER 1
         LEFT                     RIGHT

          👤                       👤
           \                       /
            ✋ ───────►     ◄──── ✋
                ATTACK

       P2 attacks RIGHT
                    ↓
              P1 loses HP

       P1 attacks LEFT
                    ↓
              P2 loses HP
```

**The webcam is the controller.**

**The player's physical hand movement is the attack input.**

**A successful directional gesture causes real HP damage.**

**No keyboard interaction is required for gameplay.**
