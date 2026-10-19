# Browser test prompts for the playcanvas-expert skill

Run each prompt in a FRESH agent session (OpenCode / Codex / Claude Code) with the skill installed and
the starter project open. Score every item 0/1 from what you SEE in a real browser, not from the agent's claims.

## Prompt 1: product viewer
"Build a premium product viewer for a glTF/GLB model I'll drop in /public/model.glb. Studio lighting, soft shadows,
orbit with damping, auto-rotate when idle, loading screen, must hit 60 fps on a mid phone."

## Prompt 2: splat viewer
"Make a Gaussian splat viewer for /public/scene.sog with orbit controls, a fast first frame, a quality toggle
(mobile / desktop splat budget) and a cinematic fade-in. Keep it smooth on phones."

## Prompt 3: physics game
"Make a small 3D marble-run game: tilt-able board (mouse/touch), a ball with real physics, 3 collectible
glowing pickups with bloom, a timer and a win screen. Polished look, no external art assets."

## Checklist (score each prompt)
| # | Check | P1 | P2 | P3 |
|---|-------|----|----|----|
| 1 | Used installed version, no v1 APIs (`entity.model`, `BasicMaterial`, `setGravity`, `Application` on WebGPU) | | | |
| 2 | Registered every component system/handler it uses (no null `addComponent`) | | | |
| 3 | Zero console errors/warnings in dev (debug build) | | | |
| 4 | Not black/blank; correct scale, orientation, framing | | | |
| 5 | HDR look: tone mapping, IBL or good lighting, shadows, subtle bloom | | | |
| 6 | Interaction works on mouse AND touch; resize works | | | |
| 7 | Frame rate >= 60 desktop; no per-frame allocations; DPR capped | | | |
| 8 | Splats: antialias off, budget set, orientation flip handled (P2) / Ammo loaded before app (P3) | | | |
| 9 | Agent verified in a browser (screenshot/console/fps) and reported honestly what it could not check | | | |
| 10 | `npm run build` passes; README run steps work | | | |
| 11 | Reused shipped scripts (CameraControls etc.) instead of rewriting | | | |
| 12 | Would a stranger say "wow" in the first 5 seconds? (subjective 0-1) | | | |

## Reading results
Score < 10/12 on any prompt: note which check failed and which skill section should have prevented it,
then edit that section (add a rule, a code block, or a triage-table row) and re-run only the failing prompt.
