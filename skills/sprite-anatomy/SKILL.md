---
name: sprite-anatomy
description: Build cuttable voxel (sprite-stacked) models with labeled interiors — volume-first modeling, baked-normal pixel rendering, and free-plane cutting. Use when the user wants a pixel/voxel model that can be sliced open to reveal organs or inner structure, or an interactive "cut it yourself" toy.
---

# Sprite Anatomy

Build voxel models that can be **cut open**, not just spun. The defining rule: **define the 3D volume first, with the insides in it.** Four-view carving (intersecting top/side/front/back images) cannot do this — holes and overhangs fill solid — so models here are generated as voxel grids in code.

## Modeling (volume first)

1. **Research the real subject before building.** Look at photos/diagrams of the actual thing — front cross-sections, fin/blade/part counts, surface markings. Guessed shapes read as wrong. (A fish is round and thick from the front, not flat; a salmonid has an adipose fin; a turbine's compressor blades get shorter toward the rear.)
2. **Generate the volume with primitives**: ellipsoids / superellipsoids for bodies, extrusions for fins and blades, boolean subtraction for cavities (cargo decks, intake ducts). Keep the long axis around 200 voxels for detailed models, ~30 for game-like miniatures.
3. **Every voxel carries a part label** (skin, flesh, bone, kidney, roe, glass, deck, blade…) plus its base color. Interiors are modeled, not improvised at cut time: when a cut exposes a voxel, its label decides what the cross-section shows.
4. **Surface detail comes from the reference**: spots, lateral lines, window rows, grooves, panel seams — place them as labeled surface voxels, not as a texture painted on afterwards.

## Rendering (Moon stack style)

Reference implementation: Moon stack by @rehan (patch.gg, 2026-11-19). Starting point for both: Kimo's sprite-stacking post.

1. **Layers = slices.** Each z-layer of the grid is one flat top-down image. Draw slices rotated to the model's angle, lifted 1px per layer; depth resolves with a height/z buffer.
2. **Art-resolution frame buffer.** Render material / shade / depth into a low-res buffer (e.g. 240×135) and upscale with `image-rendering: pixelated`. Crisp pixels, fast frames.
3. **Baked normals + fixed sun.** Estimate per-voxel normals from occupancy at build time; shade with Lambert against a sun that never moves. The model rotates, the light doesn't — that stability is what makes it read as 3D. Newly exposed cut faces get normals from the same occupancy rule.
4. **Ground shadow** in a separate pass, projected along the sun direction.
5. **Outlines once, at the end**: darken pixels at depth discontinuities on the finished frame. Never outline per slice.
6. **Per-slice animation** (offset/visibility per layer) gives assembly and exploded-view effects for free.

## Cutting

- A swipe/drag defines a cutting plane in screen space; transform it into model space and split voxels by signed distance.
- Pieces slide apart along the plane normal so all slices are visible at once.
- Cuts compose: cross-cut rounds, longitudinal fillets, oblique cuts — because the volume is real, any plane works and the cross-section is always anatomically consistent (CT-scan logic).
- Cap the piece count (e.g. 24) for performance.

## Hard-won lessons

- **The speckle bug**: never sort or reuse the render point array in place. If position/color/part pairing drifts between frames, interior colors speckle across the surface and flicker. Keep position, depth, and part label bound together per voxel; use a per-screen-cell z-buffer; verify by rotating an *uncut* model and confirming zero interior-colored pixels on the surface.
- **Resolution restraint**: more dots is not always better. A ~76-dot-wide model was rejected by users as "grainy/unpleasant" in a precision-cutting game; ~30-dot models read better there. Detailed showcase models can go to ~200. Ask what the model is for.
- **Performance**: bake everything heavy (normals, AO) at build time; keep per-frame work to a few ms. Surface-voxel-only rendering with quantized depth buckets is a good default.
- **The user's device is the ground truth for visuals.** Verify shape and color ownership with headless screenshots, then let the user judge on their own screen before publishing.

## Workflow

1. Research the subject (photos, diagrams) — look before building.
2. Write the volume generator; check silhouette and cross-sections against the reference.
3. Add interiors and surface detail as labeled voxels.
4. Render with the frame-buffer pipeline; run the speckle audit (uncut rotation, count interior colors on surface = 0).
5. Screenshot every model (initial, rotated, cut) and show the user before deploying.
