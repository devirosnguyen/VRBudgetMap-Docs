---
title: Changelog
---

# Changelog

[Home](../) · [Manual](../manual/)

## 1.0.0
First release.
- Measurement: views rendered offscreen at the device eye resolution with the build's quality level; Unity's render
  counters (draw calls, batches, SetPass, triangles, vertices, shadow casters, skinned meshes) read around each render;
  transparent / alpha-tested overdraw and particles counted without changing your renderer; analytic estimate without a GPU.
- Viewpoints: grid on the NavMesh or floor colliders × directions, Budget Viewpoint markers, Playtest Replay paths.
- Sweeps: incremental with a progress bar, cancel and resume, cached per scene content.
- Budgets: editable presets for Quest 2, Quest 3 / 3S, Pico 4 and PC VR with the source of every number; per-eye or
  both-eyes convention; multiview / multi-pass stereo model.
- Heatmap in the Scene view, worst spots, go to view, preview, likely causes and biggest objects, Markdown / CSV report.
- On-device calibration: route pushed to a development build over ADB, editor vs device error table.
- Sample level, manual, EditMode tests.
