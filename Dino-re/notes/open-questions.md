# Open questions

These are things the file alone can't settle. Each item gives what is CONFIRMED in the code and what remains open.

1. **Does anything outside the file call `updateConfigSetting`?**
   - CONFIRMED: there is no call site in the file, so `Runner.config` `GRAVITY:.6`, `INITIAL_JUMP_VELOCITY:12` and `MIN_JUMP_HEIGHT:35` are dead here.
   - Open: `Runner` is exported on `window`, so host page code could call `Runner.instance_.updateConfigSetting("GRAVITY",…)` and change the jump. Without the host HTML I can't tell.

2. **Were the Trex jump constants changed on purpose?**
   - CONFIRMED values: `GRAVITY:.47`, `INIITAL_JUMP_VELOCITY:-11.5`, `MAX_JUMP_HEIGHT:37`.
   - Open: these differ from what I remember of upstream Chromium (0.6 / −10 / 30; GUESS, from memory). I can't tell whether they are a deliberate tweak, a test fixture, or a different upstream version.

3. **Does the dino visibly clip the top of the canvas?**
   - Open: my 60-fps re-implementation puts the peak at y = −3. I haven't confirmed this in a browser.
   - Real frame deltas vary, and `Math.round` per step depends on them, so peaks will vary a little at runtime.

4. **What happens on long frames?**
   - CONFIRMED: the frame delta `s` isn't capped, so a long frame (tab hitch) makes a single large Euler step.
   - Open: I didn't determine whether that can push the dino through the y < 37 cap or produce odd landings at runtime.

5. **Where do the host-page globals come from?**
   - CONFIRMED: `HIDDEN_CLASS`, `loadTimeData` and `errorPageController` are referenced but not defined in this file.
   - CONFIRMED: `isDisabled` does `loadTimeData&&…`. If `loadTimeData` is truly undeclared, that line would throw a ReferenceError at construction.
   - Open: which page provides these globals.

6. **What are the unused fields for?**
   - CONFIRMED: `this.midair`, `this.jumpspotX` and `this.bdayFlashTimer` are set but never read.
   - Open: whether they are leftovers from removed features. I can't tell.

7. **Is `this.distance=0` in `DistanceMeter.update` a bug?**
   - CONFIRMED: it is written, but `distance` is never read anywhere.
   - Open: it looks like a leftover or a bug, but I can't confirm intent.

8. **Is the inline checkForCollision IIFE always passed a debug canvas?**
   - CONFIRMED: the call passes only two arguments (`(this.horizon.obstacles[0],this.tRex)`), so the third parameter `e` (debug canvas context) is always undefined.
   - Open: whether debug drawing was meant to be reachable another way.
