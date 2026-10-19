---
name: playcanvas-expert
description: Expert use of the PlayCanvas Engine (github.com/playcanvas/engine, npm "playcanvas" v2.x) to build breathtaking browser 3D - games, product/archviz viewers, data visualisation, WebXR, and Gaussian splat (3DGS/SOG/PLY) experiences on WebGPU/WebGL2. Use whenever the user mentions PlayCanvas, pc.Application, AppBase, Entity/Component, ESM Script, CameraFrame, gsplat, SOG, SuperSplat, splat-transform, @playcanvas/react, @playcanvas/web-components, or wants a web 3D game/scene/viewer/splat app, even if they never name the engine. Covers version-safe setup, scripting, glTF, PBR look-dev and post-processing, splats, physics, animation, XR, performance, headless verification and a v1-to-v2 pitfall list. Prefer over training-data memory: the v2 API differs from most online samples.
---

# PlayCanvas Engine Expert

Target: PlayCanvas Engine v2 (verified against npm `playcanvas@2.23.3`, Oct 2026; MIT). Open-source WebGL2 + WebGPU + WebXR runtime with glTF 2.0, Draco/Basis, ECS, ammo.js physics, state-based animation and first-class Gaussian splatting.

## 0. Prime directives (read first, obey always)

1. **Ground in the installed version, not memory.** The engine ships a release roughly every 4-6 weeks and most public samples are v1-era. Before writing code:
   - `npm ls playcanvas --depth=0` (or `npm view playcanvas version` for latest). `package.json` is not importable via `require/import` (ERR_PACKAGE_PATH_NOT_EXPORTED); read it with `fs`.
   - Grep the typings for any symbol you are not 100% sure of: `node_modules/playcanvas/build/playcanvas/src/**/*.d.ts` (also `build/playcanvas.d.ts`). If a symbol is absent, it does not exist in this version.
   - API docs and manual pages are available as clean Markdown by appending `.md`: `https://api.playcanvas.com/engine/classes/CameraFrame.md`, `https://developer.playcanvas.com/user-manual/graphics/linear-workflow/hdr-rendering.md`. Prefer these over HTML.
2. **Reuse before writing.** The package ships production scripts in `node_modules/playcanvas/scripts/esm/` (see section 9) and the repo ships examples (`examples/src/examples/<category>/*.example.mjs`, live at https://playcanvas.com/examples). Find the closest example first and adapt it.
3. **Official agent skills exist.** `npx skills add playcanvas/skills` (or `npm create playcanvas@latest` which bundles them under `.claude/skills/` and `.agents/skills/`) installs Engine workflows: build-app, apply-conventions, find-examples, reuse-scripts, inspect-glb, calibrate-model, configure-animation, assemble-scene, light-scene, add-effects, build-hud, manage-game-state. If present, use them alongside this skill; this skill is the cross-cutting playbook, theirs are task-specific.
4. **Verify in a running app, never by reading code.** Section 12. A scene that compiles can still be black, upside-down, 1000x too big or 4 fps.
5. **Aim for "breathtaking", not "working".** Section 7 is the quality bar. Default to HDR, IBL, tone mapping, shadows, bloom and a deliberate camera. A grey cube under a default light is a failure.
6. **Never use removed v1 APIs** (section 14). Debug builds warn on misuse: develop against `playcanvas/build/playcanvas.dbg.mjs`, ship `playcanvas` (production strips `Debug.*`).

## 1. Pick the authoring surface

| Need | Use |
|---|---|
| Max control, game/app logic, custom pipelines, splat viewers | **Direct Engine** (Vite + JS/TS). Default. |
| UI-heavy app already in React | `@playcanvas/react` (JSX entities, hooks such as `useModel`, `useEnvAtlas`, `usePhysics`) |
| Static page, no build tools, declarative | `@playcanvas/web-components` (`<pc-app>`, `<pc-entity>`, `<pc-gsplat>`, `<pc-rigid-body>`) |
| Artists/designers, cloud hosting, collaboration | PlayCanvas Editor (code via VS Code extension, not this workflow) |

Scaffold: `npm create playcanvas@latest my-app -- -f engine` (Vite + TypeScript + agent skills). Manual: `npm i playcanvas && npm i -D vite`, then `npx vite`.
CDN/no-build: import map `"playcanvas": "https://cdn.jsdelivr.net/npm/playcanvas/+esm"` (pin a version for production).

## 2. Bootstrap (copy, then trim)

Recommended modern setup: explicit device + `AppBase` + only the systems/handlers you use (tree-shakeable, can use WebGPU). `Application` registers everything, cannot create a WebGPU device and is expected to be deprecated; it is fine for quick prototypes and is what the official splat tutorial uses.

```js
import * as pc from 'playcanvas';

const canvas = document.getElementById('application');
const device = await pc.createGraphicsDevice(canvas, {
  deviceTypes: [pc.DEVICETYPE_WEBGPU],   // falls back to WebGL2 automatically
  antialias: false,                      // FALSE for splats (official perf guidance); with CameraFrame the scene renders to its own target, so use frame.rendering.samples / TAA instead; true only for plain LDR mesh scenes
  powerPreference: 'high-performance'
  // glslangUrl + twgslUrl ONLY if you ship custom GLSL shaders/chunks on WebGPU (built-ins are native WGSL)
});

const o = new pc.AppOptions();
o.graphicsDevice = device;
o.componentSystems = [
  pc.RenderComponentSystem, pc.CameraComponentSystem, pc.LightComponentSystem,
  pc.ScriptComponentSystem, pc.GSplatComponentSystem          // add per component used (below)
];
o.resourceHandlers = [
  pc.TextureHandler, pc.ContainerHandler, pc.ScriptHandler, pc.GSplatHandler
];
const app = new pc.AppBase(canvas);
app.init(o);
app.setCanvasFillMode(pc.FILLMODE_FILL_WINDOW);
app.setCanvasResolution(pc.RESOLUTION_AUTO);
app.graphicsDevice.maxPixelRatio = Math.min(window.devicePixelRatio, 2); // cap DPR on phones
window.addEventListener('resize', () => app.resizeCanvas());
app.start();
```

Component to register map (missing system => `addComponent` returns `null`, debug build logs an error):
`render`->RenderComponentSystem, `camera`->CameraComponentSystem, `light`->LightComponentSystem, `script`->ScriptComponentSystem + ScriptHandler, `gsplat`->GSplatComponentSystem + GSplatHandler, `anim`->AnimComponentSystem + AnimClipHandler + AnimStateGraphHandler, `rigidbody`/`collision`/`joint`->RigidBodyComponentSystem/CollisionComponentSystem/JointComponentSystem, `sound`->SoundComponentSystem + AudioHandler + `o.soundManager = new pc.SoundManager()`, `particlesystem`->ParticleSystemComponentSystem, `screen`/`element`/`button`->ScreenComponentSystem/ElementComponentSystem/ButtonComponentSystem + FontHandler/TextureAtlasHandler. Containers (GLB) need ContainerHandler; textures TextureHandler; cubemaps/env need TextureHandler. Batching, lightmapping and XR are enabled via further `AppOptions` fields: grep `framework/app-options.d.ts`.

Render-loop essentials: `app.on('update', dt => ...)`; `app.autoRender = false; app.renderNextFrame = true` for render-on-demand viewers (big battery/GPU saver for static splat/product viewers). Call `entity.destroy()`, `material.destroy()`, `texture.destroy()`, `app.destroy()` when removing things; unreleased GPU resources are the top cause of leaks.

## 3. Conventions that prevent silent bugs

- Right-handed, **+Y up, metres**, ground plane XZ. `entity.forward` and camera view direction = **-Z**; `right` = +X; `lookAt` points -Z at the target.
- Directional and spot lights shine along local **-Y**; aim with `setEulerAngles` (e.g. `(45, 30, 0)`), not `lookAt`.
- glTF/GLB models usually face +Z: expect a per-model yaw fix; measure (bounds, ground offset, orientation) once and reuse the correction. Never scale rigid-body roots or skinned bones.
- Angles in **degrees**, XYZ order. Store yaw/pitch/roll as your own state; never feed `getEulerAngles()` back into `setEulerAngles()`. Blend with `pc.math.lerpAngle` or `Quat.slerp`, never scalar lerp over the 0/360 seam.
- World setters on semantic roots, `setLocal*` below a parent.
- Hot loops (update, render callbacks, script `update`): **no allocations**. Hoist `new Vec3/Quat/Mat4` to module scope or script fields and reuse (`.copy()`, `.set()`, `.add2()`, `.mulScalar()`). No `try/catch`, closures or string concatenation in per-frame code.
- Colour textures (diffuse/albedo, emissive, sheen, specular) must be **sRGB**: when you create Assets for them pass `{ srgb: true }` in the 4th (data) argument or use `PIXELFORMAT_SRGBA8`; data textures (normal, roughness, metalness, AO) stay linear. GLB textures are handled by the glTF parser.
- Tint colours always apply in v2: to use an emissive map set `material.emissive = new pc.Color(1, 1, 1)` (default black multiplies it to nothing; this is the classic "black movie texture").
- After changing StandardMaterial flags/maps call `material.update()`.
- Physics: 1 unit = 1 m, 1 unit mass = 1 kg. Dynamic bodies are owned by physics (do not set their transform; use forces, `linearVelocity`, `teleport`).

## 4. Scripting (ESM, the supported path)

Use ESM `Script` classes (`.mjs`). Classic `pc.createScript` still works but is not for new code.

```js
import { Script, Vec3 } from 'playcanvas';

export class Bob extends Script {
  static scriptName = 'bob';           // name used by script.create('bob') and the Editor

  /** @attribute */ amplitude = 0.25;  // JSDoc @attribute exposes it to the Editor and typing
  /** @attribute */ speed = 2;

  _base = new Vec3(); _t = 0;

  initialize() { this._base.copy(this.entity.getLocalPosition()); }   // once, after all entities exist
  update(dt) {                                                        // every frame; no allocation
    this._t += dt * this.speed;
    this.entity.setLocalPosition(this._base.x, this._base.y + Math.sin(this._t) * this.amplitude, this._base.z);
  }
}
// usage
entity.addComponent('script');
entity.script.create(Bob, { properties: { amplitude: 0.4 } });
```

Lifecycle: `initialize` -> `postInitialize` -> per frame `update` -> `postUpdate`; plus events `this.on('enable' | 'disable' | 'destroy' | 'state' | 'error', ...)`, `this.on('attr', (name, newV, oldV) => ...)` and `this.on('attr:speed', (newV, oldV) => ...)`. Always unsubscribe `app.on/off`, `window`/`document` listeners in `destroy`. Scripts from files: `new pc.Asset('x', 'script', { url })` then `entity.script.create('scriptName')`.
Events: `this.app.fire/on` for cross-script messages; keep one global state/event bus script for game state.

## 5. Assets and glTF

```js
const assets = {
  car:  new pc.Asset('car', 'container', { url: 'models/car.glb' }),
  env:  new pc.Asset('env', 'texture', { url: 'env/studio.hdr' }, { mipmaps: false })
};
await new Promise(res => new pc.AssetListLoader(Object.values(assets), app.assets).load(res));
const car = assets.car.resource.instantiateRenderEntity();   // render components, correct for v2
app.root.addChild(car);
```

- Never use `instantiateModelEntity` / `entity.model` (legacy model component).
- Compression: **Draco** meshes and **Basis/KTX2** textures need their WASM modules. Check `pc.dracoInitialize` / `pc.basisInitialize` and `pc.WasmModule.setConfig(...)` in typings; self-host the files for production.
- Optimise models before they reach the browser: `gltf-transform optimize` (prune, dedup, weld, meshopt/Draco, WebP/KTX2), cap texture sizes (2k default, 4k hero only), aim for <50 draw calls on mobile.
- Splat GLBs load as **container** assets and expose `asset.resource.gsplats` (section 8).
- Scene hierarchy from the Editor: `new pc.Asset('scene','template' ...)` only for Editor exports; standalone apps build entities in code.

## 6. Physics (ammo.js, opt-in)

Nothing happens until ammo is loaded; components are inert placeholders and raise no errors (silent failure!). Load before creating the app, copy the three ammo files from `examples/assets/wasm/ammo/` or the `sync-ammo` package:

```js
pc.WasmModule.setConfig('Ammo', { glueUrl: 'ammo/ammo.wasm.js', wasmUrl: 'ammo/ammo.wasm.wasm', fallbackUrl: 'ammo/ammo.js' });
await new Promise(r => pc.WasmModule.getInstance('Ammo', () => r()));
// then add RigidBodyComponentSystem, CollisionComponentSystem (and JointComponentSystem) to AppOptions
app.systems.rigidbody.gravity = new pc.Vec3(0, -9.81, 0);       // setGravity() is deprecated
box.addComponent('collision', { type: 'box', halfExtents: new pc.Vec3(0.5, 0.5, 0.5) });
box.addComponent('rigidbody', { type: pc.BODYTYPE_DYNAMIC, mass: 1, friction: 0.6, restitution: 0.2 });
const hit = app.systems.rigidbody.raycastFirst(from, to);      // picking, ground checks
```

Rules: fixed 1/60 s steps with interpolation; contact/trigger events fire per step, not per frame; static/kinematic bodies are moved by you via the entity transform, dynamic by forces/velocity/`teleport`; keep shapes simple (box/sphere/capsule/compound before mesh); the `joint` component is alpha. In a React app pass `usePhysics`; in web components add `<pc-wasm name="Ammo" .../>`.

## 7. The "breathtaking" playbook (lighting, look, motion)

Quality bar checklist, in order of payoff:
1. **IBL first.** A single HDRI gives believable PBR instantly. Generate the env atlas and (optionally) a sky:
   ```js
   // `hdr` = loaded 'texture' asset (equirect HDR). From the official examples:
   const skybox = pc.EnvLighting.generateSkyboxCubemap(hdr.resource);
   app.scene.skybox = skybox;
   app.scene.envAtlas = pc.EnvLighting.generateAtlas(hdr.resource);   // or pre-baked RGBP atlas PNG via type: pc.TEXTURETYPE_RGBP
   app.scene.skyboxMip = 1;           // blur the background, keep sharp reflections
   app.scene.skyboxIntensity = 1;
   app.scene.sky.type = pc.SKYTYPE_DOME;   // grounds the horizon; also SKYTYPE_BOX / INFINITE
   ```
   Check the installed typings for `EnvLighting` signatures (they have changed across 2.x).
2. **One key light with shadows**, plus IBL as fill; rim light for separation. `light.addComponent('light', { type: 'directional', castShadows: true, shadowResolution: 2048, shadowDistance: 20, shadowBias/normalOffsetBias tuned, intensity: 1-3 })`. Tight `shadowDistance` = crisp shadows. Shadow types: `SHADOW_PCF3`, `SHADOW_PCF5`, `SHADOW_PCSS_32F` (soft), VSM variants. Use `light.shadowUpdateMode`/static shadows for static scenes.
3. **HDR pipeline with `CameraFrame`** (bloom, SSAO, DoF, TAA, grading, vignette, fringing, colour LUT, volumetric fog). Always call `frame.update()` after changing settings.
   ```js
   const frame = new pc.CameraFrame(app, cameraEntity.camera);
   frame.rendering.toneMapping = pc.TONEMAP_NEUTRAL;   // NEUTRAL (faithful colour), ACES/ACES2 (filmic punch), FILMIC, HEJL, LINEAR
   frame.rendering.samples = 4;                        // MSAA; set 1 when frame.taa.enabled = true
   frame.rendering.sharpness = 0;                      // 0-1, raise (~0.3-0.5) to counter TAA/low render scale softness
   frame.bloom.intensity = 0.012;                      // >0 enables bloom (there is NO bloom.enabled); keep subtle
   frame.ssao.type = pc.SSAOTYPE_COMBINE;              // NONE | LIGHTING | COMBINE; intensity 0-1, radius 0-100, samples 1-64
   frame.ssao.intensity = 0.5;
   frame.grading.enabled = true; frame.grading.contrast = 1.05; frame.grading.saturation = 1.1;
   frame.vignette.intensity = 0.35;                    // inner/outer/curvature/color available
   frame.fringing.intensity = 5;                       // chromatic aberration, 0-100, default 0 (off); start low
   frame.dof.enabled = false;                          // focusDistance, focusRange, blurRadius, blurRings; animate for rack focus
   frame.update();
   ```
   For bloom to read as light, bright sources must exceed 1.0 in HDR: `material.emissive = new pc.Color(1,.6,.2); material.emissiveIntensity = 20;`. Inspect stages with `frame.debug = 'ssao' | 'bloom' | 'depth' | 'dofcoc' ...` (set back to `null`).
   When `CameraFrame` is active `StandardMaterial.toneMapping` is ignored; `camera.toneMapping` then only affects what renders after it (UI). Alpha blending happens in linear HDR so translucents look different from LDR; retune opacity.
   Colour LUT textures must be 256x16 horizontal-strip (Unreal layout) loaded with `srgb: true, mipmaps: false, minfilter: 'linear'`.
   Legacy script post effects (`scripts/posteffects/*.js`) still work but are the old LDR path; prefer `CameraFrame`.
4. **Materials with intent.** `StandardMaterial` is metal/gloss PBR (`useMetalness = true`; `metalness`, `gloss` where gloss = 1 - roughness, `clearCoat`, `sheen`, `iridescence`, `refraction` + `thickness` + `useDynamicRefraction` (glass/transmission), `dispersion`, `anisotropy`, `emissiveIntensity`; confirm exact names on a live `new pc.StandardMaterial()`). Textured glTF materials import correctly; only hand-tune hero assets. Vary roughness; perfectly uniform surfaces look CG.
5. **Atmosphere.** `app.scene.fog` (type/color/density, set via `app.scene.fog.type = pc.FOG_EXP2` etc.), CameraFrame volumetric fog/light shafts, soft particles, subtle depth haze, a ground contact (shadow catcher script `scripts/esm/shadow-catcher.mjs`, blurred planar reflections `blurred-planar-reflection.mjs`, procedural sky `sky/procedural-sky.mjs`, water `water.mjs`).
6. **Camera is the director.** Ease everything (critically damped smoothing, `pc.math.lerp` with `1 - Math.exp(-k*dt)`), slow parallax drifts, FOV 35-50 for products, lead-in/out moves, depth of field on hero shots. Never snap.
7. **Polish**: preloader/progress UI while assets stream (`AssetListLoader` + progress events), fade-in from black, DOM HUD/overlays for text (crisper and cheaper than in-engine UI; see official `build-hud`), audio with `SoundManager`, responsive layout, `prefers-reduced-motion`, graceful WebGPU->WebGL2 fallback, input for mouse + touch + gamepad.
8. **Budget**: 60 fps on mid phones. Adaptive quality: lower `maxPixelRatio`, `frame.rendering.renderTargetScale`, shadow resolution, SSAO samples, splat budget when `app.stats`/frame time degrade.

## 8. Gaussian splats (3DGS)

Formats: **SOG** (compressed, recommended; streamed multi-LOD "Streamed SOG" for large scenes), compressed PLY, PLY, SPZ (parser script `scripts/esm/parsers/spz-parser.mjs`), GLB (loads as `container`). Make/clean/convert with SuperSplat (editor.supersplat) and the `splat-transform` CLI (`splat-transform scene.ply scene.sog`, `... scene.glb`).

```js
const splatAsset = new pc.Asset('scene', 'gsplat', { url: 'scene.sog' });   // needs GSplatComponentSystem + GSplatHandler
await new Promise(r => new pc.AssetListLoader([splatAsset], app.assets).load(r));
const splat = new pc.Entity('splat');
splat.setPosition(0, -0.7, 0);
splat.setEulerAngles(0, 0, 180);          // most COLMAP/PLY captures are upside-down; calibrate per capture
splat.addComponent('gsplat', { asset: splatAsset });
app.root.addChild(splat);
```

Rules of thumb (from the official performance guide and typings):
- Fill rate is the bottleneck (overdraw + alpha blending). Create the device with `antialias: false`, cap device pixel ratio, avoid MSAA; use TAA/`renderTargetScale` if you need smoothing.
- Every gaussian is sorted every frame: trim floaters/background in SuperSplat, keep counts sane (~1M mobile, 3M+ desktop target budgets).
- Global budget for streamed scenes: `app.scene.gsplat.splatBudget = 4_000_000` (default 1M); `app.scene.gsplat.splatBudgetMode = pc.GSPLAT_BUDGET_TARGET | GSPLAT_BUDGET_LIMIT`; `app.scene.gsplat.lodMode = pc.GSPLAT_LODMODE_DISTANCE | GSPLAT_LODMODE_ERROR`; per component `lodFalloff`, `lodRangeMin/Max`. Fast first frame: clamp `lodRangeMin/Max` to the coarsest level, then reopen on `app.systems.gsplat.on('frame:ready', ...)`.
- Renderer: `app.scene.gsplat.renderer` = `GSPLAT_RENDERER_AUTO` (GPU sort on WebGPU, CPU sort on WebGL), `_RASTER_CPU_SORT`, `_RASTER_GPU_SORT`, `_COMPUTE`. Other params: `antiAlias`, `alphaClip`, `minPixelSize`, `fisheye`, `twoDimensional` (2DGS), `colorizeLod`/`debug` for LOD debugging.
- Mix with mesh content freely; depth/DoF with splats needs `GSplatParams.sceneDepthWrite` (see typings) and CameraFrame; shadows, picking, relighting, tonemapping/exposure, procedural/text splats and render-on-demand each have a manual page under `/user-manual/gaussian-splatting/building/`.
- Ready-made effect scripts in `scripts/esm/gsplat/`: `streamed-gsplat` (LOD presets), `reveal-radial|rain|grid-eruption`, `shader-effect-dissolve|crop|box`, `gsplat-weather`, `gsplat-text|image|mesh|lines|flipbook|trees|relighting`. A `camera-controls.mjs` orbit/fly rig is the standard viewer camera.
- Viewer UX: orbit with damping, auto-orbit when idle, loading spinner while LOD 0 streams, annotation hotspots (`scripts/esm/annotations.mjs`), cinematic reveal on load.

## 9. Production scripts shipped in the package (import, do not rewrite)

```js
import { CameraControls } from 'playcanvas/scripts/esm/camera-controls.mjs';
entity.addComponent('script'); entity.script.create(CameraControls, { properties: { focusPoint: new pc.Vec3(0, 1, 0) } });
```
In `scripts/esm/`: camera-controls (orbit/fly/pan, touch+gamepad), first-person-controller, third-person-controller, camera-frame (Editor wrapper of CameraFrame), planar-renderer, blurred-planar-reflection, shadow-catcher, grid, water, annotations, sky/procedural-sky, camera/perspective-correction, gsplat/* (above), vat/* (vertex animation textures: crowds), xr/* (session, controllers, navigation/teleport, manipulation, menu), parsers (obj, spz). Classic helpers in `scripts/`: camera (orbit/fly/follow/tracking), physics (vehicle, render-physics), posteffects, animation/tween.js, utils/cubemap-renderer, textmesh.

## 10. Animation, input, audio, UI, XR

- **Animation**: GLB clips + `anim` component with a state graph (`entity.anim.loadStateGraph({...})`, `entity.anim.baseLayer`, `findAnimationLayer`, `assignAnimation`), blend parameters (`setFloat/setBoolean/setTrigger`), cross-fade transitions. For one-off property animation use a tween/easing helper (`scripts/animation/tween.js`) or hand-rolled spring/lerp in `update`. Retarget only after inspecting joint names/hierarchy.
- **Input**: `app.keyboard`, `app.mouse` (pointer lock via `app.mouse.enablePointerLock()`), `app.touch`, `app.gamepads`; newer `extras/input` sources/controllers power camera-controls. Always support touch. Clamp `dt` (e.g. `Math.min(dt, 1/20)`) to survive tab switches.
- **Picking**: physics `raycastFirst` (colliders), `pc.Picker` (GPU id picking of mesh instances), splat picking per manual.
- **Audio**: `sound` component with `SoundManager`; start audio on first user gesture (autoplay policy).
- **UI**: prefer DOM overlays; in-engine Screen/Element components for world-space or XR UI.
- **XR**: `app.xr.start(cameraComponent, pc.XRTYPE_VR | XRTYPE_AR, pc.XRSPACE_LOCALFLOOR)`; needs `xrCompatible` device and HTTPS; WebXR on WebGPU is supported in recent 2.x; use `scripts/esm/xr/*` for session/controllers/teleport/menus. Test on device or the WebXR emulator.

## 11. Performance and robustness

- Measure first: `new pc.MiniStats(app)` (on-screen CPU/GPU/draw-call graphs; exported from the main package), `app.stats`, the browser GPU profiler, `pc.Tracing`. Fix draw calls, then overdraw/fill, then vertex count, then JS allocations.
- Draw calls: static batching/instancing (`BatchManager`, `MeshInstance` instancing, `render` `batchGroupId`), merge materials/atlas textures, frustum/occlusion-friendly layout, LOD via distance switching scripts.
- Shadows are the usual hog: one cascaded directional, small `shadowDistance`, 1024-2048 maps, no point-light shadows unless essential.
- Textures: KTX2/Basis, mipmaps on, anisotropy 4-8, power-of-two for atlases, dispose unused.
- Mobile: DPR cap, no SSAO/DoF by default, lower splat budget, test thermal throttling.
- Device loss and context restore exist (`/graphics/advanced-rendering/device-loss`); do not cache raw GL/GPU objects.
- Loading: stream and show something in <2 s (low LOD splat / low-res env first), lazy-load physics/audio/heavy scripts.
- Ship: Vite build, hashed assets, gzip/brotli, CDN, HTTPS, `<meta name=viewport>`, `touch-action:none` on canvas, `overflow:hidden` body.

## 12. Verification workflow (mandatory before declaring done)

1. `npm run build` (and `tsc --noEmit` for TS) with zero errors. Lint if configured.
2. **Headless logic test (Node, no GPU)** for scripts/state/physics-free logic using `NullGraphicsDevice` (verified pattern from the manual):
   ```js
   import { AppBase, AppOptions, NullGraphicsDevice, ScriptComponentSystem, Entity } from 'playcanvas';
   const canvas = { width: 1, height: 1 };
   const o = new AppOptions(); o.graphicsDevice = new NullGraphicsDevice(canvas); o.componentSystems = [ScriptComponentSystem];
   const app = new AppBase(canvas); app.init(o); app.start();
   // build entities, entity.script.create(MyScript), then: for (let i = 0; i < 60; i++) app.update(1 / 60);
   // assert on entity transforms / state. Asset loading needs jsdom (see manual).
   ```
3. **Real-browser check** (Playwright/Puppeteer or the agent's browser tool): load the dev server, wait for first frame (`app.on('frameend')` or a `window.__ready` flag you set), then
   - capture console: **zero errors/warnings** (use the debug build in dev: deprecation + misuse warnings are only there),
   - screenshot and assert the canvas is not uniformly black/white (sample pixels), compare composition visually (framing, lighting, scale, orientation),
   - read `app.stats` / frame time over ~3 s: target 60 fps desktop, 30+ low-end mobile,
   - rotate/orbit/move input paths once, resize the window, toggle fullscreen.
   Headless Chrome flags that usually enable rendering: `--use-angle=swiftshader --enable-unsafe-swiftshader` (WebGL2 software) or `--enable-unsafe-webgpu --enable-features=Vulkan` (WebGPU where drivers allow). If no GPU path exists, say so and verify what you can; never claim visual correctness you did not see.
4. Re-read the original request and tick every requirement, then list any assumption you made.

Triage table:

| Symptom | Usual cause |
|---|---|
| Black canvas | no light/env, camera inside object or looking +Z, emissive tint black, `app.start()` missing, device creation failed, tone mapping/exposure extreme |
| `addComponent` returns null | component system not in `AppOptions.componentSystems` |
| Asset never loads | handler missing, wrong asset type string, CORS, relative URL base, parse error (check network tab) |
| Physics does nothing | ammo not loaded before app creation, no collision+rigidbody pair, moving a dynamic body by transform |
| Model sideways/backwards/tiny | +Z facing, unit scale (cm vs m), pivot; calibrate |
| Splat upside-down / invisible | missing 180° Z flip, far plane/near clip, wrong scale, `splatBudget` too low, no `gsplat` system/handler |
| Splat slow | MSAA on, DPR 3, huge counts, no LOD/budget |
| Washed out / too dark | gamma/tonemapping mismatch, non-sRGB colour textures, `GAMMA_NONE` outside HDR path |
| Custom GLSL fails on WebGPU | transpile needs `glslangUrl` + `twgslUrl`, or write WGSL; ShaderMaterial must call `gammaCorrectOutput`/`toneMap`/`addFog` itself (since 2.5) |
| Flicker/shimmer | shadow bias/resolution, z-fighting, TAA without sharpness |

## 13. Working method for any request

1. Restate the deliverable in one line; list assumptions (surface, target devices, assets available/needed, WebGPU or WebGL2).
2. Inspect environment (section 0) and find the closest example/script.
3. Make a task checklist; build the smallest end-to-end vertical slice first (scene, camera, light, one asset), verify, then layer systems (look-dev, interaction, physics, UI, audio), verifying each.
4. Assets: use user-provided; otherwise procedural (primitives, shaders, particles, `Mesh` generation) or CC0 sources; never hot-link unlicensed assets. Record licences.
5. Deliver runnable code (README with `npm i && npm run dev`), plus a short note of what was verified and what could not be.
6. If a starter project is available (Vite + `src/app.js` bootstrap, `src/look.js` CameraFrame + lights preset, `tests/` headless tests, `tools/verify-browser.mjs`), start from it instead of a blank folder; otherwise recreate those four pieces from sections 2, 7 and 12.

## 14. v1 / stale-sample pitfalls (do not emit these)

`entity.model` / `ModelComponent` and `instantiateModelEntity` (use `render`); `pc.BasicMaterial` (removed; use StandardMaterial emissive); WebGL1 (gone); legacy pre-2016 scripts; `pc.script.create`; `Scene.gammaCorrection`/`Scene.toneMapping` (now per camera or CameraFrame); `Scene.fogColor/fogEnd` (use `scene.fog.*`); callbacks on `Layer` for rendering (use Scene events); render target on Layer (set on camera); `setGravity()`; `audiosource` (use `sound`); `Application` on WebGPU (use `createGraphicsDevice` + `AppBase`); old `playcanvas-stable.min.js` CDN scripts; `pc.createScript` in new code; assuming `bloom.enabled` exists; copying `ShaderMaterial` shaders written before 2.5 without explicit gamma/tonemap/fog calls; reading global `pc.` in module code (import it).

## 15. Resource map

- Repo: https://github.com/playcanvas/engine (read `AGENTS.md` only when contributing to the engine itself; hierarchy core -> platform -> scene -> framework; JSDoc public APIs; WebGL2+WebGPU parity; GLSL and WGSL chunks both)
- Manual: https://developer.playcanvas.com/user-manual/ (append `.md` per page) · API: https://api.playcanvas.com/engine/ · Examples: https://playcanvas.com/examples · Migrations: /user-manual/engine/migrations/ · AI guide: /user-manual/engine/developing-with-ai/
- Splats: /user-manual/gaussian-splatting/ · SuperSplat https://playcanvas.com/supersplat/editor · SplatTransform /user-manual/splat-transform/ · showcase https://github.com/playcanvas/awesome-playcanvas
- Packages: `playcanvas`, `@playcanvas/react`, `@playcanvas/web-components`, `create-playcanvas`, official agent skills `playcanvas/skills`
- Community: https://forum.playcanvas.com (release posts "Engine v2.x.0" list new features), Discord, r/PlayCanvas
