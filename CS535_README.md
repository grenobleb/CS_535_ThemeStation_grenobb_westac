# ThemeStation — CS535 Project Fork

> **This repository is a fork of [ThemeStation](https://github.com/3DThemeStation/ThemeStation), made for CS535 Computer Graphics at SUNY Polytechnic Institute.**
> The original work, code and paper belong to the ThemeStation authors (Wang et al., SIGGRAPH 2024). See the original [README.md](README.md) for their setup and usage instructions and for how to cite them.

This file tracks what our team adds to the project. The original README is unchanged.

---

## Team

| Name | Role / Focus |
|------|--------------|
| _TBD_ | _TBD_ |

**Course:** CS535 Computer Graphics, SUNY Polytechnic Institute
**Professor:** Dr.Reale

---

## Project Goal

The original authors left one item unfinished in their to-do list:

- [ ] **Update our code to support more powerful image-to-3D models, such as InstantMesh/CRM, to obtain the initial 3D**

Our project is to finish that item: let ThemeStation use a newer image-to-3D model, either [InstantMesh](https://github.com/TencentARC/InstantMesh) or [CRM](https://github.com/thu-ml/CRM), in place of Wonder3D to build the initial (coarse) 3D model in Stage II, and then compare the results.

---

## How the Initial 3D Is Made Currently (Baseline)

In the original code, Wonder3D produces the initial 3D model. This happens in the first two steps of `run_pipe()` in [run.py](run.py):

1. **Multi-view images:** [run.py:54](run.py#L54) runs `Wonder3D/run_mv.sh` on the concept image. It writes multi-view RGB and normal images to `Wonder3D/outputs/cropsize-192-cfg3.0/`.
2. **Coarse 3D reconstruction:** [run.py:77](run.py#L77) runs `Wonder3D/instant-nsr-pl/run_coarse3d.sh`. This trains an instant-nsr-pl SDF and saves a checkpoint to:
   ```
   Wonder3D/instant-nsr-pl/exp/mesh-ortho-{obj_name}/ckpts/epoch=0-step=5000.ckpt
   ```

Later stages load that checkpoint, not a mesh file:

- `scripts/run_optimization.sh`, `scripts/run_coarse3d_video.sh` and `scripts/run_render_init_images.sh` pass it in as `system.geometry_convert_from`, together with `system.prev_cfg_dir=Wonder3D/instant-nsr-pl/configs/parsed.yaml`.
- [configs/theme-station-optimization.yaml](configs/theme-station-optimization.yaml) is tuned to match Wonder3D's network: `mlp_network_config.otype: VanillaMLPWon3D`, `global_step_offset: 5000` (the number of Wonder3D training steps) and `geometry_convert_inherit_texture: true`.
- The checkpoint path is hard-coded in several places in `run.py` (lines 79, 93, 118 and 211, with more in the resume logic around lines 271–325).

**Why this matters:** InstantMesh and CRM output a textured **mesh** (`.obj`/`.glb`), not an instant-nsr-pl checkpoint. So swapping models is more than changing one script. The way Stage II receives its initial geometry (and texture) has to change too.

---

## Planned Approach

_Draft. We will update this section as decisions are made._

1. **Pick the model:** InstantMesh or CRM (or both), based on GPU memory, install effort and output quality on our test concept images.
2. **Add a coarse-3D step that uses the new model:** run it on the background-removed concept image in `data/concept_images/` and save the output mesh. Keep Wonder3D as the default and add a `run.py` flag to choose the backend.
3. **Load the mesh into Stage II.** Options we are looking at:
   - **Option A: mesh → SDF initialization.** `ImplicitSDF` can already initialize from a mesh with `shape_init: "mesh:<path>"` ([implicit_sdf.py:163](threestudio/models/geometry/implicit_sdf.py#L163)). However, `ThemeStation.on_fit_start()` currently raises `"not implemented error"` when `shape_init` is set ([themestation.py:98-102](threestudio/systems/themestation.py#L98-L102)), so that guard needs to be addressed. Texture would not be inherited, so color would need to be learned or baked separately.
   - **Option B: fit a Wonder3D-compatible checkpoint from the mesh.** Train a short instant-nsr-pl / threestudio stage on renders of the InstantMesh/CRM mesh so the rest of the pipeline stays unchanged.
4. **Update the rendering scripts** that read the coarse 3D (`run_coarse3d_video.sh`, `run_render_init_images.sh`) so the DreamBooth concept-prior step still gets its random views.
5. **Evaluate:** run the same concept images and reference models through the Wonder3D baseline and the new backend, then compare initial-3D quality, final-3D quality and runtime.

---

## Progress Tracker

### Milestones

- [ ] Environment set up and the original pipeline runs end to end (Wonder3D baseline)
- [ ] Baseline results generated for our test set
- [ ] Image-to-3D model chosen (InstantMesh / CRM)
- [ ] New model installed and run on our concept images on its own
- [ ] New model added as a coarse-3D backend in `run.py`
- [ ] Stage II accepts the new initial 3D (Option A or B above)
- [ ] Rendering / DreamBooth steps work with the new initial 3D
- [ ] Full pipeline runs end to end with the new backend
- [ ] Comparison against the Wonder3D baseline (qualitative + runtime)
- [ ] Final report / presentation


## Notes & Known Issues

- **Possible path mismatch in the original code:** the "skip if coarse 3D exists" check at [run.py:79](run.py#L79) builds the path without `{suffix}`, but the paths at lines 93, 118 and 211 include it. For concept images named `*_clipdrop-background-removal.png`, the check may never find the existing checkpoint. We may fix this while reworking the step.
- Hardware / environment notes: _TBD (GPU, CUDA version, OS)_

---

## Credits

- **ThemeStation:** Zhenwei Wang, Tengfei Wang, Gerhard Hancke, Ziwei Liu, Rynson W.H. Lau. [Project page](https://3dthemestation.github.io/) | [Paper](https://arxiv.org/abs/2403.15383)
- **InstantMesh:** [TencentARC/InstantMesh](https://github.com/TencentARC/InstantMesh)
- **CRM:** [thu-ml/CRM](https://github.com/thu-ml/CRM)
- **Wonder3D**, **threestudio**, **Diffusers DreamBooth**, **Objaverse rendering:** see the original [README.md](README.md#acknowledgments)
