# Trainer Scene

## Summary
This repo now includes a `TRAINER SCENE` that hijacks a chosen level and runs timing drills in the normal game loop.

Default hook:
- `World 1-1` (`WorldNumber = 0`, `LevelNumber = 0`)

Compile-time toggles (in `common/practice.asm`):
- `ENABLE_TRAINER_SCENE` (default `1`)
- `TRAINER_HIJACK_WORLD` (default `World1`)
- `TRAINER_HIJACK_LEVEL` (default `Level1`)

Set `ENABLE_TRAINER_SCENE` to `0` to disable the feature in one place.

## Controls
- `SELECT`: cycle drill (`A-only` -> `Hold-B-then-A`)
- `START`: reset trainer stats + reset attempt
- `START + SELECT`: return to title

## Drills
- Drill A: press `A` within `W` frames at the START gate.
- Drill B: hold `B` for exactly `N` frames starting at the START gate, then press `A` within `W` frames.

Defaults:
- `N = 3`
- `W = 2` (inclusive window)

## Frame Semantics
- Input source is the same controller poll used by gameplay (`ReadJoypads`, then `SavedJoypad1Bits`).
- Trainer logic runs once per game frame in the normal game update cadence (hooked in `lost/lost.asm` game engine path).
- `A` detection is edge-triggered (`new press`, not held).
- Attempt reset is in-scene only (no ROM reboot): marker + FSM + player position are reset.

## HUD and Visuals
- START gate and moving marker are rendered as sprites.
- HUD is written through `VRAM_Buffer1` in `RedrawUserVars`, so updates are vblank-safe.
- HUD fields:
  - drill, `N`, `W`
  - last result (`HIT`, `EARL`, `LATE`, `REL`, `LONG`)
  - `H` hits, `M` misses, `A` accuracy %, `S` streak, `B` best streak

## Build / Run
Use existing repo flow:

```bash
make -j4
```

Optional emulator run target:

```bash
make run
```

## Self-test Checklist
1. Build succeeds (`make -j4`).
2. Enter hijacked level (`World 1-1` by default) and verify trainer HUD appears.
3. Verify marker reaches gate and defines start frame consistently.
4. Drill A:
   - press `A` before gate -> `EARL`
   - press in window -> `HIT`
   - press after window -> `LATE`
5. Drill B:
   - release `B` early -> `REL`
   - keep holding `B` too long -> `LONG`
   - hold exactly `3`, then `A` within `2` -> `HIT`
6. After any result, verify auto-reset after short delay and consistent start position.
7. Press `START`: attempt + stats reset.
8. Press `SELECT`: drill cycles and attempt resets.
9. Press `START+SELECT`: returns to title.
10. Enter a non-hijacked level and verify normal gameplay is unchanged.

## Hook Map (Discovery + Changes)
Primary runtime symbols/hooks used:
- NMI/frame cadence: `lost/lost.asm` `NonMaskableInterrupt` path (`Enter_PracticeOnFrame`, then `OperModeExecutionTree`)
- Controller poll: `common/game.asm` `ReadJoypads`, stored in `SavedJoypadBits`/`SavedJoypad1Bits`
- Main gameplay loop: `lost/lost.asm` `GameMode` -> `GameCoreRoutine_RW` -> `GameEngine`
- Level-load hook: `lost/lost.asm` `SecondaryGameSetup` -> `Enter_ProcessLevelLoad`
- VBlank HUD path: `common/practice.asm` `RedrawUserVars` with `VRAM_Buffer1`

Files changed for trainer implementation:
- `common/practice.asm`
- `lost/lost.asm`
- `inc/macros.inc`
- `common/common.asm`
- `docs/trainer.md`
