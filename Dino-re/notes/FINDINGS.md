# Findings: game.min.js

- **Target:** `game.min.js` (35,853 bytes, one line; SHA-256 `e8204e9d86106a2033ebc46bd47acf586f75a0ba5e6571359a9ce980b86dcf9b`). Not modified.
- **Tags:** **CONFIRMED** means I read it directly in the code. **GUESS** means I inferred it.
- **Evidence:** each claim quotes a short snippet that can be searched for in `game.min.js`. Readable names (`Trex`, `Runner`, …) come from [renames.md](renames.md).

## Method and tools

- **Reading the code:** I split the file at `;` and `{` into a scratch copy, read all 329 resulting lines, and used `grep -o` to count identifier occurrences. The original file was never edited.
- **REA `analyze_javascript_application`:**
  - On `game.min.js` directly, it was refused because the tool only accepts a directory or `.asar`.
  - On a scratch directory holding a byte-identical copy, it worked: 1 file parsed, 9,876 AST nodes, 0 parse failures, no Electron, IPC, network, or child-process behavior found. That supports the conclusion that this is a self-contained browser script.
  - The semantic graph was "partial/unknown" and added nothing beyond reading the code by hand.
- **REA `search_strings`:** failed with "No app is open". It needs a binary opened through `open_binary`, so I didn't use it further.
- **Jump simulation:** I wrote a small Node script that **re-implements** the `updateJump` math using the constants from this file. `game.min.js` itself was never executed. The script assumes fixed 60 fps frames; results are tagged GUESS (computed).

---

## 1. What kind of program is this?

**Summary:** a browser-based, canvas-rendered endless-runner game. A T-Rex runs to the right, jumps or ducks to avoid cacti and pterodactyls, and scores by distance.

- **Recognition (GUESS):** it is almost certainly a copy of Chromium's offline "No internet" dinosaur game (`Runner`/`Trex`). Supporting strings, all **CONFIRMED** present:
  - `window.Runner=t`
  - `"offline-sound-press"`, `"offline-resources-1x"`
  - `"chrome://dino/"==document.title`
  - `errorPageController.trackEasterEgg()`
  - `loadTimeData.valueExists("disabledEasterEgg")`
- **Departs from the upstream game (GUESS):** the T-Rex jump constants differ from what I remember of upstream Chromium. See §2.9.

### Main parts

Everything is wrapped in one IIFE, `!function(){"use strict";…}()`. **CONFIRMED.**

| Part (renamed) | Minified | What it does | Evidence |
|---|---|---|---|
| **Runner** (game controller, singleton) | `t` | Config, sprite loading, input, main loop, scoring, collision, game over, restart, night-mode toggle, arcade mode | `if(t.instance_)return t.instance_`, `window.Runner=t`, `t.config={ACCELERATION:.001,…}` |
| **Trex** (player) | `m` | Position, jump physics, duck, animation and blink, collision boxes | `this.jumpVelocity=0,this.reachedMinHeight=!1,this.speedDrop=!1` |
| **Obstacle** | `T` | Cactus or pterodactyl: scrolling, animation, gap to the next obstacle | `T.types=[{type:"CACTUS_SMALL"` |
| **Horizon** | `S` | Owns the ground line, clouds, night mode and obstacles; spawns obstacles | `updateObstacles:function`, `addNewObstacle:function` |
| **HorizonLine** (ground) | `p` | Two 600-px ground tiles that scroll and swap in a random "bumpy" variant | `p.dimensions={WIDTH:600,HEIGHT:12,YPOS:127}`, `this.bumpThreshold=.5` |
| **Cloud** | `I` | Background clouds | `I.config={HEIGHT:14,MAX_CLOUD_GAP:400` |
| **NightMode** | `E` | Moon phases and stars, fade in and out | `E.phases=[140,120,100,60,40,20,0]` |
| **DistanceMeter** (score) | `f` | Score digits, high score, flashing every 100 points | `ACHIEVEMENT_DISTANCE:100,COEFFICIENT:.025` |
| **GameOverPanel** | `c` | Draws "GAME OVER" and the restart icon | `c.dimensions={TEXT_X:0,TEXT_Y:13,TEXT_WIDTH:191` |
| **CollisionBox** | `g` | `{x,y,width,height}` | `function g(t,i,s,e){this.x=t,this.y=i,this.width=s,this.height=e}` |
| Collision helpers | `l`, `d`, `u`, plus an inline IIFE | AABB overlap test, box offsetting, debug drawing; the IIFE is checkForCollision | `t.x<e+i.width&&t.x+t.width>e` |
| Utilities | `o`, `a`, `r` | Random int, base64 → ArrayBuffer (sounds), timestamp | `Math.floor(Math.random()*(i-t+1))+t`, `atob(t)` |

#### Other behaviour (all CONFIRMED)

- **Main loop:**
  - `Runner.update` runs on `requestAnimationFrame` (`this.raqId=requestAnimationFrame(this.update.bind(this))`).
  - The frame delta is real time in ms (`s=i-(this.time||i)`), and nothing caps it.
- **Speed:**
  - Starts at `SPEED:6`.
  - Each frame it rises by `ACCELERATION:.001`, up to `MAX_SPEED:13` (`this.currentSpeed<this.config.MAX_SPEED&&(this.currentSpeed+=this.config.ACCELERATION)`).
  - When the canvas is narrower than 600 px, it is scaled by `MOBILE_SPEED_COEFFICIENT:1.2`.
- **Collision:**
  - Checked only after `CLEAR_TIME:3e3` ms of running.
  - Only the first obstacle is tested (`this.horizon.obstacles[0]`).
  - It first tests outer boxes shrunk by 1 px on each side, then the per-sprite inner boxes.
  - On a hit, `gameOver()` runs.
- **Score:** `distanceRan += currentSpeed*dt/msPerFrame`. The displayed score is `Math.round(distance*.025)`.
- **Night mode:** the page inverts every `INVERT_DISTANCE:700` displayed points, for `INVERT_FADE_DURATION:12e3` ms.
- **Obstacles:**
  - Types: small cactus, large cactus, and pterodactyl. The pterodactyl only appears at speed ≥ `minSpeed:8.5`.
  - At most `MAX_OBSTACLE_DUPLICATION:2` of the same type in a row.
- **Controls:** `t.keycodes={JUMP:{38:1,32:1},DUCK:{40:1},RESTART:{13:1}}`, plus touch.

---

## 2. How the jump works

**Coordinates (CONFIRMED):** canvas coordinates, with y growing **downward**. A negative `jumpVelocity` therefore means moving **up**.

- The jump state lives on the Trex: `jumping`, `jumpVelocity`, `reachedMinHeight`, `speedDrop`, `jumpCount`.

### 2.0 Which constants are actually used

There are **two** sets of jump constants in the file:

| Constant | `Runner.config` (`t.config`) | `Trex.config` (`m.config`) |
|---|---|---|
| Gravity | `GRAVITY:.6` | `GRAVITY:.47` |
| Initial jump velocity | `INITIAL_JUMP_VELOCITY:12` | `INIITAL_JUMP_VELOCITY:-11.5` (sic, typo in the file) |
| Min jump height | `MIN_JUMP_HEIGHT:35` | `MIN_JUMP_HEIGHT:30` |
| Max jump height | – | `MAX_JUMP_HEIGHT:37` |
| Drop velocity | – | `DROP_VELOCITY:-5` |
| Speed-drop coefficient | `SPEED_DROP_COEFFICIENT:3` | `SPEED_DROP_COEFFICIENT:3` |

- **The Trex reads only its own set (CONFIRMED):** its constructor does `this.config=m.config`, and every jump function reads `this.config.*`.
- **The `Runner.config` values never reach the Trex inside this file (CONFIRMED):**
  - They could only get there through `updateConfigSetting`, which does `case"GRAVITY":case"MIN_JUMP_HEIGHT":case"SPEED_DROP_COEFFICIENT":this.tRex.config[t]=i` and `case"INITIAL_JUMP_VELOCITY":this.tRex.setJumpVelocity(i)`.
  - `updateConfigSetting` occurs **once** in the file, at its definition. Nothing in the file calls it.
  - `setJumpVelocity` occurs twice: its definition, and the call inside `updateConfigSetting`.
- **Conclusion:** the effective values are the `Trex.config` column. **CONFIRMED** for this file in isolation.
- **Outside callers (GUESS):** outside code could still call `Runner.instance_.updateConfigSetting(...)`, because `Runner` is exported on `window`.

**Derived positions (CONFIRMED arithmetic from the code):**

| Value | Source | Result |
|---|---|---|
| Ground y | `groundYPos = defaultDimensions.HEIGHT(150) − Trex HEIGHT(47) − BOTTOM_PAD(10)` | **93** |
| `minJumpHeight` (a y coordinate) | `this.minJumpHeight=this.groundYPos-this.config.MIN_JUMP_HEIGHT` → 93 − 30 | **63**, i.e. 30 px above the ground |
| `MAX_JUMP_HEIGHT` | Compared directly with `yPos` (`this.yPos<this.config.MAX_JUMP_HEIGHT`) | **y = 37**, which is 56 px above the ground. It is an absolute y coordinate, not a height. |

### 2.1 What starts a jump

All of the following is **CONFIRMED**.

1. **Input handling:** `Runner.handleEvent` sends `keydown`, `touchstart` and `pointerdown` to `onKeyDown`.
2. **The jump branch:** `t.keycodes.JUMP[i.keyCode]||i.type==t.events.TOUCHSTART`.
   - The jump key codes are **38 and 32**. That they are Arrow Up and Space is the standard DOM keyCode mapping (GUESS-free general knowledge).
   - A `touchstart` anywhere also jumps.
3. **Mouse clicks do not jump:** `pointerdown` reaches `onKeyDown`, but it has no `keyCode` and isn't `touchstart`, so the branch is skipped.
4. **Required conditions:**
   - `this.isCanvasInView()`, meaning the canvas top is greater than −10 (`CANVAS_IN_VIEW_OFFSET:-10`).
   - The game is not crashed or paused.
5. **First key press:** if the game isn't playing yet, it starts the game first (`this.loadSounds(),this.setPlayStatus(!0),this.update()`).
6. **Guard:** `this.tRex.jumping||this.tRex.ducking||(this.playSound(this.soundFx.BUTTON_PRESS),this.tRex.startJump(this.currentSpeed))`. There is no double jump and no jumping while ducking. The press sound plays.
7. **`startJump`:**
   ```js
   startJump:function(t){this.jumping||(this.update(0,m.status.JUMPING),
     this.jumpVelocity=this.config.INIITAL_JUMP_VELOCITY-t/10,
     this.jumping=!0,this.reachedMinHeight=!1,this.speedDrop=!1)}
   ```
   - **Initial velocity = −11.5 − currentSpeed/10** (px per frame, upward).
   - At start speed 6 it is **−12.1**. At max speed 13 it is **−12.8**. So the faster the game, the harder the jump.
   - The sprite switches to the `JUMPING` frame (`JUMPING:{frames:[0],msPerFrame:1e3/60}`).

### 2.2 How fast it rises (per-frame integration)

`Runner.update` calls `this.tRex.jumping&&this.tRex.updateJump(s)`, where `s` is the elapsed ms. The body of `updateJump` (**CONFIRMED**):

```js
var s=t/m.animFrames[this.status].msPerFrame;          // s = dt / (1000/60)  -> "number of 60fps frames"
this.speedDrop ? this.yPos+=Math.round(this.jumpVelocity*this.config.SPEED_DROP_COEFFICIENT*s)
               : this.yPos+=Math.round(this.jumpVelocity*s);
this.jumpVelocity+=this.config.GRAVITY*s;
(this.yPos<this.minJumpHeight||this.speedDrop)&&(this.reachedMinHeight=!0);
(this.yPos<this.config.MAX_JUMP_HEIGHT||this.speedDrop)&&this.endJump();
this.yPos>this.groundYPos&&(this.reset(),this.jumpCount++)
```

- **Integration method:** explicit Euler. Position is updated first, using the *old* velocity, then gravity is added.
- **Units:** velocity in px per 60-fps frame; gravity in px/frame².
- **Rounding:** the position step is rounded to whole pixels each frame.
- **Frame-rate scaling:** time is normalised to 60 fps through `msPerFrame` of the JUMPING animation (`1e3/60`). Because `dt` is uncapped (CONFIRMED), a long frame gives one large step.

### 2.3 What pulls it back down

- **Gravity (CONFIRMED):** **`GRAVITY = 0.47`** px/frame² (`Trex.config`), added every frame: `this.jumpVelocity+=this.config.GRAVITY*s`. Runner's `GRAVITY:.6` is unused, as shown in §2.0.
- **Speed drop / fast fall (CONFIRMED):**
  - Pressing Down (key code 40) while jumping calls `this.tRex.jumping?this.tRex.setSpeedDrop()`.
  - That does `setSpeedDrop:function(){this.speedDrop=!0,this.jumpVelocity=1}`: velocity becomes **+1** (downward) immediately.
  - Each position step is then multiplied by **`SPEED_DROP_COEFFICIENT = 3`**. Gravity still adds 0.47 per frame to the un-multiplied velocity.
  - Releasing Down mid-air clears `speedDrop` (`this.tRex.speedDrop=!1,this.tRex.setDuck(!1)` in `onKeyUp`), which turns the ×3 multiplier back off.

### 2.4 What limits how high it goes

Three mechanisms, all built on `endJump`:

```js
endJump:function(){this.reachedMinHeight&&this.jumpVelocity<this.config.DROP_VELOCITY&&(this.jumpVelocity=this.config.DROP_VELOCITY)}
```

`endJump` clamps the upward speed to **`DROP_VELOCITY = −5`** px/frame. It acts only once `reachedMinHeight` is true and only if the dino is rising faster than 5 px/frame. **CONFIRMED.**

1. **Minimum height (CONFIRMED):** `reachedMinHeight` becomes true once `yPos < minJumpHeight (63)`, i.e. 30 px above the ground. Before that, `endJump` does nothing.
2. **Early release, i.e. a variable-height jump (CONFIRMED):**
   - `onKeyUp` calls `this.tRex.endJump()` when `this.isRunning()` and the event is a jump key, `touchend` or `pointerup`.
   - If that release happens **before** the minimum height is reached, it is silently ignored and **never re-applied later**: the only other caller is the automatic cap below.
3. **Automatic cap (CONFIRMED):**
   - Every frame where `yPos < MAX_JUMP_HEIGHT (37)`, `endJump()` is called automatically.
   - So whether the key is held or not, the upward speed is clamped to −5 as soon as the dino passes y = 37.
   - After that it keeps coasting upward until gravity cancels the −5.

**Computed jump profile (GUESS, from my re-implementation at a fixed 60 fps, `s = 1`):**

| Scenario | Peak y | Peak height above ground | Airtime |
|---|---|---|---|
| Speed 6, key held (or released before frame 4) | **−3** | 96 px | 38 frames ≈ 633 ms |
| Speed 6, released after frame 3 | 28 | 65 px | 31 frames ≈ 517 ms |
| Speed 6, released after frame 4 | 17 | 76 px | 34 frames ≈ 567 ms |
| Speed 6, released after frame 5 | 7 | 86 px | 36 frames ≈ 600 ms |
| Speed 13, key held | 4 | 89 px | 36 frames ≈ 600 ms |
| Speed 6 or 13, Down pressed on frame 10 | 13 / 16 | 80 / 77 px | 19 frames ≈ 317 ms |

How to read the table:
- **Only a narrow release window shortens the jump:** about frames 4–6 (~50–100 ms after the press). Releasing earlier is ignored because `reachedMinHeight` is still false. Releasing later does nothing because the automatic cap has already fired.
- **A full jump at start speed goes above the canvas:** it peaks at **y = −3**, so the T-Rex sprite rises 3 px above the top of the 150-px-tall canvas.
- **Rounding makes height non-monotonic in speed:** max speed jumps slightly *lower* than start speed, because of where the y = 37 cap triggers.

### 2.5 How landing is detected

- **Landing check (CONFIRMED):** `this.yPos>this.groundYPos&&(this.reset(),this.jumpCount++)`.
  - Landing triggers when the dino ends up *below* ground (y > 93), not merely on it.
  - `reset()` then snaps it back: `this.yPos=this.groundYPos,this.jumpVelocity=0,this.jumping=!1,this.ducking=!1,this.update(0,m.status.RUNNING),this.midair=!1,this.speedDrop=!1,this.jumpCount=0`.
- **`jumpCount` (CONFIRMED):** `reset()` sets it to 0 and `updateJump` then increments it, so after any landing it equals **1**.
  - `Runner.update` checks `1!=this.tRex.jumpCount||this.playingIntro||this.playIntro()`.
  - The **landing of the very first jump** therefore starts the intro animation: `"intro .4s ease-out 1 both"`, which widens the canvas, then `startGame`. On later landings `playIntro` only does work if the game is crashed.
- **Landing from a speed drop turns into a duck (CONFIRMED by tracing statement order):**
  1. Inside `reset()`, `update(0,RUNNING)` runs while `speedDrop` is still true and `yPos==groundYPos`.
  2. So `Trex.update` executes `this.speedDrop&&this.yPos==this.groundYPos&&(this.speedDrop=!1,this.setDuck(!0))`.
  3. The dino lands already ducking, and stays down until the Down key is released (`onKeyUp` → `setDuck(!1)`).
- **Collision during a jump (CONFIRMED):** the `RUNNING` collision boxes are used because `ducking` is false: `s.ducking?m.collisionBoxes.DUCKING:m.collisionBoxes.RUNNING`.

### 2.6 Exact numeric values used by the jump

| Quantity | Value | Where |
|---|---|---|
| Gravity | 0.47 px/frame² | `m.config` `GRAVITY:.47` |
| Base jump velocity | −11.5 px/frame | `INIITAL_JUMP_VELOCITY:-11.5` |
| Speed bonus | −currentSpeed / 10 | `INIITAL_JUMP_VELOCITY-t/10` |
| Release / cap velocity | −5 px/frame | `DROP_VELOCITY:-5` |
| Min jump height | 30 px (y = 63) | `MIN_JUMP_HEIGHT:30` |
| Auto-cap line | y < 37 | `MAX_JUMP_HEIGHT:37` |
| Speed-drop velocity | +1 px/frame | `this.jumpVelocity=1` |
| Speed-drop multiplier | ×3 | `SPEED_DROP_COEFFICIENT:3` |
| Physics frame | 1000/60 ms | `JUMPING:{frames:[0],msPerFrame:1e3/60}` |
| Ground y | 93 | 150 − 47 − 10 |
| Game speed range | 6 → 13, +0.001/frame | `SPEED:6`, `MAX_SPEED:13`, `ACCELERATION:.001` |
| Unused (Runner) | `GRAVITY:.6`, `INITIAL_JUMP_VELOCITY:12`, `MIN_JUMP_HEIGHT:35` | `t.config` |

### 2.7 Small oddities (CONFIRMED)

- **`this.midair`:** only ever set to false; it is never read.
- **`jumpspotX`:** initialised to 0 and never used.
- **`setJumpVelocity(v)`:** would set `INIITAL_JUMP_VELOCITY=-v` and `DROP_VELOCITY=-v/2`. It mutates the shared `m.config` object, because `this.config` is that object, not a copy.

### 2.8 Other confirmed oddities in the file

- **`HIDDEN_CLASS`:** used in `setPlayStatus` but not defined in this file. It must come from the host page.
- **Other host-page globals:** `loadTimeData` and `errorPageController` are also not defined here.
- **`t.classes.PLAYER`:** referenced, but `t.classes` has no `PLAYER` key. The canvas class therefore stays `"runner-canvas"`.
- **`this.bdayFlashTimer=null`:** written once in `restart` and never read.

### 2.9 Comparison with upstream Chromium (GUESS, from memory, not evidence)

- **What I remember:** the original Chromium `Trex.config` uses `GRAVITY: 0.6`, `INIITAL_JUMP_VELOCITY: -10` and `MAX_JUMP_HEIGHT: 30`.
- **Upstream typo:** I believe the `INIITAL` misspelling is also present in Chromium.
- **What this file has instead:** **0.47 / −11.5 / 37**, meaning lower gravity, a stronger launch, and a higher cap line.
- **Probable effect:** these changes would explain the "jumps off the top of the canvas" result above.
- **Status:** none of this can be verified from the file itself.
