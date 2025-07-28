---

### **Textual Flowchart: The DeepAAT Pipeline**

**Objective:** To perform fast, accurate, and robust Automated Aerial Triangulation (AAT) on a large set of UAV images.

---

#### **Phase 0: Offline Preparation & Data Loading**

*   **Description**: This phase covers everything that happens *before* the main inference script is run. It prepares the raw data into manageable, structured clusters.
*   **Theoretical Concept**: *Hierarchical SfM / Divide and Conquer Strategy*.

**`[START]`**

1.  **Data Acquisition & Initial Matching**
    *   **Action**: A UAV captures thousands of images over a survey area. Each image is geotagged with noisy GPS data. A standard SIFT algorithm performs initial feature matching between nearby images.
    *   **Input**: Raw UAV Images, GPS logs.
    *   **Output**: A set of images with pairwise feature correspondences.

2.  **Image Clustering**
    *   **Action**: A "scene graph" is constructed where images are nodes and shared features are edges. The graph is partitioned into smaller, strongly-connected clusters using a `normalized cut` algorithm.
    *   **Input**: All images and their feature correspondences.
    *   **Output**: Disjoint sets (clusters) of images. **Example**: The scene `ortho1_10_129` is one such cluster containing 129 images.

3.  **Data Loading & Object Creation (in `inference.py`)**
    *   **Code**: `data_loader = myDataSet(...)` -> `loaddata(...)` -> `SceneData(...)`
    *   **Action**: The main script iterates through the clusters. For each cluster, it loads all associated data and encapsulates it into a single `SceneData` object.
    *   **Input**: A scene name (e.g., `'ortho1_10_129'`).
    *   **Output**: A comprehensive `scene_data` object containing:
        *   **Raw Data**: `M` (raw 2D points), `raw_xs`, `gpss`, `color`.
        *   **Ground Truth**: `Rs_gt`, `ts_gt`, `gtmask` (for evaluation later).
        *   **Critical Network Inputs**:
            *   `scene_data.x`: Sparse tensor of 2D point observations.
            *   `scene_data.dsc`: Sparse tensor of SIFT feature descriptors.
            *   `scene_data.egps_sparse`: Sparse tensor of expanded GPS data.
        *   **Data Count**: `all_pts` = **193,488** (for scene `ortho1_10_129`).

**`[END OF PHASE 0]`**

#### **Phase 1: DeepAAT Neural Network Inference**

*   **Description**: The core of the DeepAAT paper. The prepared `scene_data` is fed into the trained model to get a robust initial guess for poses and a reliable outlier mask.
*   **Theoretical Concepts**: *Spatial-Spectral Feature Aggregation*, *Global Consistency-based Outlier Rejecting Module*, *Pose Decode Module*.

**`[START OF PHASE 1]`**

1.  **Model Inference (in `inference.py`)**
    *   **Code**: `pred_mask, pred_cam = model(scene_data)`
    *   **Action**: The `model`'s forward pass processes the sparse input tensors.
        *   The *Outlier Rejecting Module* analyzes global consistency and generates a probability score for each feature match.
        *   The *Pose Decode Module* predicts the camera orientation and the positional offset relative to the GPS data.
    *   **Input**: The `scene_data` object.
    *   **Output**: Two dictionaries:
        *   `pred_mask`: A sparse tensor of raw outlier scores (probabilities). **Example shape**: Values for 893,353 observations.
        *   `pred_cam`: A dictionary containing `{'quats': [129, 4], 'ts': [129, 3]}`.

**`[END OF PHASE 1]`**

#### **Phase 2: Post-Processing and Initial Triangulation**

*   **Description**: This phase translates the network's abstract output into concrete geometric entities (poses and a 3D point cloud).
*   **Code**: `prepare_predictions_2()` in `evaluation.py`.

**`[START OF PHASE 2]`**

1.  **Outlier Filtering**
    *   **Action**: A threshold (e.g., 0.5) is applied to the raw scores in `pred_mask` to create a binary mask of inliers/outliers. This mask is used to filter the original 2D points (`raw_xs`).
    *   **Input**: `pred_mask`, `raw_xs`.
    *   **Output**: Filtered 2D observations `xs`.
    *   **Data Link**: The number of points drops. `all_pts` (193,488) -> `pred_pts` (**131,290**).

2.  **Initial Pose Calculation**
    *   **Action**: The predicted quaternions (`pred_cam['quats']`) are converted to rotation matrices (`Rs`). The predicted translational offsets (`pred_cam['ts']`) are added to the initial GPS data (`data.gpss`) to get the absolute camera positions (`ts`).
    *   **Input**: `pred_cam`, `data.gpss`.
    *   **Output**: Initial Camera Poses (`Rs`, `ts`). **Metrics**: `ts_mean` = 4.144m, `Rs_mean` = 1.483°.

3.  **Initial 3D Triangulation**
    *   **Action**: A classical triangulation algorithm is used to compute the 3D positions of the filtered points using their 2D observations (`xs`) and the initial camera poses (`Rs`, `ts`).
    *   **Input**: `xs`, `Rs`, `ts`, `Ks`.
    *   **Output**: An initial 3D point cloud `pts3D_pred`. **Metric**: `our_repro` = 44.77 pixels.

**`[END OF PHASE 2]`**

#### **Phase 3: Refined Bundle Adjustment (BA)**

*   **Description**: The final optimization step. This takes the strong initial guess from the network and refines it for maximum precision and consistency.
*   **Theoretical Concept**: *Bundle Adjustment*, a non-linear least squares optimization.
*   **Code**: `euc_ba()` in `ba_functions.py` (the `refined=True` path).

**`[START OF PHASE 3]`**

1.  **Stage 1: Pose Refinement**
    *   **Action**: Run BA using the initial poses and the **filtered, high-confidence** set of 131,290 points.
    *   **Goal**: To get very stable and accurate camera pose estimates.
    *   **Input**: Initial Poses, `xs` (131k points), `pts3D_pred` (131k points).
    *   **Output**: Intermediate refined camera poses.

2.  **Stage 2a: Point Recovery via Re-Triangulation**
    *   **Code**: `dlt_triangulation()`
    *   **Action**: Use the stable camera poses from Stage 1 and the **complete set of original 2D observations (`raw_xs`)** to triangulate a new, denser 3D point cloud. This attempts to recover valid points that the network may have filtered out.
    *   **Input**: Stable poses from Stage 1, `raw_xs` (for all 193k points).
    *   **Output**: A new, larger intermediate point cloud.

3.  **Stage 2b: Final Full-Scale BA**
    *   **Action**: Run BA a final time on the stable poses and the new, denser point cloud. This performs a final polish and uses its own internal threshold (`repro_thre`) to discard any remaining geometric outliers.
    *   **Input**: Stable poses, new denser point cloud.
    *   **Output**: The final, highly consistent results:
        *   `Rs_ba`, `ts_ba`: Final refined camera poses.
        *   `Xs_ba`: Final refined 3D point cloud.
    *   **Data Link**: The final point count in `Xs_ba` is **160,222**.

**`[END OF PHASE 3]`**
a
#### **Phase 4: Final Evaluation and Reporting**

*   **Description**: The system's performance is numerically quantified by comparing all generated results against the ground truth.
*   **Code**: `compute_errors()` in `evaluation.py` and the pandas DataFrame creation in `inference.py`.

**`[START OF PHASE 4]`**

1.  **Calculate All Metrics**
    *   **Action**: The code systematically computes the error for each stage.
        *   **Outlier Metrics**: Compares `premask` vs `gtmask` -> `F1` (**0.983**).
        *   **Initial Pose Error**: Compares `Rs`/`ts` vs `Rs_gt`/`ts_gt` -> `ts_mean` (**4.144m**), `Rs_mean` (**1.483°**).
        *   **Final Pose Error**: Compares `Rs_ba`/`ts_ba` vs `Rs_gt`/`ts_gt` -> `ts_ba_mean` (**4.791m**), `Rs_ba_mean` (**1.232°**).
        *   **Final Consistency**: Calculates reprojection error for the final model -> `repro_ba` (**0.47 pixels**).

2.  **Generate Report**
    *   **Action**: All calculated metrics, along with timing information (`Inference time`) and data counts (`all_pts`, etc.), are aggregated into a pandas DataFrame.
    *   **Input**: All metrics from the previous step.
    *   **Output**: The final results table that you provided.

**`[PROCESS COMPLETE]`**