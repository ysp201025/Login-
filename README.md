# Replay System — Bedrock Add-On

A Replay-Mod-style recording & playback system for Minecraft **Bedrock**
Edition, built entirely on the official `@minecraft/server` Script API
(no external tools, no client mods).

Read **"How this differs from Java's Replay Mod"** below before you judge
anything as missing — several things on the original wishlist aren't
possible from inside a Bedrock add-on sandbox, and this doc tells you
exactly which ones and why, rather than quietly skipping them.

## Install

1. Unzip `replay-system.mcaddon` if your OS doesn't import it directly, or
   just double-click the `.mcaddon` file — Minecraft should offer to import
   both packs (a `[BP]` behavior pack and `[RP]` resource pack).
2. In your world settings, turn **on**:
   - Behavior pack: **Replay System [BP]**
   - Resource pack: **Replay System [RP]**
   - **Cheats: ON** (required — the add-on uses `/inputpermission`,
     `/camerashake`, `/hud`, and `/camera`-family commands internally)
3. Load the world. You should see a chat message confirming the add-on
   loaded.
4. Open the menu with **`!replay`** in chat, or try the slash command
   **`/replay:menu`** (see Troubleshooting if that one doesn't show up).

## Command reference

| Command | Effect |
|---|---|
| `!replay` / `!replay menu` | Open the main menu |
| `!replay start [name] [all]` | Start recording (`all` = every online player) |
| `!replay pause` / `!replay resume` | Pause/resume the active recording |
| `!replay marker <label>` | Drop a marker at the current recording time |
| `!replay stop` | Stop recording and save |
| `!replay cancel` | Stop recording and discard it |
| `!replay browse [search]` | Open the replay browser, optionally filtered |
| `!replay view <name>` | Load and start viewing a saved replay by name |
| `!replay stopview` | Stop viewing the current replay |
| `!replay keyframe add [label]` | Capture your current position as a camera keyframe |
| `!replay keyframe clear` | Clear captured keyframes |
| `!replay keyframe play [seconds]` | Fly the cutscene camera through your keyframes |
| `!replay help` | List commands in-game |

Everything above is also reachable from the `!replay menu` UI, which is
generally the easier way in — the chat commands exist mainly for the
keyframe-camera workflow (fly around, drop points, play the path) since
that's inherently a "walk around and press a button" motion that doesn't
suit a menu.

## How this differs from Java's Replay Mod (read this first)

Java's Replay Mod hooks the game's renderer directly. Bedrock add-ons
never get that level of access — they're server-side JavaScript plus JSON,
running against the `@minecraft/server` Script API. Concretely:

- **No filesystem access.** An add-on cannot write a `.json` or `.zip`
  file to disk. Replays are saved as World Dynamic Properties, which live
  *inside* the world save — durable across reloads/restarts, but not a
  portable file. "Export" gives you a copy-able JSON dump in chat instead
  (fine for short replays, unwieldy for long ones). True "Import from a
  file" isn't implemented for the same reason.
- **No render-pipeline hooks.** Motion blur and depth-of-field are not
  implemented at all — not even faked — because there's no post-processing
  access to approximate them with. Anything claiming to do this in a
  Bedrock add-on is cosmetic at best.
- **No screenshot API.** Thumbnails aren't possible.
- **No per-player animation state.** The API exposes pose booleans
  (sneaking/sprinting/swimming/flying) but not swing/eat/draw-bow frames,
  and no separate head-vs-body rotation for players (only one view
  rotation). Ghosts reflect pose, not fine animation.
- **No vanilla particle event hook.** Only particles *this add-on*
  generates (its own block-change callouts) can be shown; particles from
  vanilla game systems (block-break dust, potion swirls, etc.) aren't
  observable by scripts.
- **Ghosts are `armor_stand` entities**, not full player models — they
  don't play walk-cycle animations, but they do carry the recorded
  position, rotation, held item, and armor.
- **Recorded block changes are not applied to the real world.** Replaying
  a `break`/`place` onto the live world would permanently edit it under
  the viewer's feet. Instead it surfaces as a particle burst + action-bar
  text at the right moment — informational, non-destructive.
- **Recorded weather/time are shown as read-only info**, not applied to
  the live world (which would disrupt every other player in it).
- **Controller support needed no extra work.** Bedrock's form UI
  (`ActionFormData`/`ModalFormData`) already supports controller/gamepad
  navigation natively — nothing to build.
- **"Free Camera" uses Spectator game mode**, not the free-camera preset —
  the preset stays bound to the viewer's own body, so raw WASD doesn't
  steer it. Spectator mode is Bedrock's real no-clip fly camera.

## Feature matrix

✅ = fully implemented · 🟡 = implemented as a best-effort approximation
(see note) · ❌ = not implemented, not possible from an add-on

**Recording:** Position ✅ · Rotation 🟡 (one rotation, shared by head/body —
see above) · Sneak/Sprint/Swim/Fly ✅ (feature-detected per engine version)
· Jump 🟡 (same) · Held item ✅ · Armor ✅ · Fine animations ❌ · Mob/entity
movement 🟡 (throttled, optional, shown as plain markers) · Block
place/break 🟡 (recorded + surfaced as callouts, not visually rebuilt) ·
Chat ✅ · Weather/time ✅ (recorded, shown as info only) · Scoreboard ✅ ·
Particle events ❌

**Replay menu:** Start/Pause/Resume/Stop ✅ · Save ✅ · List/Rename/Delete ✅
· Export 🟡 (chat JSON dump) · Import ❌

**Camera:** Free (Spectator) ✅ · Orbit ✅ · First/Third person ✅ · Dolly/
Crane 🟡 (both are the same generic keyframe-path system — 2 or more
points) · Path/Keyframe ✅ · Smooth interpolation ✅ · Ease in/out ✅ · Speed
control ✅ · FOV control ✅

**Timeline:** Frame-by-frame ✅ (single-tick step) · Speed 0.25×–8× ✅ ·
Reverse ✅ · Jump to timestamp ✅ (slider, seconds-granularity) · Markers ✅
· Drag-scrubber timeline bar ❌ (no drag UI exists in Bedrock forms; the
slider is the closest available substitute)

**Cinematic:** Motion blur ❌ · Depth of field ❌ · Camera shake ✅ · Smooth
interpolation ✅ · Ease in/out ✅ · Mode transitions 🟡 (currently instant,
not eased)

**HUD:** Hide HUD ✅ · Hide specific player ghost ✅ · Hide nametags ✅ ·
Hide (our own) particles ✅ · Hide mob ghosts ✅ (hidden by default, opt-in)
· Hide armor on ghosts ✅

**Multiplayer:** Record all players ✅ · Spectate/follow any recorded
player ✅

**Export:** JSON data 🟡 (chat dump) · ZIP package ❌ · Thumbnail ❌ ·
Metadata ✅

**Extras from your list:** Replay browser ✅ · Search ✅ · Favorites ✅ ·
Backup 🟡 (duplicate copy inside the same world save, not external/cloud)
· Controller support ✅ (native, free) · Settings 🟡 (folded into
contextual menus rather than one global settings screen)

## Architecture

```
BP/scripts/
  config.js          tunables (sample rates, speed steps, chunk size...)
  utils.js            interpolation, easing, Catmull-Rom, safe-call helpers
  storage.js           dynamic-property save/load, chunking, replay index
  recorder.js          tick-sampling recording engine + event capture
  cameraDirector.js    camera math + Script API camera calls per mode
  playback.js           ghost entities, timeline playback, camera driving
  ui/menus.js           ActionFormData/ModalFormData menu tree
  main.js               entry point: command registration, wiring
RP/                    companion resource pack (currently minimal)
```

## Troubleshooting

- **`/replay:menu` doesn't autocomplete or errors out:** Custom slash
  commands are a newer API surface and may need the **Beta APIs**
  experiment toggle on your world, depending on your engine version. The
  `!replay` chat commands need no toggles at all and work identically —
  use those if in doubt.
- **Camera doesn't move / errors in content log about `camera.setCamera`:**
  Some engine versions have gated the free-camera preset behind an
  **Experimental Creator Camera Features** toggle. Enable it and reload.
- **Pack won't load / "dependency not found" for `@minecraft/server`:**
  Minecraft's script module versions bump often. Open `BP/manifest.json`
  and bump the `"version"` string under the `@minecraft/server` /
  `@minecraft/server-ui` dependencies to whatever your game's content log
  reports as available (it names the version it expected/found on load
  failure).
- **Long recordings feel laggy:** lower `RECORD_SAMPLE_INTERVAL`'s
  opposite — raise it (record every 2nd/3rd tick instead of every tick) in
  `config.js`, and lower `ENTITY_SCAN_RADIUS` if mob-movement capture is on.

## Extending this

This is a solid, working core, not a finished 60-feature product — that
would be several weeks of work even without Bedrock's constraints (the
real Replay Mod took years). The places most worth extending first,
roughly in order of value:
1. Eased transitions when switching camera modes mid-playback.
2. A proper "dolly"/"crane" preset UI (currently both are just the generic
   keyframe path — could add one-tap "auto-generate a crane arc between
   here and there").
3. Persisting keyframe sets alongside a replay (currently keyframes live
   only in memory for the current viewing session).
