# Renames: scrambled name → meaningful name

**How these were assigned.** Every name below comes from my reading of `game.min.js`.

- **Evidence column:** quotes code that can be searched for, and gives the reason for the name.
- **Tags:** CONFIRMED means the code shows the role directly (e.g. `window.Runner=t`). GUESS means the name is my inference from what the code does.
- **Scope:** the IIFE reuses single letters as local variables, so these names apply only at the **top level of the IIFE**.

## Top-level constructors / classes

| Minified | Meaningful name | Tag | Evidence |
|---|---|---|---|
| `t` | Runner | CONFIRMED | `window.Runner=t` |
| `m` | Trex (player) | GUESS (strong) | `this.jumpVelocity=0`, `TREX:{x:848,y:2}` sprite passed in: `new m(this.canvas,this.spriteDef.TREX)` |
| `T` | Obstacle | GUESS (strong) | `T.types=[{type:"CACTUS_SMALL"…`, `T.MAX_OBSTACLE_LENGTH=3` |
| `S` | Horizon | GUESS | `this.horizon=new S(...)` |
| `p` | HorizonLine | GUESS | `new p(this.canvas,this.spritePos.HORIZON)` |
| `I` | Cloud | GUESS | `new I(this.canvas,this.spritePos.CLOUD,…)` |
| `E` | NightMode | GUESS | `this.nightMode=new E(this.canvas,this.spritePos.MOON,…)` |
| `f` | DistanceMeter | GUESS | `this.distanceMeter=new f(...)` |
| `c` | GameOverPanel | GUESS | `this.gameOverPanel=new c(...)` |
| `g` | CollisionBox | GUESS | `this.x=t,this.y=i,this.width=s,this.height=e` |

## Top-level functions

| Minified | Meaningful name | Tag | Evidence |
|---|---|---|---|
| `o(min,max)` | getRandomNum | GUESS | `Math.floor(Math.random()*(i-t+1))+t` |
| `a(b64)` | decodeBase64ToArrayBuffer | GUESS | `atob(t)`, `new ArrayBuffer(i)` |
| `r()` | getTimeStamp | GUESS | `(new Date).getTime():performance.now()` |
| `l(a,b)` | boxCompare (AABB overlap) | GUESS | `t.x<e+i.width&&t.x+t.width>e` |
| `d(box,adj)` | createAdjustedCollisionBox | GUESS | `new g(t.x+i.x,t.y+i.y,t.width,t.height)` |
| `u(ctx,a,b)` | drawCollisionBoxes (debug) | GUESS | `strokeStyle="#f00"`, `strokeStyle="#0f0"` |
| anonymous IIFE in `Runner.update` | checkForCollision(obstacle, tRex, ctx) | GUESS | `e&&function(i,s,e){…}(this.horizon.obstacles[0],this.tRex)` |

## Top-level constants

| Minified | Meaningful name | Tag | Evidence |
|---|---|---|---|
| `i` | DEFAULT_WIDTH (600) | GUESS | `var i=600`, `t.defaultDimensions={WIDTH:i` |
| `s` | FPS (60) | GUESS | `s=60`, `this.msPerFrame=1e3/s` |
| `e` | IS_HIDPI | GUESS | `e=window.devicePixelRatio>1` |
| `n` | IS_IOS | GUESS | `n=/iPad\|iPhone\|iPod/.test(window.navigator.platform)` |
| `h` | IS_MOBILE | GUESS | `h=/Android/.test(window.navigator.userAgent)\|\|n` |

## Notable properties

These are not scrambled; listed for clarity.

| Name in file | Meaning | Tag | Evidence |
|---|---|---|---|
| `m.config.INIITAL_JUMP_VELOCITY` | Trex initial jump velocity (typo for INITIAL) | CONFIRMED | `INIITAL_JUMP_VELOCITY:-11.5` |
| `Trex.minJumpHeight` | y coordinate of minimum jump height (63) | CONFIRMED | `this.minJumpHeight=this.groundYPos-this.config.MIN_JUMP_HEIGHT` |
| `Trex.groundYPos` | y of dino on ground (93) | CONFIRMED | `t.defaultDimensions.HEIGHT-this.config.HEIGHT-t.config.BOTTOM_PAD` |
| `Runner.raqId` | requestAnimationFrame handle | CONFIRMED | `this.raqId=requestAnimationFrame(` |
| `Trex.speedDrop` | fast-fall active | GUESS | `setSpeedDrop:function(){this.speedDrop=!0,this.jumpVelocity=1}` |
