## 🚁 UAV Video Stabilization using Computer Vision

### ⭐ Why Stabilization Matters for Vehicle Trajectory & Safety Analysis



### 📌 1. Vehicle Trajectory Analysis  
To understand the exact path a vehicle follows, the **background in the video must stay still**.  
When the UAV camera shakes, it becomes impossible to tell whether **the vehicle moved or the camera did**, causing errors in tracking.

 Stabilization removes all unwanted camera motion  
 The video becomes steady  
 Algorithms can track **only the real movement** of vehicles with high accuracy  



### 📌 2. Safety Analysis  
For analysing accidents, near-misses, or risky driving patterns, even small vibrations in the footage can cause:

 Wrong distance calculations  
 Faulty object detection  
 Incorrect conclusions about driver behaviour  

Stable footage ensures:  
 Clear understanding of events  
 Reliable safety metrics  
 Better decision-making for road safety 
 

 ## 🌀 Stabilization Challenges in Our UAV Videos

UAV footage captured in real outdoor environments brings several challenges that directly affect the accuracy of feature tracking, motion estimation, and final stabilization quality.



1. Swaying Trees (Dynamic Background Elements)  
Wind-driven leaf and branch movement introduces **non-rigid, unpredictable motion**, which can fool feature detectors and reduce the accuracy of camera-motion estimation.



2. Moving Vehicles on the Road (Foreground Distractions)  
Vehicles move independently of the camera.  
If these motions are mistakenly treated as background motion, they can create **false motion vectors**, leading to incorrect stabilization and misleading trajectory paths.



3. Water Body with Reflective Surface  
The canal produces constantly changing reflections and ripples, generating **false keypoints and unstable motion vectors**.  
Without masking, these reflections disrupt feature matching and distort trajectory smoothing.



 4. Parallax and Height Variation  
Objects at different depths—roads, vehicles, trees, buildings—move differently when the UAV shifts.  
This **parallax effect**, especially in low-altitude shots, cannot be fully captured by simple transformations like affine or basic homography, making stabilization more complex.

---

### ⭐Proposed Methodology

### 📌 1. Feature Detection – ORB (Oriented FAST and Rotated BRIEF)

ORB is an efficient feature detection and description method designed to identify reliable keypoints across video frames. It combines three core components:

### FAST – Corner Detection
FAST identifies sharp changes in pixel intensity, which typically occur at corners.
- Each pixel is checked against a circular neighbourhood of 16 pixels.
- If enough surrounding pixels are significantly brighter or darker than the center pixel, it is marked as a corner.
- FAST is extremely fast, making it suitable for real-time applications such as video stabilization.

### Orientation – Determining Keypoint Direction
Once corners are detected, ORB assigns an orientation to each keypoint.
- Orientation ensures that the same feature can be recognized even if the image rotates.
- The direction is computed using brightness (intensity) patterns around the keypoint.

### BRIEF – Binary Feature Description
BRIEF generates a compact binary descriptor for each keypoint.
- It compares predefined pairs of pixels around the corner.
- Each comparison is converted into a binary value (0 or 1).
- The resulting binary descriptor allows fast and consistent matching of keypoints across frames.

This combination of FAST, orientation assignment, and BRIEF enables ORB to provide fast, rotation-invariant, and reliable feature detection suitable for UAV video stabilization.

![image alt](https://github.com/SimarSaka/Industrial_Training/blob/main/WhatsApp%20Image%202025-12-04%20at%201.42.46%20AM.jpeg?raw=true)

### 📌  2. Region-Based Feature Filtering

### Water Masking
Water surfaces often appear shiny, reflective, and low-texture. These properties make it difficult for feature detectors to reliably identify stable points.

- Even still water appears to “move” due to changing reflections as the drone moves.
- This can mislead the algorithm into thinking the background itself is shifting.
- Such false motion causes incorrect feature tracking and inaccurate motion estimation.

To prevent this, all shiny, low-texture water regions are masked out before processing. Removing these unstable areas ensures that only reliable background features are used.

### HSV-Based Water Masking
To isolate water regions accurately, the image is converted from RGB to HSV (Hue, Saturation, Value). HSV makes colour-based segmentation easier and more robust.

Process summary:
- Convert each frame to HSV.
- Identify water-like regions based on typical Hue values for natural water (blue/blue-green).
- In OpenCV (Hue range: 0–180), water commonly falls between **90–140**.
- Apply thresholds on Saturation and Value to avoid extremely dark or overly bright pixels.
- Create a binary mask that isolates water regions, which are then removed from feature detection.

This ensures that reflections, ripples, and brightness changes do not introduce false keypoints.

---

### 🔍 Motion Outlier Removal

UAV videos often contain independently moving objects such as vehicles, trees swaying in the wind, and pedestrians. These movements do not represent camera motion, yet the algorithm may mistakenly treat them as background changes.

To ensure accurate motion estimation, only features belonging to stable, non-moving background elements are retained.

### Steps:

1. **Compute Motion Vectors**  
   Calculate how much each feature point moved between two consecutive frames:  
   `motion_vectors = pts2 - pts1`

2. **Compute Motion Magnitudes**  
   Determine the Euclidean distance (magnitude) of each motion vector:  
   `magnitude = sqrt(dx² + dy²)`  
   This gives a scalar measure of motion for each keypoint.

3. **Calculate Median Motion**  
   Compute the median of all magnitudes:  
   `median_motion = median(motion_magnitudes)`  
   This represents the typical background motion.

4. **Compute MAD (Median Absolute Deviation)**  
   Measure deviations from the median:  
   `MAD = median(|motion_magnitudes - median_motion|)`  
   This helps identify points that move abnormally.

5. **Set Threshold**  
   Define an outlier threshold using:  
   `threshold = median_motion + 2 * MAD`  
   The factor 2 approximates a 95% confidence interval for Laplacian-like motion distributions.

6. **Filter Outliers**  
   Keep only points whose motion magnitude falls below the threshold:  
   `mask = motion_magnitude < threshold`

This process removes points on moving vehicles, swaying trees, and reflective water, ensuring only true background features remain.

### Why Not RANSAC?
RANSAC relies purely on geometric consistency and may retain unstable matches, especially in scenes with heavy foreground movement.  
Our approach is **context-aware**, removing points based on how much they move and where they are located, making it more reliable and better suited for UAV footage.

---
### 📌3. Motion Estimation Using Homography

To determine how the camera moves between two video frames, we use a Homography Matrix.  
A homography is a 3×3 matrix that describes how points in one frame correspond to points in the next.

By analyzing how matched feature points shift between frames, the homography captures the underlying camera motion.  
It can represent:

- Translation (shifting the frame left/right or up/down)
- Rotation (slight turning of the frame)
- Scaling (zooming in or out)
- Shearing or tilt (leaning the camera to one side)
- Perspective distortion (viewpoint changes such as objects appearing stretched or angled)

In UAV footage, the drone moves over complex 3D environments—roads, canals, buildings, and trees—where the camera frequently tilts, rotates, or changes altitude.  
These actions introduce noticeable perspective changes, such as roads narrowing in the distance or buildings appearing slanted.

A homography matrix models all these effects, making it ideal for aerial video stabilization.  
In contrast, an Affine transformation (with only 6 parameters) assumes a flat scene and cannot capture depth or perspective changes, making it less suitable for real-world UAV footage.

---

### 📌4. Trajectory Formation

Once homographies are computed for each pair of consecutive frames, we can reconstruct the full motion of the camera across the video.

- Each homography represents a small step in camera movement.
- From each matrix, we extract motion parameters (translation, rotation, etc.).
- These incremental movements are accumulated sequentially.
- The result is a complete trajectory showing how the camera moved over time (left-right, up-down, tilt, zoom, and other motions).

This trajectory forms the basis for stabilization and later smoothing.

---

### 📌5. Adaptive Smoothing of Camera Motion

After constructing the camera trajectory, smoothing is applied to remove unwanted jitter.  
Adaptive smoothing adjusts the smoothing strength depending on how unstable the motion is at any point.

### Short Window (e.g., 5 frames)
Used when the trajectory contains sharp spikes or sudden jumps caused by quick drone movements.  
A short window smooths these abrupt changes without introducing delay.

### Long Window (e.g., 15 frames)
Used in sections where the trajectory is mostly stable.  
This prevents oversmoothing and preserves natural camera movement such as steady pans.

Kalman filtering works well for predictable, smooth motion.  
However, UAV motion is highly irregular due to wind, swaying trees, moving vehicles, and water reflections.  
Kalman predictions can become unreliable under such variability, making adaptive smoothing a more effective choice.

---

### 📌6. Creating Stabilized Homography Matrices

Once the camera trajectory is smoothed, we generate new homography matrices to align the frames with the stabilized motion rather than the original shaky motion.

- Original homographies contain jitter and sudden changes.
- The smoothed trajectory represents the ideal motion path.
- New stabilized homographies are created to map frames to this smooth path.

These stabilized matrices allow all frames to be warped as if the camera had moved smoothly from the beginning.

---

### 📌7. Frame Warping

Each stabilized homography matrix answers the question:  
“How should this frame be transformed so it follows the smooth camera path?”

Using OpenCV’s `cv2.warpPerspective()`:

- The original frame is transformed based on the stabilized homography.
- Pixels are repositioned to match the smooth trajectory.
- Each frame is corrected to remove jitter and unwanted movement.

This step produces visually stable frames while preserving scene geometry.

---

### 📌 8. Writing the Stabilized Video

After every frame has been warped according to its stabilized homography, the frames are assembled into a final output video using OpenCV’s `VideoWriter`.

This pipeline achieved a **57% reduction in overall motion**, a strong result given the presence of swaying trees, moving vehicles, and reflective water surfaces.

The performance is comparable to commercial tools such as Adobe Premiere Pro’s Warp Stabilizer.  
However, this approach is more specifically tuned for UAV footage, making it potentially more effective for aerial scenes with complex multi-layered motion.











