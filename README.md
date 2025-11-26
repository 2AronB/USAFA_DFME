# USAFA_DFME

## Automated CAD Grading via Geometry Overlap

This document outlines a lightweight approach to grading in-class CAD quizzes by comparing student STEP submissions to a reference model using geometric overlap rather than units or orientation assumptions.

## Web app prototype: local geometry overlap grader

`index.html` is a static, client-side tool (currently **v0.2.2**, shown in the page header) that lets you load an instructor reference mesh and a student mesh (STL or OBJ), auto-aligns/normalizes scale, computes a symmetric distance score, and highlights high-error regions in red on the overlapped view.

- **Privacy-friendly:** Files never leave the browser; everything runs locally with WebGL and WASM libraries loaded from CDNs or an optional `/libs` folder.
- **Network hint:** The page now cycles through multiple CDNs (jsDelivr, unpkg, cdnjs + Skypack) and will also try local files under `/libs`. If the button stays disabled, check the console/status panel for which sources failed.
- **Supported formats:** STL and OBJ for now. Convert STEP to STL/OBJ using your CAD tool or `freecad-cli`/`occ` before loading.
- **Output:** A percent difference (lower is better), average and max symmetric distance, and a 3D overlay showing mismatched points in red.

### How to run locally
1. Clone or download the repo.
2. Open `index.html` directly in a modern browser (Chrome/Edge/Firefox). No build step is required.
3. Drop/upload your reference and student files (STL/OBJ). The **Compare geometry** button enables once both are present. A tiny sample mesh (`ARD06004-25 Camera Cover.stl`) is included for smoke testing.
4. Click **Compare geometry** to compute the symmetric distance score and visualize mismatches.

Offline / air-gapped use:
- Place the following files in a local `libs/` folder next to `index.html` (mirroring the Three.js folder structure):
  - `three.module.js`
  - `examples/jsm/controls/OrbitControls.js`
  - `examples/jsm/loaders/STLLoader.js`
  - `examples/jsm/loaders/OBJLoader.js`
  - `examples/jsm/utils/BufferGeometryUtils.js`
  - `three-mesh-bvh.module.js` (from `three-mesh-bvh`)
- The page will try the local copies first, then CDNs. If a source fails, the status box lists each attempted source.

Troubleshooting:
- If the button does not enable, ensure JavaScript is allowed and both file inputs show a selected file (some browsers only fire `change` when a new file is picked).
- Use an up-to-date browser with WebGL enabled; module/CDN loads can fail if offline.

### How to host on GitHub Pages (free)
1. Commit `index.html` (and any assets) to the default branch (e.g., `main`).
2. In GitHub, go to **Settings → Pages**, choose **Source: Deploy from a branch**, and pick the branch/root folder. GitHub will serve the site at `https://<username>.github.io/<repo>/`.
3. If you prefer isolation, place the web assets in a `docs/` folder and point GitHub Pages to `/docs`.

### Where to store data files
- **Small meshes (a few MB):** Check them into the repo (or `docs/`) so they’re served with the page.
- **Larger assets:** Attach them to a GitHub Release or use Git LFS (free up to quota). Reference them via HTTPS URLs in your app.
- **Privacy-sensitive files:** Keep them out of the repo and let instructors drop them in at runtime (the current tool works fully client-side).

### Key Principles
- Normalize geometry automatically: align and scale by best-fit ICP/PCA against the reference so unit/orientation differences are not penalized.
- Evaluate shape similarity: use signed-distance or voxelized overlap to quantify how much of the student part matches the reference.
- Localize mismatches: identify where material is missing or extra to give actionable feedback (e.g., missing/ misplaced holes, overbuilt bosses).
- Keep it fast: target <30–60 seconds total on typical laptops by using coarse-but-robust methods.

### Grading Features
1. **File intake and safety rails**: Accept STEP parts ≤5–10 MB; reject or downsample larger files. Parse client-side when possible to preserve privacy.
2. **Mesh generation**: Tessellate both reference and submission to a modest triangle budget (e.g., 50k–100k) to speed downstream comparisons.
3. **Auto-alignment without penalties**:
   - Infer scale from bounding boxes or ICP; apply a global 25.4× check to correct unit mismatches.
   - Align via PCA for initial pose, then refine with Iterative Closest Point (ICP) on meshes/point clouds.
4. **Overlap scoring**:
   - Compute voxelized occupancy or use a signed-distance field (SDF) on the reference.
   - Sample points from the student mesh; measure inside/outside distances to the reference SDF to detect extra or missing material.
   - Calculate symmetric Chamfer or Hausdorff distances for a single similarity score (0–100%).
5. **Feedback extraction**:
   - Flag clusters of positive distance (extra material) and negative distance (missing material) to pinpoint errors like absent holes or misplaced cuts.
   - Summarize with traffic-light feedback and short remediation tips tied to the largest mismatch regions.
6. **Tolerance and robustness**:
   - Use generous spatial tolerances (e.g., ±0.5–1.0 mm) to avoid penalizing small modeling noise.
   - Provide a low-resolution fallback (coarser voxels) if performance limits are hit.

### Build Steps
1. Choose a client-side STEP-to-mesh pipeline (WASM OpenCascade or xeokit) with Web Worker support.
2. Implement mesh simplification and unit/pose normalization (PCA + ICP) against the reference mesh.
3. Generate an SDF or voxel grid for the reference model; cache it for reuse across submissions.
4. Sample points from the student mesh and compute overlap metrics (inside/outside counts, Chamfer distance).
5. Map mismatch regions to rubric items (e.g., missing through-hole, oversized boss) and generate visual highlights in a WebGL viewer.
6. Package a precheck tool so students can run the same overlap analysis before submitting to get instant guidance.
