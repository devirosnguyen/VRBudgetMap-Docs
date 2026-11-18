---
title: Manual
---

# VR Budget Map – Manual

[Home](../) · [Changelog](../changelog/)

Version 1.0 · Unity 2022.3 LTS and Unity 6 · URP 14+

VR Budget Map measures the render cost of your level from many viewpoints – a grid over the walkable area in every
direction, recorded playtest paths and points you place – and compares each view with the budget of a standalone VR
headset. The result is a heatmap on your level, a list of the worst spots, and the likely cause at each one.

> **Budget metrics, not a frame-rate prediction.** VR Budget Map counts what the GPU is asked to do per view (draw calls,
> triangles, shadow casters, transparent overdraw…) and compares it with the ranges the device makers recommend. It does
> not predict frame rate or GPU time. A green level can still be slow (heavy shaders, fill rate); a red spot tells you
> where the counts are above the recommended range.

Requirements: Unity 2022.3 or newer (tested on 2022.3.62f3 and 6000.3.9f1), the Universal Render Pipeline (URP 14 or
newer). In a project without URP the window only shows a notice. Editor only: nothing is added to your builds unless you
make a calibration build (section 8).

## 1. Quick start (2 minutes)

1. Open `Samples/Scenes/VRBudgetMap_SampleLevel`.
2. **Window ▸ Analysis ▸ VR Budget Map**.
3. Keep **Device budget: Quest 2**, press **Sweep scene**. About 5,000 views are measured in under a minute on a desktop GPU (progress bar, Cancel
   keeps what was measured).
4. The Scene view shows the heatmap: green within the budget, yellow above the warning level, red over the limit, purple at
   1.5× the limit and more. The market hall is red (draw calls), the smoke in the yard purple (transparent overdraw).
5. Click the first line of **Worst spots**: the Scene view moves to that eye position and direction, and **Likely causes**
   explains it (here: shadow casters, extra materials, a mesh repeated 120 times).
6. **Report** writes a Markdown report and a CSV of every view.

For your level: open it, check the build target (Android for standalone headsets – see 6.3), pick the device, sweep.

## 2. The window

### Device budget
Choose the preset of the device you ship on. The line under it shows the limits, the convention (per eye / both eyes),
the stereo mode, the eye buffer and the platform whose quality level is measured. **Edit** selects the preset asset (the
first time it offers to save editable copies of the defaults).

### Viewpoints
- **Grid on the walkable area** – a point per cell on the baked NavMesh, or, without a NavMesh, on floor colliders
  (downward rays: surfaces flatter than 40° with head room above). Stacked floors get one point each. **Cell size** (2 m)
  and **Directions** (8) set the density; the grid grows coarser by itself above 2,000 points.
- **Eye height** (1.6 m) and **Head pitch** (5° down).
- **Budget Viewpoint markers** – add the *Budget Viewpoint* component to any GameObject (spawn points, vistas, known
  heavy places). It looks along its forward, or turns around like a grid point with *All Directions*; its *Label* shows in
  the list.
- **Playtest Replay paths** – with Playtest Replay in the project, add `.ptrec` recordings: the head poses of real
  playtesters are measured every 1.5 m, looking where they looked.
- **Player camera** – optional: the measuring camera copies its clear flags, culling mask and URP options (renderer,
  post-processing, shadows, depth/opaque texture).
- **Estimate only (no GPU)** – see 6.2.

### Sweep, Re-measure, Report, Clear
**Sweep scene** reuses a cached result when the scene content, the settings and the viewpoints are unchanged (stored in
`Library/VRBudgetMap/Sweeps`). **Re-measure** ignores the cache. A cancelled sweep continues where it stopped next time.

### Results
The summary line says how the numbers were obtained (*rendered, GPU counters* or *ESTIMATE*), the URP configuration and
the build target. Each **spot** (a standing place) is scored by its **worst direction** – in VR the player can look
anywhere. The heatmap can show all limits or one counter.

Selecting a spot shows its per-eye counters, the whole stereo frame, overdraw, **Go to view**, **Preview** (the measured
view rendered small), **Likely causes** with *Select N* buttons, and **Biggest objects** (copies of one object grouped).

## 3. What is measured

For each view the level is rendered offscreen at the device's eye resolution, with the quality level the build uses
(the platform's default level: its URP asset, LOD bias, shadows), and Unity's own render counters are read around that
render:

| Counter | Meaning |
|---|---|
| Draw calls, batches, SetPass calls | as in the Stats panel / Frame Debugger, all passes: depth prepass, shadow maps, opaque, transparent, skybox, full-screen passes |
| Triangles, vertices | submitted over all passes (a mesh in the shadow map and the opaque pass counts twice) |
| Visible triangles / renderers | each visible renderer counted once (selected LOD, frustum) |
| Shadow casters | renderers drawn into the main light's shadow map, per cascade |
| Transparent overdraw | blended layers shaded per screen pixel, averaged over the view; peak; share of the view with 3+ layers |
| Alpha-tested overdraw | cut-out layers per pixel (they defeat early depth rejection on tile GPUs) |
| Particles | live particles in view (stopped systems are simulated while measuring and cleared afterwards) |

**Both eyes.** Shadow maps are rendered once for both eyes. *Multiview* (single-pass instanced): draw calls and SetPass
stay as one eye, eye triangles double. *Multi-pass*: eye draw calls double too. The preset says whether its limits apply
per eye or to the whole frame; both numbers are always shown.

**Causes** are analytic: each renderer's share of the view (draws, shadow draws, triangles, size on screen) is estimated
and checked against known patterns – pipeline passes (depth prepass from SSAO / depth priming / MSAA, opaque texture
copy), shadow casters, extra materials (more materials than submeshes draw the mesh again), multi-material meshes, a
mesh repeated without instancing, dense meshes without LOD, transparency and alpha test, realtime additional lights,
skinned meshes, uncompressed / non-ASTC / very large textures (with the texture memory of the view in the platform's
formats).

## 4. Budget presets

| Preset | Draw calls warn / max | Triangles warn / max | Source |
|---|---|---|---|
| Quest 2 | 200 / 300 | 750k / 1M | Meta, "Testing and performance analysis" (medium per-draw cost) |
| Quest 3 / 3S | 400 / 600 | 1.3M / 1.8M | same page |
| Pico 4 | 200 / 300 | 750k / 1M | triangles: Pico SDK guide; draw calls are not published by Pico, Quest 2 values are used |
| PC VR | 500 / 1000 | 1M / 2M | Meta, PC VR (Link) range |

Every preset also has *Transparent overdraw* 1 / 2 layers per pixel: no maker publishes a number; it is a starting point
to adjust. Meta does not state whether its numbers are per eye or per frame; the presets use **per eye** (multiview draw
calls are the same either way). Eye buffers and fields of view are typical defaults – set them to your XR settings. Each
limit keeps its source next to it; edit or add limits (batches, SetPass, vertices, shadow casters, skinned meshes,
particles) in the preset asset.

## 5. Accuracy

Measured views were compared with Unity's Frame Debugger on test scenes with known content (identical meshes with and
without the SRP Batcher and GPU instancing, multi-material meshes, LOD groups at known distances, shadows with 1 and 4
cascades, stacked transparent quads), on Unity 2022.3.62f3 / URP 14 and 6000.3.9f1 / URP 17, Windows and Android build
targets:

- **Draw calls: identical to the Frame Debugger in every view** of the scenes rendered to the eye texture. With some URP
  assets the offscreen render adds one fixed full-screen draw that a Game view render does not have.
- Triangles: identical, apart from the same fixed blit (5–6 triangles).
- Transparent overdraw: 4 stacked 3 × 3 m panes at 3–4.5 m – geometric coverage 0.611 layers per pixel, measured 0.613,
  peak 4.
- The analytic estimate (no GPU) matched the draw calls in 45 of 48 views; the others over-counted shadow casters with 4
  cascades (≈ 4 %).
- On-device comparison (section 8) is the way to check your own project on your headset.

## 6. Limitations

### 6.1 Occlusion culling is not applied
Unity's baked occlusion culling is not applied to the offscreen renders VR Budget Map makes (checked in edit and play
mode). Where walls hide objects in the game, the counts are **upper bounds**. The window and the report warn when the
scene has baked occlusion data.

### 6.2 Estimate only (no GPU)
Without a graphics device (`-nographics`, some CI machines) views are estimated: frustum culling, LOD selection,
submeshes × passes, shadow casters per cascade, instancing. No occlusion, no dynamic batching, no particles, no
overdraw. Results are labelled ESTIMATE.

### 6.3 Build target
URP chooses some passes per platform at compile time (for example depth priming in *Auto* mode is off on Android). For
numbers that match a standalone headset build, switch the build target to Android before sweeping; the window warns when
it differs from the preset's platform.

### 6.4 Other
- One eye is rendered with a symmetric frustum; the two eyes together see slightly more at the sides.
- Particle systems are simulated for a few seconds when stopped; systems driven by scripts may look different in play.
- Custom shaders without a `_Cull` property are counted as back-face culled in overdraw.
- Terrain and world-space UI are rendered and counted, but no cause rule targets them yet.

## 7. Report and CSV

**Report** writes `<name>.md`: the disclaimer, scene, Unity version and build target, measurement source, pipeline, eye
and stereo settings, every limit with its source, the spot summary, and for the 20 worst spots their counters (per eye
and both eyes), the likely causes with suggestions, the biggest objects and the texture memory of the view.
`<name>.csv` has one line per view with every counter, the score, the status and the worst metric.

## 8. On-device calibration

To check the editor's numbers on your headset:

1. Build the scene as a **Development Build** for the headset (OpenXR, your usual settings) and install it.
2. In the window, open **Device calibration**, check the *Package name* (defaults to your Player Settings identifier),
   choose how many worst spots and spread views to visit, press **Run on device** (ADB from the Android SDK Unity uses).
3. The app (only in development builds, and only when the route file was pushed) stops head tracking, visits each view,
   waits 30 frames, records the render counters of the whole stereo frame over 30 frames and the frame times, writes a
   result file and quits. The editor waits, pulls the result and writes `Library/VRBudgetMap/Calibration/*.md`: the error
   of draw calls, batches, SetPass and triangles per view, and on average. Device frame times are listed for context only.

Use **Pull results** if the editor was closed while the device ran.

## 9. Scripting

Everything the window does is available from editor code (namespace `JieYingStudio.VRBudgetMap.EditorTools`):

```csharp
var preset = BudgetPresets.All()[0];
var sampling = new SamplingSettings { cellSize = 2f };
var views = ViewpointSampler.Expand(ViewpointSampler.FloorPoints(sampling, out _), sampling);
var sweep = new BudgetSweep(views, preset.ToMeasureSettings());
BudgetSweep.RunWithProgressBar(sweep, s => BudgetReport.Write("Logs/budget", s.Result, preset, sampling.eyeHeight));
```

`GpuViewMeasurer` measures single views (call `WarmUp()` once per editor update until it returns true), `ViewEstimator`
estimates them, `CauseAnalyzer` explains them. All of them switch the quality level and profiler settings temporarily and
restore them when disposed.

## 10. Troubleshooting

- *"Unity's render counters did not start"* – another profiling session may hold the profiler; close the Profiler window's
  recording and try again. The editor must be able to update (not paused in a modal dialog).
- *"Disable the Frame Debugger before measuring"* – the counters stop while the Frame Debugger is enabled.
- *No viewpoints* – the scene needs a baked NavMesh, colliders on the floor layers, Budget Viewpoint markers or recordings.
- *Numbers differ from my build* – check the build target (6.3), the quality level of the platform, and the eye buffer
  and FOV of the preset; then run the on-device calibration.

## 11. Support

devirosnguyen@gmail.com
