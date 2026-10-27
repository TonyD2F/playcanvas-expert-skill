# PlayCanvas Expert Skill

An agent skill for building high-end browser 3D with the **PlayCanvas Engine v2** (WebGL2 + WebGPU + WebXR): games, product/architecture viewers, data visualisation, and Gaussian splat experiences.

Verified against `playcanvas@2.23.3` (Oct 2026). Written because most public samples and model memory are v1-era, and the v2 API breaks them.

## What it enforces

- **Version-grounded work** — inspect the installed engine and typings before writing code; never emit removed v1 APIs.
- **Reuse over invention** — start from the closest shipped example or engine script.
- **Visual quality bar** — HDR, IBL, tone mapping, shadows, bloom, deliberate camera. A grey cube under a default light is a failure.
- **Runtime verification** — a compiling scene can still be black, upside-down, 1000× too big, or 4 fps. Test in a real browser.
- **Tree-shakeable bootstrap** — `createGraphicsDevice` + `AppBase`, or `Application` for prototypes.
- **Splats and XR covered** — 3DGS/SOG/PLY viewers, `splatBudget`, WebXR, `@playcanvas/react`, and web components.

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | The skill: authoring surfaces, bootstrap, scripting, look-dev, splats, physics, animation, XR, performance, headless verification, v1→v2 pitfalls |
| `browser-test-prompts.md` | Three scored end-to-end prompts (product viewer, splat viewer, physics game) for testing an agent |
| `playcanvas-starter.zip` | Vite starter: `src/app.js`, `src/look.js` (CameraFrame + lights), `src/scene.js`, `tests/logic.test.mjs`, `tools/verify-browser.mjs`, `evals/prompts.md` |

## Install

```bash
# OpenCode
mkdir -p ~/.config/opencode/skills/playcanvas-expert
cp SKILL.md ~/.config/opencode/skills/playcanvas-expert/

# Claude Code
mkdir -p ~/.claude/skills/playcanvas-expert
cp SKILL.md ~/.claude/skills/playcanvas-expert/

# Starter project
unzip playcanvas-starter.zip && cd pc-starter && npm install && npm run dev
```

The skill also points at the official PlayCanvas agent skills (`npx skills add playcanvas/skills`); use them alongside it — this one is the cross-cutting playbook, those are task-specific.

## How to use it

Give the agent a concrete deliverable and target devices, for example:

> Build a premium product viewer for a GLB at `public/model.glb`. Studio lighting, soft shadows, orbit with damping, auto-rotate when idle, loading screen, 60 fps on a mid-range phone.

The skill makes the agent inspect the installed engine version, reuse the nearest example, build the smallest vertical slice, then layer look-dev and interaction while verifying each step in a browser.

## Test the skill

Run `browser-test-prompts.md` in fresh agent sessions with the starter open. Score each item 0/1 from what appears in a real browser, not from the agent's claims. The checklist covers correct v2 APIs, asset loading, visual quality, frame time, mobile behavior, and error handling.

## Key v2 pitfalls it prevents

`entity.model` → `render` · removed `pc.BasicMaterial` · WebGL1 · `pc.script.create` · `Scene.gammaCorrection` / `toneMapping` · `Scene.fogColor` · `Application` on WebGPU · old `playcanvas-stable.min.js` CDN · `pc.createScript` in new code · `ShaderMaterial` without explicit gamma/tonemap/fog · global `pc.*` in module code.

## Links

- Engine: https://github.com/playcanvas/engine
- Manual: https://developer.playcanvas.com/user-manual/
- API: https://api.playcanvas.com/engine/
- Examples: https://playcanvas.com/examples
- Gaussian splatting: https://developer.playcanvas.com/user-manual/gaussian-splatting/

## License

MIT © 2026 TonyD2F — see `LICENSE`.
