# DeepAAT 

### 1. Overview of the Pipeline

The pipeline processes clusters of images (scenes), each containing multiple cameras and feature points. It operates in three main stages:
- **Data Preparation**: The `scene_data` object organizes all data for a scene.
- **Inference**: `inference.py` runs the model to predict outlier masks and camera poses.
- **Post-Processing and Evaluation**: `evaluation.py` refines predictions and computes errors.

Key files and functions:
- **`scene_data` object**: A container for scene-specific data.
- **`inference.py`**: Drives the inference loop and model execution.
- **`evaluation.py`**: Handles post-processing and error metrics.
- **`euc_ba`**: Refines camera poses and 3D points via Bundle Adjustment.
- **`dlt_triangulation`**: Computes 3D points from 2D observations.

---

### 2. The `scene_data` Object: Data Foundation

The `scene_data` object is a structured container holding all data for a single scene, such as `ortho1_10_129` (129 cameras, 193,488 unique 3D feature points). It’s created in `inference.py` and passed to the model.

#### Key Attributes
| **Attribute**         | **Shape (Example)**    | **Description**                                      | **Paper Concept**         |
|-----------------------|-----------------------|-----------------------------------------------------|---------------------------|
| `scan_name`           | `'ortho1_10_129'`     | Scene identifier                                   | Single scene/cluster      |
| `M`                   | `[258, 193488]`       | 2D feature observations (258 = 129 cameras × 2 for u,v) | Measurement matrix W_mes  |
| `Ns`, `Ks`            | `[129, 3, 3]`         | Camera intrinsic matrices and inverses             | Intrinsic matrix K        |
| `mask`                | `[129, 193488]`       | Ground truth outlier mask (1 = inlier, 0 = outlier)| Outlier mask             |
| `gpss`                | `[129, 3]`            | Noisy GPS data for each camera                    | Geotag matrix W_gps       |
| `Rs`, `ts`, `quats`   | `ts: [129, 3]`, `quats: [129, 4]` | Ground truth poses (rotation, translation) | Camera poses             |
| `color`, `scale`      | Varies                | Visualization and normalization data              | Auxiliary data            |
| `x`                   | Sparse Tensor         | Sparse version of `M` (network input)             | Sparse measurement matrix |
| `mask_sparse`         | Sparse Tensor         | Sparse version of `mask`                          | Sparse outlier mask       |
| `egps_sparse`         | Sparse Tensor         | Sparse GPS data aligned with `x`                  | Sparse geotag matrix      |
| `dsc`                 | Sparse Tensor         | SIFT feature descriptors                          | Descriptor matrix W_des   |
| `dsc.cam_per_pts`     | `[193488, 1]`         | Cameras observing each 3D point                  | Helper attribute          |
| `dsc.pts_per_cam`     | `[129, 1]`            | 3D points visible per camera                     | Helper attribute          |
| `valid_pts`, `norm_M` | Varies                | Pre-processed data for loss/normalization         | Pre-processing            |

#### Debugging Tips
- **Shape Check**: Confirm `M` has 258 rows (129 × 2) and 193,488 columns for `ortho1_10_129`.
- **Sparsity**: Verify `x`, `mask_sparse`, and `egps_sparse` align in non-zero indices.
- **Ground Truth**: Use `mask`, `Rs`, `ts`, and `quats` to validate predictions.

---

### 3. Code Flow in `inference.py`: Model Execution

`inference.py` orchestrates the pipeline by iterating over scenes and running the model.

#### Flow
1. **Data Loading**:
   - A `data_loader` (e.g., `myDataSet`) yields `scene_data` objects for each scene in `scans_list`.
   - Example: `scene_data` for `ortho1_10_129`.

2. **Model Inference**:
   ```python
   pred_mask, pred_cam = model(scene_data)
   ```
   - **Inputs**: `scene_data.x`, `scene_data.egps_sparse`, `scene_data.dsc`.
   - **Outputs**:
     - `pred_mask`: Predicted outlier mask (aligns with `scene_data.x`).
     - `pred_cam`: Predicted camera poses (`quats`: `[129, 4]`, `ts`: `[129, 3]`).

3. **Evaluation**:
   ```python
   errors = evaluation.compute_errors(outputs, conf, bundle_adjustment, refined=True)
   ```
   - Passes predictions to `evaluation.py` for post-processing and error computation.

#### Key Variables
- **`pred_mask`**: Binary mask identifying inliers (e.g., 131,290 points kept from 193,488).
- **`pred_cam`**: Pose predictions (quaternions for rotation, translation offsets added to `gpss`).

#### Debugging Tips
- **Input Validation**: Ensure `scene_data.x` and `dsc` have matching sparsity patterns.
- **Output Shapes**: Check `pred_mask` matches `scene_data.x` size, `pred_cam` has 129 entries.

---

### 4. Post-Processing and Evaluation in `evaluation.py`

`evaluation.py` refines model predictions and computes performance metrics.

#### Main Tasks
1. **Post-Processing** (`prepare_predictions_2`):
   - Filters outliers, triangulates 3D points, and runs BA.
2. **Error Calculation** (`compute_errors`):
   - Compares predictions to ground truth.

#### Key Variables in `outputs` Dictionary
| **Variable**         | **Shape (Example)**    | **Description**                          | **Paper Concept**         |
|---------------------|-----------------------|-----------------------------------------|---------------------------|
| `prediction`         | `(893353,)`           | Raw outlier scores                     | Outlier score             |
| `premask`, `gtmask`  | `(893353,)`           | Predicted vs. ground truth masks       | Outlier masks             |
| `raw_color`, `raw_xs`| `(3, 193488)`, `(129, 193488, 2)` | Initial points pre-filtering | All initial points        |
| `precolor`, `xs`     | `(3, 131290)`, `(129, 131290, 2)` | Filtered points         | Filtered points           |
| `Rs_gt`, `ts_gt`     | `(129, 3, 3)`, `(129, 3)` | Ground truth poses              | Ground truth poses        |
| `Rs`, `ts`           | `(129, 3, 3)`, `(129, 3)` | Predicted poses pre-BA          | Predicted poses           |
| `Ps`                 | `(129, 3, 4)`         | Predicted projection matrices          | Projection matrices       |
| `pts3D_pred`         | `(4, 131290)`         | Triangulated 3D points pre-BA          | Initial 3D points         |
| `Rs_ba`, `ts_ba`     | `(129, 3, 3)`, `(129, 3)` | Poses post-BA                  | Refined poses             |
| `Xs_ba`              | `(4, 160222)`         | Refined 3D points post-BA              | Refined 3D points         |
| `valid_points`       | `(129, 160222)`       | Visibility mask for final points       | Visibility mask           |
| `Rs_ba_fixed`, `ts_ba_fixed`, `Xs_ba_fixed` | Same as above | Aligned final results | Final aligned results     |

#### Debugging Tips
- **Point Count Changes**: Track reduction from 193,488 (`raw_xs`) to 131,290 (`xs`) to 160,222 (`Xs_ba`).
- **Pose Comparison**: Compare `Rs`/`ts` (pre-BA) to `Rs_ba`/`ts_ba` (post-BA) against `Rs_gt`/`ts_gt`.

---

### 5. Bundle Adjustment in `euc_ba`

The `euc_ba` function (in `ba_functions`) refines poses and points to minimize reprojection error. With `refined=True`, it uses a two-stage process.

#### Inputs
- `xs`: Filtered 2D points (`[129, 131290, 2]`).
- `raw_xs`: All original 2D points (`[129, 193488, 2]`).
- `Rs`, `ts`: Predicted poses (`[129, 3, 3]`, `[129, 3]`).
- `Ks`: Intrinsics (`[129, 3, 3]`).
- `Xs`: Initial 3D points (`[131290, 4]`).

#### Flow
1. **Pose Refinement**:
   - Uses `xs` and `Xs` to optimize `Rs` and `ts`.
   - Output: `new_Ps`, refined projection matrices.

2. **Point Cloud Recovery**:
   - **Re-Triangulation**: Uses `new_Ps` and `raw_xs` to recover points (via `dlt_triangulation`).
   - **Final BA**: Refines all poses and points, filters outliers.

#### Outputs
- `Rs_ba`, `ts_ba`: Refined poses.
- `Xs_ba`: Refined 3D points (`[160222, 4]`).
- `valid_points`: Visibility mask.

#### Debugging Tips
- **Stage Transition**: Verify `new_Ps` improves reprojection error before re-triangulation.
- **Point Recovery**: Check how `raw_xs` increases point count, then final BA adjusts it.

---

### 6. Triangulation in `dlt_triangulation`

The `dlt_triangulation` function (in `geo_utils`) computes 3D points using the Direct Linear Transformation (DLT) algorithm.

#### Inputs
- `Ps`: Refined projection matrices (`[129, 3, 4]`).
- `xs`: Original 2D points (`raw_xs`).
- `visible_points`: Visibility mask.

#### Flow
1. **Camera Check**: Ensures each point is seen by ≥2 cameras.
2. **Linear System**: Builds matrix `A` from projection equations.
3. **SVD Solution**: Solves for 3D coordinates using singular value decomposition.
4. **Optimization**: Uses Dask for points seen by >40 cameras.
5. **Normalization**: Converts homogeneous coordinates to 3D.

#### Role
- Recovers points discarded by the network, explaining the jump from 131,290 to ~193,488, then refined to 160,222 after final BA.

#### Debugging Tips
- **Visibility**: Confirm `visible_points` aligns with `raw_xs`.
- **Output Size**: Trace point count changes post-triangulation.

---

### 7. Results Table: Performance Summary

The results table evaluates the pipeline across scenes.

#### Key Metrics
| **Category**          | **Metrics**                     | **Description**                          |
|----------------------|--------------------------------|-----------------------------------------|
| **Outlier Rejection**| TP, FP, TN, FN, Accuracy, Precision, Recall, F1 | Classification performance |
| **Point Counts**     | `all_pts`, `pred_pts`, `gt_pts` | Effect of filtering             |
| **Pose Quality (Pre-BA)** | `our_repro`, `ts_meanmedmax`, `Rs_meanmedmax` | Initial pose accuracy |
| **Pose Quality (Post-BA)** | `repro_ba`, `ts_ba_meanmedmax`, `Rs_ba_meanmedmax` | Final pose accuracy |
| **Efficiency**       | Inference time                 | Model runtime                   |

#### Example Interpretation (`ortho1_10_129`)
- **F1**: 0.983 (98.3% outlier rejection accuracy).
- **Points**: 193,488 → 131,290 (predicted) vs. 132,782 (ground truth).
- **Pre-BA**: 44.77px reprojection error, 4.144m translation error.
- **Post-BA**: 0.47px reprojection error, 4.791m translation error.

---

### 8. Debugging Tips

- **Shapes**: Verify tensor dimensions match expected values (e.g., 129 cameras).
- **Data Flow**: Trace from `scene_data` → `pred_mask`/`pred_cam` → `outputs`.
- **Point Counts**: Monitor changes at each stage (filtering, triangulation, BA).
- **Error Analysis**: Compare predicted vs. ground truth values.
- **Profiling**: Measure inference time for efficiency bottlenecks.

This guide equips you to navigate the DeepAAT codebase, understand its variables and functions, and follow the data flow, bridging the gap between the paper’s concepts and their implementation.