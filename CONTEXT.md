# sbyhunter — Design Context

A mobile-first clone of the 1983 Bally Midway arcade game *Spy Hunter*, built as a single HTML file for iPhone/iPad via GitHub Pages. Sister project to `btempest` (Tempest clone).

This document captures every mechanic we've built and the design reasoning behind each. Read this before suggesting changes so you understand *why* things are the way they are.

---

## Architecture

- **Single `index.html`** — all HTML, CSS, JavaScript inline. No build step, no frameworks.
- **Lives in `docs/`** — GitHub Pages serves only this folder. Dev files (CLAUDE.md, CONTEXT.md) stay in repo root and are not publicly accessible.
- **HTML5 Canvas 2D** for rendering. No WebGL.
- **Virtual resolution** of 360×640 (portrait). Scaled to fit screen with letterbox.
- **Mobile-first.** Touch controls. iPad fixed via `navigator.maxTouchPoints > 1`.

## PWA setup (add-to-home-screen + offline)

Same pattern as sister project `btempest`. Everything is inline in the single HTML file — no external manifest.json or sw.js files needed.

- **Icon**: Canvas-generated 512×512 PNG (road + car + "SPY HUNTER" text), injected as data URL into `<link rel="apple-touch-icon">` and `<link rel="icon">`.
- **Manifest**: Created as a JSON blob URL at runtime, appended as `<link rel="manifest">`. `display: fullscreen`, `orientation: any`.
- **Service worker**: Inline blob-registered worker using **stale-while-revalidate** caching:
  - **Install**: Caches the page. `skipWaiting()` to activate immediately.
  - **Fetch**: Serves cached version instantly (offline works). Fetches fresh copy from network in background. If network succeeds, updates cache for next visit.
  - **Activate**: Cleans up old cache versions. `clients.claim()` to take control.
  - **Cache name**: `'spyhunter-v1'` — bump this version string when deploying breaking changes to force a full re-cache.
- **To install on iPhone**: Open in Safari → Share → "Add to Home Screen". Launches fullscreen, works offline.

## Core game loop

- `requestAnimationFrame` loop calls `update(dt)` then `render()`
- Both wrapped in try/catch — errors log to console with stack but don't kill the loop
- States: `'menu'`, `'playing'`, `'crashing'`, `'awaiting_truck'`, `'gameover'`
- `gameMode` and `state` are separate — `state` controls whether update runs at all; `gameMode` is the sub-state within play

## Coordinate system (IMPORTANT — we got this wrong twice)

- **World Z** increases as you drive forward. `trackPos` is the player's current world Z.
- **POSITIVE vz means moving forward through the world** (same direction as player).
- For an enemy to appear to *hover* on screen, their `vz` ≈ player's world speed (`player.speed * player.maxSpeed`).
- For an enemy to drift *down the screen toward player*, their `vz` < player's world speed.
- For an enemy to *rise on screen* (rare — used in slasher attack), `vz` > player's world speed.
- **Screen Y** uses `zToScreenY(z) = getPlayerScreenY() - (z - trackPos)`. Higher Z = higher up on screen.
- Bullets travel with positive `vz` (forward).
- **DO NOT use negative vz to make enemies "come at" the player.** That was an early bug — enemies appeared to plummet because their effective screen-Y speed was player_speed + enemy_abs_speed.

## Player screen position (dynamic)

The player's car render position **moves up the screen with speed**:
- `PLAYER_SCREEN_Y_BASE = PLAY_BOTTOM - 70` (at speed 0 or near-cruise)
- `PLAYER_SCREEN_Y_LIFT = 60` (max upward shift at full speed)
- Lift activates above 30% speed, scales linearly to 95% speed
- `getPlayerScreenY()` returns the current dynamic position
- Critical: **all projection functions use `getPlayerScreenY()`, not the base constant**

The lift gives forward visibility at speed AND opens up rear-view space below the player for:
- Watching cars you've passed
- Watching chaser enemies catch up
- Drop zone for oil slicks and (future) smoke screen

## Steering band

- Bottom 170px of the screen is the dedicated steering zone
- Drag horizontally → player.x (lateral position)
- Drag vertically within the band → player.speed (top of band = fast, bottom = slow)
- Touch only starts if it begins within the band
- A subtle visual panel with FAST/SLOW labels and a touch indicator (crosshair when dragging)

## Player car

- Anchored at `getPlayerScreenY()` (no Y change from input — only speed-based lift)
- `player.x` is pixel offset from road center (not normalized)
- `player.vx` tracked frame-to-frame for **bumping** mechanics
- `player.alive` — false during crash sequence; truck rescue restores it
- `player.invincibleTimer` — flicker render and immunity to collisions

## Weapon buttons (left-side vertical column)

Bernie steers with right thumb, so all action buttons are on the left:

```
[MSL]   ← Missiles (not yet implemented in gameplay)
[SMK]   ← Smoke Screen (not yet implemented)
[OIL]   ← Oil Slick (v1.1+) — tap to drop slick behind
[GUNS]  ← Always-available auto-fire while held
```

- Each shows ammo count when active ("OIL 3")
- Disabled buttons render at 0.32 opacity
- Refreshed each frame via `refreshWeaponButtons()` based on `player.ammo`
- Cap: 5 per weapon type
- Upper buttons sit ABOVE the steering band, over the left grass area (acceptable since left grass is dead visual space)

## Road

- Straight gray asphalt ribbon (NOT pseudo-3D curves — that was an early version. The original arcade road is essentially top-down straight with width variations.)
- White solid edge lines
- Two dashed yellow lane dividers at ±33% of half-width
- Bright Spy Hunter green grass on both sides (`#2da44a`)
- Subtle darker green motion stripes (`#249339`) that scroll with `trackPos` for speed sensation
- Road **width** varies via `roadHalfWidthAt(z)`:
  - Default: 90px half-width
  - Narrow: 60px (pinch points)
  - Wide: 120px (room for weapons van delivery — future)
  - Smoothstep transitions between
- Width events generated procedurally ahead of player (`maybeAddRoadEvents`)

## Enemies — overview

All enemies live in `enemies[]`. Each has:
- `type`: 'switchblade' | 'bulletproof' | 'sedan' | 'taxi'
- `faction`: 'enemy' | 'civilian'
- `state`: 'alive' | 'spinning'
- `aiState`: 'stalker' | 'slasher' | 'blocker' | 'chaser'
- `bumped`: true after player bump (changes physics)
- `bulletproof`: true for armored sedans
- `hp` / `maxHp`: hit points
- `warningHits` / `warningTimer`: for civilian shoot-warning escalation

### Switchblade (enemy)
- Dark blue sedan with chrome wheel blades
- 3 HP — takes 3 bullets to kill
- AI cycles: stalker (40%), slasher (40%), blocker (20%)
- Speed ratios cycle every 0.6-1.5 sec:
  - Stalker: 72-86% of player speed
  - Slasher: 88-98% (briefly pacing alongside)
  - Blocker: 60-72% (slows to force ram)
- **Blades** = side-swipe attack: if their `vx` is closing toward player (>30 px/s) AND X-overlap is in side-swipe band, **instant crash on player**
- Spawn vz: 75-83% of player speed at spawn (so they're cars you're catching up to)
- 3 hits → spin out in random shallow direction (15-30° off-straight). Tree collision during spin = +300 bonus

### Bulletproof car (enemy, Level 2+)
- Drab olive armored sedan with chrome bumper and narrow slit windows
- **Bullets BOUNCE** — only kill is bumping off the road
- AI: stalker (60%) or blocker (40%) — no slasher (no blades)
- Slower than Switchblade (62-80% of player speed)
- When bumped, enters `bumped` state — `vx` cap increases, edge clamping disabled
- Bumped Bulletproof off-road for 0.5+ seconds → tumble + explode (+500 score)
- Tree collision while bumped → instant explode (+500)
- Hard bumps push them harder but DO NOT instant-kill — has to go off-road

### Sedan (civilian)
- Friendly sky-blue, no blades
- Drives at 50-65% of player speed
- Crashing into them = full player crash
- Shooting them — 3-hit warning system:
  - Hit 1: white spark, slight wobble, no penalty
  - Hit 2: orange spark, slight knock, no penalty (final warning)
  - Hit 3: -500 score penalty, spin-out down the lane, "⚠ CIVILIAN HIT ⚠" flash
- Warning state resets after 2.5 sec without follow-up shot

### Taxi (civilian)
- Yellow with black checker stripe and roof sign
- Drives at 70-80% of player speed (faster than sedans)
- Same shooting penalty system as sedans

### Chaser AI (passed enemies)
Once an enemy is passed (trackPos - e.z > 30), they "roll" for chase mode:
- Switchblade: 30% chance → boosts to 108% player speed, slasher attack from behind
- Bulletproof: 50% chance → boosts to 106% player speed, ram attempt from behind
- Once in chase mode, they stay in chase mode (no further state cycling)
- Bullet collision still detected for Switchblades
- This is what makes oil slicks essential

## Trees
- Procedural placement in grass on both sides of road
- Two kinds: chunky pine trees (70%) and bushes (30%)
- Random sizing 0.85-1.25× scale, random Z spacing 22-57
- Occasional 2-3 tree clusters (12% chance per row)
- Player collision = instant crash
- Spinning Switchblade collision = +300 (during spin-out)
- Bumped enemy (Switchblade or Bulletproof) tree collision = +300 or +500

## Bumping mechanic

Critical risk/reward layer.

- **CRASH zone**: `|dx| < 19` AND `|dz| < 25` — player slammed enemy, player dies
- **BUMP zone**: `19 ≤ |dx| < 28` AND `|dz| < 25` — side-swipe contact
- Bump only registers if player has positive closing velocity (`playerClosingVx > 5`)
- Switchblade bumps:
  - Soft bump: push them sideways (with `bumped = true`)
  - Hard bump (closing >60): instant spin-out, +75 score
- Bulletproof bumps:
  - All bumps push them sideways (harder force for harder bumps), `bumped = true`
  - **NO instant spin-out** — they must go off-road or hit a tree
- Civilian bumps:
  - Light shove, wobble, no penalty, no spin-out
- Player gets recoil (`player.x -= bumpDir * 4` for Switchblades, `* 5` for Bulletproof)

## Crash & rescue sequence

When player dies (and lives > 0):
1. `triggerCrash` — explode particles, spawn wreck (rotating debris), `player.alive = false`, `gameMode = 'crashing'`, modeTimer = 1.2
2. Wreck scrolls down the road with random rotation + flames + smoke
3. modeTimer expires → if lives <= 0, `endGame()`; else `spawnTruck()`
4. Truck enters from top of screen with `vz = -160` (rescue truck moves toward player)
5. Decelerates as it approaches its rest position (`zOffset = 45` relative to player)
6. **Truck position is tracked as a Z-OFFSET, not absolute world Z** — because trackPos keeps moving during the rescue. Don't change this.
7. Ramp opens (visible black + ramp triangle)
8. After 0.8 sec, player respawns at truck X with 2.5 sec invincibility
9. Truck pulls away, eventually despawns

## Weapons Van (mid-play, Level 2+)

Separate from rescue truck. Lives in `weaponsVans[]`.

- Spawns every 30-45 sec at Level 2+
- Tracks as **absolute world Z** with `vz` (like enemies)
- Side panel shows weapon type (oil/missiles/smoke) with colored band + glyph + text
- States:
  - `'cruising'`: drives at 85% player speed
  - `'waiting'`: slows further when player closes within 80 Z, ramp drops
  - `'closing'`: ramp up after 4 sec timeout (if player didn't engage)
  - `'leaving'`: accelerate forward and despawn
  - `'delivered_exit'`: faster acceleration after successful delivery
- Player drives INTO the trailer rear (within 18px X of van, 20 Z of trailer rear) → `deliverWeapon()`
- Delivery: +3 ammo (capped at 5), +250 score, 1 sec invincibility
- If `player.ammo[weaponType] >= ammoMax` already, delivery skipped — player can let truck pass

## Oil Slick (v1.1)

- `oilSlicks[]` — each slick is a world entity scrolling with the road
- Dropped by tapping OIL button (one-shot, not held)
- Spawns BEHIND player at current X (`z: trackPos - 18`)
- Visual: dark puddle with sheen highlight, fades out near end of 6-sec life
- Enemy collision (`!= civilian && state == alive`): spins them out
- Each slick tracks `_hitEnemies` Set so it only affects each enemy once
- Civilians are exempt (no penalty for trapping civilians)
- Bonus: +75 for Switchblade kill, +250 for Bulletproof

## Level system

- `currentLevel` increments every 6000 trackPos units (~45 sec at cruise speed)
- Big yellow "LEVEL N" banner pops up for 2 sec on level-up
- `levelConfig(lvl)` returns:
  - `bulletproofUnlocked`: lvl >= 2
  - `bulletproofChance`: ramps up from 10% at lvl 2 to 30% at lvl 7+
  - Spawn rate increases slightly per level
- HUD has "LVL" indicator between SCORE and SPEED

## HUD

Top row: SCORE | LVL | SPEED | LIVES (▲ icons)

Plus dynamic elements:
- `#warning` — red box for "CIVILIAN HIT" alerts
- `#level-banner` — center-screen level-up announcement
- Yellow Courier-style monospace font throughout

## Roadmap (not yet built)

- **v1.2: Missiles** — forward-firing one-shot weapon, kills Bulletproof, takes ammo
- **v1.2: Smoke Screen** — deployable behind, blinds chasers for a few seconds
- **v1.2: Helicopters** — fly over the road, drop bombs that leave craters; only killable with missiles
- **v1.3: Random weapon type** in weapons van — instead of always oil, the truck picks randomly. Player decides whether to catch based on side label.
- **v1.4: Road forks** — periodic side-road branches the player can choose between
- **v1.5: Sound** — Peter Gunn theme via Web Audio, plus SFX for guns/explosions/crashes
- **v1.6: Boat section** — periodic transition where road becomes water, Dr. Torpedo enemies, etc.

## Open known issues / things to verify

- Civilians don't visually swerve around oil slicks (currently they just drive through without any reaction) — TODO: add a small lane-shift when civilian is near a slick
- "Drove into truck" detection currently uses a Z proximity (`|trackPos - trailerRearZ| < 20`) — might be missable at very high or very low speeds. Could need tuning.
- Bulletproof car off-road timer (0.5 sec) — Bernie may want to tune this. Currently feels right but watch for "got bumped, drifted back on road, no kill" frustration.
- The `_chaserChecked` flag is set once per enemy lifecycle; an enemy that drifts back ahead of the player won't re-check. That's intentional but Bernie may want it to be re-checkable.

## Bernie's preferences (carry these forward!)

- Loves the Janet personality
- Wants to see the iteration loop fast — build, play, comment
- Prefers gameplay feel over technical specs
- "Vibe coding" — don't over-architect, do the simplest thing that works
- Likes mechanics that have visible feedback (sparks, smoke, damage tints)
- Prefers iPhone portrait play
- Bernie's name convention: project names get a B somewhere (`btempest`, `sbyhunter`)
