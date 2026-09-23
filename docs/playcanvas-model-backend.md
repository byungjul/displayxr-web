# Plan: a PlayCanvas backend for `addModel` (`engine: 'playcanvas'`)

Status: **plan, nothing built.** Epic [#36](https://github.com/DisplayXR/displayxr-web/issues/36).
`addModel` renders with three.js only. `addSplat` has had a PlayCanvas backend since 1.8.0
([playcanvas-adapter.md](playcanvas-adapter.md)). This note says what
`addModel(wall, canvas, src, { engine: 'playcanvas' })` needs, what already exists, and what does
not.

## Why

- **One engine per tile.** A catalogue that mixes captured splats and vendor meshes wants both in
  one scene. A PlayCanvas splat tile can already hold a glTF: from 1.9.1, `handle.engine`
  registers the `Container` handler and the `Render`, `Light` and `Anim` systems. The model
  sample's mixed tile does it (`samples/model/`, `?splat=<.sog>`). What is missing is the
  *mesh-first* call: a tile whose subject is the mesh, framed, lit and handled the way
  `addModel` does it on three.
- **A page that uses both calls loads both engines.** Today a product page with a splat tile and a
  model tile pays for PlayCanvas and three.js.
- **Not** for speed. A single glTF is cheap on either engine. The splat is where PlayCanvas pays
  (≥3.5× in stereo).

## What already exists and is reused as is

The PlayCanvas splat adapter (`js/inline3d-splat-playcanvas.js`) is mostly subject-agnostic:

- One `AppBase` per tile on our own WebGL2 device, with no `XrManager` and no input.
- One camera, with `camera.xrViews` holding one `RenderView` per runtime view. That is the eye
  math, both rigs and the mono fallback.
- The camera's parent carries the inverse pivot. The viewer (`PlayCanvasSplatViewer`) owns fit,
  margin, `fitSweep`, `depthLimit`, zoom and pitch limits, orbit (tilt-and-relax), `idleSpin`,
  `setFocus` / `setPose` / `resetPose`, `feather` and `renderScale`, all with SceneViewer's
  shared constants.
- The handle skeleton: `ready`, `remove`, `exclude` / `unexclude`, `setPose`, `resetPose`,
  `frame`, `engine`, and the queued calls made before the module loaded.
- From 1.9.1, the systems and handlers a glTF needs, skinned and animated included.

## Work items

| # | item | notes | estimate |
|---|---|---|---|
| 1 | **Factor the tile host out of the splat adapter** | `createPlayCanvasTile(wall, canvas, opts)` → `{ app, root, camera, viewer, handle }`. `addSplat` and `addModel` both build on it. The splat-only parts (gsplat asset, footprint chunk fix, cloud pass, rig waterfall, streamed SOG, `setSource`) stay in the splat module. The viewer class gets a subject-neutral name; the splat one stays as an alias. Pinned by the existing parity and trace tests, which must not move. | 1–1.5 d |
| 2 | **Loader** | `pc.Asset(…, 'container', { url })`, then `resource.instantiateRenderEntity()` under `root`. The three path pre-reads the glTF JSON for `extensionsUsed` and wires only the decoders an asset declares. Keep that, and keep the error model: `ready` rejects with an Error naming the extension, the option and the path, and sets `err.gltfExtension`. | 0.5 d |
| 3 | **Draco** (`KHR_draco_mesh_compression`) | Built into the engine: `WasmModule.setConfig('DracoDecoderModule', { glueUrl, wasmUrl, fallbackUrl })` (or `dracoInitialize`). Same serve-it-yourself policy as three: `decoderPath.draco`, no CDN fallback. **To verify:** the engine's draco files and three's `libs/draco/` are both Google's decoder builds; check that one served folder works for both engines before documenting a single path. | 0.5 d |
| 4 | **KTX2 / Basis** (`KHR_texture_basisu`) | Built in: `basisInitialize({ glueUrl, wasmUrl, fallbackUrl })`. The engine's transcoder files are its own, not three's `libs/basis/`, so `decoderPath.basis` has to name them per engine. Keep three's load-time check: a mis-served transcoder must reject `ready`, not resolve a model with no textures. | 0.5 d |
| 5 | **Meshopt** (`EXT_meshopt_compression`) | **Not in engine 2.22.3** (the build has no meshopt reader). Add it through the container's `options.bufferView.processAsync` hook: decode each compressed bufferView with meshoptimizer's `MeshoptDecoder` (the same decoder three uses, from the `meshoptimizer` package, so there is no three dependency). Nothing to serve (the decoder is JS with inline wasm). This is the one piece of new decoding code. | 0.5–1 d |
| 6 | **Framing** | `handle.frame` from the render mesh instances' AABBs, in content space, same shape as three's (`center`, `extent`). Static meshes are exact. **Skinned meshes need a check:** three's `Box3` reads bind-pose vertices, while the engine derives a skinned AABB from bones. Gate: the Fox's box must match three's 25.19 × 79.03 × 154.72 within 1 %, or framing moves. | 0.5 d |
| 7 | **Lighting: `environment: 'room'`** (the default) | The three default bakes IBL from `RoomEnvironment` through PMREM, so metal and glass are not black. PlayCanvas needs an `envAtlas`. Two routes: (a) bake RoomEnvironment once, offline, to an equirect and ship it (~100–300 KB) → `EnvLighting.generateAtlas` at load; (b) build the room procedurally in the tile's app and render a cubemap at runtime. (a) is simpler and deterministic. It needs an asset in the package, so the size has to be checked against the SDK's footprint budget. **The biggest visual-parity risk.** | 1–2 d |
| 8 | **Lighting: `'studio'`, `'none'`, `envMap`** | `studio` maps three's three-point rig onto directional lights. Engine lights shine along local −Y, so they are aimed by rotation, never `lookAt` (the 1.9.1 gotcha). `envMap` is typed as a three PMREM texture today. On this engine it would be a `pc.Texture` atlas, so the option's meaning becomes per-engine; document it, don't overload silently. | 0.5 d |
| 9 | **Colour** | three renders these tiles untonemapped with sRGB output. The splat adapter sets `TONEMAP_NONE` on the camera; meshes also need `GAMMA_SRGB` and material parity (`StandardMaterial` vs `MeshStandardMaterial`: specular model, light units). Gate: grey MAE against the three render of the same asset at the same pose. Target ≤ 3/255 on the Fox and the Duck, and a metal asset with `room` to pin item 7. | 0.5–1 d |
| 10 | **Handle, types, docs** | `ModelHandle` unchanged: `model` becomes the root `pc.Entity`, `viewer` the PlayCanvas viewer, and `engine`/`backend` are added as on splats. `model.d.ts`, CHANGELOG, `playcanvas-adapter.md` §Divergences. No animation autoplay (three's `addModel` plays none); a page animates through `handle.engine` (the `Anim` system is registered). | 1 d |
| 11 | **Gates** | Headless Chrome on the real GPU, as in P1: `samples/model/` and `samples/shopify/` with `?engine=playcanvas`, 0 console errors, MAE vs three per asset (plain, Draco, KTX2, meshopt, skinned), `npm test`. A page that never passes `engine` still makes 0 `playcanvas` requests. | 1 d |

**Total: about 7–10 days**, with item 7 (the room IBL) as the risk that decides the upper end.

## Out of scope for the first cut

- A model `setSource` crossfade (the splat one exists; nobody has asked for it on meshes).
- Picking on meshes (three's `addModel` has none either).
- Making PlayCanvas the default for `addModel`. That follows the splat default flip, which is a
  separate decision (see the epic).

## Order

1 → 2 → 3/4/5 (independent) → 6 → 7/8/9 (the visual gates) → 10 → 11. Item 1 is also what the
page-driven camera mode wants: the F1000 port needs a tile host whose camera the page moves. So
item 1 should land first and on its own.
