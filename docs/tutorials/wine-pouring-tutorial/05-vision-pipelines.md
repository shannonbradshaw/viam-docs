# Part 5: Vision Service Pipelines

In this part, you'll connect your ML models and cameras into a unified vision system. You'll build a pipeline that detects glasses using both 2D ML inference and 3D point cloud segmentation.

**What you'll learn:**
- Configuring vision services
- Connecting ML models to vision services
- Point cloud segmentation for object detection
- Building camera-to-detection pipelines
- Testing detections programmatically

**Time required:** 45 minutes - 1 hour

**Prerequisites:** Complete [Part 4: Training ML Models](./04-ml-training.md)

## Vision Architecture Overview

The wine pouring demo uses two complementary vision approaches:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ML-BASED DETECTION                          │
│  (For monitoring the pour - 2D bounding boxes)                     │
│                                                                     │
│  cam-glass ──► pour-glass-find-mlmodel ──► pour-glass-find-service │
│  (webcam)       (TFLite model)              (vision service)       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                   POINT CLOUD SEGMENTATION                         │
│  (For locating glasses on table - 3D positions)                    │
│                                                                     │
│  cam-merged-cup ──► cup-finder-segment                             │
│  (merged depth)     (segmentation service)                         │
└─────────────────────────────────────────────────────────────────────┘
```

## Step 1: Create the ML Vision Service

The vision service wraps your ML model and provides a standard API for detections.

### 1.1 Add the Vision Service

1. Go to **CONFIGURE** → **+** → **Service**
2. Search for `mlmodel` and select `vision / rdk:builtin:mlmodel`
3. Name it `pour-glass-find-service`
4. Configure:

```json
{
  "mlmodel_name": "pour-glass-find-mlmodel",
  "camera_name": "cam-glass",
  "default_minimum_confidence": 0.4
}
```

**Attributes:**
- `mlmodel_name`: The ML model service to use for inference
- `camera_name`: Default camera for detections
- `default_minimum_confidence`: Threshold for detections (0.0-1.0)

### 1.2 Understanding Confidence Thresholds

The `default_minimum_confidence` of 0.4 means:
- Detections with confidence ≥40% are returned
- Lower threshold = more detections, more false positives
- Higher threshold = fewer detections, might miss valid glasses

Start at 0.4 and adjust based on testing:
- Too many false positives? Raise to 0.5-0.6
- Missing glasses? Lower to 0.3

## Step 2: Test ML Detections

### 2.1 Test in the Control Tab

1. Go to **CONTROL** tab
2. Find `pour-glass-find-service` in the services list
3. Click to expand
4. View the camera stream with detection overlays

You should see bounding boxes around detected glasses.

### 2.2 Test via SDK

```python
from viam.services.vision import Vision, Detection

async def test_ml_detection(robot):
    vision = Vision.from_robot(robot, "pour-glass-find-service")
    
    # Get detections from the configured camera
    detections = await vision.get_detections_from_camera("cam-glass")
    
    print(f"Found {len(detections)} objects:")
    for d in detections:
        print(f"  - {d.class_name}: {d.confidence:.2%} at "
              f"({d.x_min}, {d.y_min}) to ({d.x_max}, {d.y_max})")
```

Sample output:
```
Found 1 objects:
  - wine_glass: 87.34% at (423, 156) to (612, 489)
```

## Step 3: Point Cloud Segmentation

For 3D glass positioning, we use point cloud segmentation rather than ML. This approach:
- Doesn't require training data
- Provides precise 3D coordinates
- Works by finding clusters of points above the table surface

### 3.1 Add the Obstacles Point Cloud Module

1. Go to **CONFIGURE** → **+** → **Module** (if not already added)
2. Search for `obstacles-pointcloud`
3. Add `viam:obstacles-pointcloud`

Or in JSON:
```json
{
  "modules": [
    {
      "type": "registry",
      "name": "viam_obstacles-pointcloud",
      "module_id": "viam:obstacles-pointcloud",
      "version": "latest"
    }
  ]
}
```

### 3.2 Configure the Segmentation Service

Add a vision service using the obstacles-pointcloud model:

```json
{
  "name": "cup-finder-segment",
  "api": "rdk:service:vision",
  "model": "viam:obstacles-pointcloud:obstacles-pointcloud",
  "attributes": {
    "camera_name": "cam-merged-cup",
    "min_points_in_plane": 100000000,
    "min_points_in_segment": 100,
    "clustering_strictness": 0.3,
    "ground_plane_normal_vec": {
      "z": 1
    }
  }
}
```

**Attributes explained:**
- `camera_name`: The point cloud camera to process
- `min_points_in_plane`: Minimum points to consider a plane (set very high to disable plane detection)
- `min_points_in_segment`: Minimum points for a valid object cluster
- `clustering_strictness`: How strictly to group nearby points (0-1, lower = more lenient)
- `ground_plane_normal_vec`: The "up" direction for filtering ground plane (Z=1 means up is +Z)

### 3.3 How Point Cloud Segmentation Works

The segmentation service:
1. Takes a point cloud from the merged camera
2. Optionally removes the ground plane (table surface)
3. Clusters remaining points into groups
4. Returns each cluster as a detected "obstacle"

For our setup:
- The cropped cameras already filtered to just above the table
- Glasses appear as clusters of points
- Each cluster becomes a detection with 3D position

## Step 4: Test Point Cloud Detections

### 4.1 Test via SDK

```python
from viam.services.vision import Vision

async def test_pointcloud_detection(robot):
    vision = Vision.from_robot(robot, "cup-finder-segment")
    
    # Get 3D object detections
    detections = await vision.get_object_point_clouds("cam-merged-cup")
    
    print(f"Found {len(detections)} objects in 3D:")
    for i, obj in enumerate(detections):
        # obj.geometries contains the 3D shape
        # obj.point_cloud contains the raw points
        center = obj.geometries[0].center if obj.geometries else None
        if center:
            print(f"  Object {i}: center at "
                  f"x={center.x:.1f}, y={center.y:.1f}, z={center.z:.1f} mm")
```

Sample output:
```
Found 2 objects in 3D:
  Object 0: center at x=312.4, y=245.8, z=85.2 mm
  Object 1: center at x=298.1, y=567.3, z=84.9 mm
```

These 3D coordinates can be used directly for arm motion planning!

## Step 5: Create the ML Crop Camera

The demo uses a clever optimization: crop the camera view based on ML detections, then run more detailed analysis on just that region.

### 5.1 Add the Detection Crop Camera

The `pc-detect-crop-camera` model from vmodutils crops a point cloud to the region where detections are found:

```json
{
  "name": "glass-finder-left-crop",
  "api": "rdk:component:camera",
  "model": "erh:vmodutils:pc-detect-crop-camera",
  "attributes": {
    "src": "left-cam",
    "service": "glass-finder-first-service"
  }
}
```

This camera:
1. Gets detections from `glass-finder-first-service`
2. Crops the point cloud from `left-cam` to just those detection regions
3. Provides a focused view for precise positioning

### 5.2 Configure the Supporting Service

Add an ML model and vision service for initial glass finding:

```json
{
  "name": "glass-finder-first-model",
  "api": "rdk:service:mlmodel",
  "model": "viam:mlmodel-tflite:tflite_cpu",
  "attributes": {
    "package_reference": "YOUR_ORG_ID/pour-glass-locate-on-top-model",
    "model_path": "${packages.ml_model.pour-glass-locate-on-top-model}/pour-glass-locate-on-top-model.tflite",
    "label_path": "${packages.ml_model.pour-glass-locate-on-top-model}/labels.txt"
  }
}
```

```json
{
  "name": "glass-finder-first-service",
  "api": "rdk:service:vision",
  "model": "rdk:builtin:mlmodel",
  "attributes": {
    "mlmodel_name": "glass-finder-first-model",
    "camera_name": "left-cam",
    "default_minimum_confidence": 0.5
  }
}
```

## Step 6: The Complete Vision Pipeline

Here's the full pipeline for glass detection:

```
                    POUR MONITORING
                    ═══════════════
cam-glass ──► pour-glass-find-mlmodel ──► pour-glass-find-service
(webcam)           (TFLite)                  (2D detections)
                                                    │
                                                    ▼
                                            Used during pour
                                            to verify glass
                                            is in position


                    TABLE SCANNING
                    ══════════════
left-cam ──► glass-finder-first-model ──► glass-finder-first-service
(depth)            (TFLite)                  (2D detections)
    │                                              │
    │                                              │
    ▼                                              ▼
left-cam ────────────────────────────► glass-finder-left-crop
                                       (cropped point cloud)
    │
    │
    ▼
cam-left-cup-crop ◄────────────────────────────────┘
       │
       │     cam-right-cup-crop
       │            │
       ▼            ▼
    cam-merged-cup ──► cup-finder-segment
                         (3D positions)
```

## Step 7: Organize with UI Folders

The configuration is getting complex. Viam supports UI folders to organize components:

```json
{
  "name": "glass-finder-first-service",
  "api": "rdk:service:vision",
  "model": "rdk:builtin:mlmodel",
  "attributes": { ... },
  "ui_folder": {
    "name": "vision"
  }
}
```

Common folder organization:
- `hardware` - Physical components (arms, cameras)
- `vision` - ML models and vision services
- `obstacle` - Obstacle definitions
- `cropped-cams` - Processed camera views
- `app` - Application services

## Step 8: Verify the Complete Pipeline

Test each layer of the pipeline:

### 8.1 Test ML Detection
```python
# Should return 2D bounding boxes
detections = await pour_glass_find_service.get_detections_from_camera("cam-glass")
```

### 8.2 Test Point Cloud Segmentation
```python
# Should return 3D object positions
objects = await cup_finder_segment.get_object_point_clouds("cam-merged-cup")
```

### 8.3 Test the Full Pipeline

```python
from viam.services.vision import Vision
from viam.components.camera import Camera

async def test_full_pipeline(robot):
    # Layer 1: Raw cameras
    left_cam = Camera.from_robot(robot, "left-cam")
    img = await left_cam.get_image()
    print(f"✓ left-cam: captured {img.width}x{img.height} image")
    
    # Layer 2: ML detection
    glass_finder = Vision.from_robot(robot, "glass-finder-first-service")
    detections = await glass_finder.get_detections_from_camera("left-cam")
    print(f"✓ glass-finder-first-service: found {len(detections)} detections")
    
    # Layer 3: Cropped cameras
    merged_cam = Camera.from_robot(robot, "cam-merged-cup")
    pcd, _ = await merged_cam.get_point_cloud()
    print(f"✓ cam-merged-cup: captured {len(pcd)} bytes of point cloud")
    
    # Layer 4: 3D segmentation
    cup_finder = Vision.from_robot(robot, "cup-finder-segment")
    objects = await cup_finder.get_object_point_clouds("cam-merged-cup")
    print(f"✓ cup-finder-segment: found {len(objects)} 3D objects")
    
    # Layer 5: Pour monitoring
    pour_vision = Vision.from_robot(robot, "pour-glass-find-service")
    pour_detections = await pour_vision.get_detections_from_camera("cam-glass")
    print(f"✓ pour-glass-find-service: found {len(pour_detections)} glasses for pour")

# Run test
asyncio.run(test_full_pipeline(robot))
```

Expected output:
```
✓ left-cam: captured 640x480 image
✓ glass-finder-first-service: found 2 detections
✓ cam-merged-cup: captured 245832 bytes of point cloud
✓ cup-finder-segment: found 2 3D objects
✓ pour-glass-find-service: found 1 glasses for pour
```

## Checkpoint: What You've Accomplished

✅ Created ML-based vision services for 2D detection  
✅ Configured point cloud segmentation for 3D object detection  
✅ Built a multi-stage vision pipeline  
✅ Connected cameras, ML models, and vision services  
✅ Organized configuration with UI folders  
✅ Tested each pipeline layer programmatically  

## Vision Service API Reference

The vision service provides these key methods:

### For 2D Detection (ML-based)
```python
# Get detections from a specific camera
detections = await vision.get_detections_from_camera(camera_name)

# Get detections from a provided image
detections = await vision.get_detections(image)

# Each detection has:
# - class_name: str (e.g., "wine_glass")
# - confidence: float (0.0-1.0)
# - x_min, y_min, x_max, y_max: int (bounding box pixels)
```

### For 3D Detection (Point Cloud)
```python
# Get 3D objects from a point cloud camera
objects = await vision.get_object_point_clouds(camera_name)

# Each object has:
# - geometries: list of 3D shapes with center positions
# - point_cloud: raw PCD data for the object
```

### For Classification
```python
# Classify an entire image
classifications = await vision.get_classifications_from_camera(camera_name, count=5)

# Each classification has:
# - class_name: str
# - confidence: float
```

## Troubleshooting

### Vision service returns empty detections
- Check the ML model loaded correctly (view logs)
- Verify confidence threshold isn't too high
- Test with the camera directly to ensure image quality
- Ensure the model was trained on similar images

### Point cloud segmentation finds too many/few objects
- Adjust `min_points_in_segment` (higher = fewer detections)
- Adjust `clustering_strictness` (higher = more separate clusters)
- Check that crop bounds include the objects
- Verify cameras have proper frame definitions

### "Camera not found" errors
- Check camera name spelling matches exactly
- Ensure the camera component is configured and working
- Verify dependencies are met (cropped cameras need source cameras)

### Slow inference
- TFLite CPU inference depends on model size and image resolution
- Consider reducing camera resolution
- Point cloud operations scale with number of points - use aggressive cropping

## Performance Considerations

### ML Inference Speed
- TFLite on CPU: 100-500ms per frame typical
- Reduce image size for faster inference
- Consider `viam:mlmodel-onnx` for GPU acceleration

### Point Cloud Processing
- Cropping reduces processing time significantly
- Merging adds overhead - only merge when necessary
- Segmentation time scales with point count

### Pipeline Optimization
```
Fast path (pour monitoring):
  cam-glass → ML model → detections
  ~200ms total

Full path (table scanning):
  cameras → crop → merge → segment
  ~500-1000ms total
```

## Next Steps

In [Part 6: Motion Planning & Choreography](./06-motion-planning.md), you'll use these vision detections to guide arm movements and create the complete pour sequence.

---

## Reference: Vision Pipeline Configuration

```json
{
  "services": [
    {
      "name": "pour-glass-find-mlmodel",
      "api": "rdk:service:mlmodel",
      "model": "viam:mlmodel-tflite:tflite_cpu",
      "attributes": {
        "package_reference": "YOUR_ORG_ID/pour-glass-find-model",
        "model_path": "${packages.ml_model.pour-glass-find-model}/pour-glass-find-model.tflite",
        "label_path": "${packages.ml_model.pour-glass-find-model}/labels.txt"
      },
      "ui_folder": { "name": "vision" }
    },
    {
      "name": "pour-glass-find-service",
      "api": "rdk:service:vision",
      "model": "rdk:builtin:mlmodel",
      "attributes": {
        "mlmodel_name": "pour-glass-find-mlmodel",
        "camera_name": "cam-glass",
        "default_minimum_confidence": 0.4
      },
      "ui_folder": { "name": "vision" }
    },
    {
      "name": "glass-finder-first-model",
      "api": "rdk:service:mlmodel",
      "model": "viam:mlmodel-tflite:tflite_cpu",
      "attributes": {
        "package_reference": "YOUR_ORG_ID/pour-glass-locate-on-top-model",
        "model_path": "${packages.ml_model.pour-glass-locate-on-top-model}/pour-glass-locate-on-top-model.tflite",
        "label_path": "${packages.ml_model.pour-glass-locate-on-top-model}/labels.txt"
      },
      "ui_folder": { "name": "vision" }
    },
    {
      "name": "glass-finder-first-service",
      "api": "rdk:service:vision",
      "model": "rdk:builtin:mlmodel",
      "attributes": {
        "mlmodel_name": "glass-finder-first-model",
        "camera_name": "left-cam",
        "default_minimum_confidence": 0.5
      },
      "ui_folder": { "name": "vision" }
    },
    {
      "name": "cup-finder-segment",
      "api": "rdk:service:vision",
      "model": "viam:obstacles-pointcloud:obstacles-pointcloud",
      "attributes": {
        "camera_name": "cam-merged-cup",
        "min_points_in_plane": 100000000,
        "min_points_in_segment": 100,
        "clustering_strictness": 0.3,
        "ground_plane_normal_vec": { "z": 1 }
      },
      "ui_folder": { "name": "vision" }
    }
  ],
  "components": [
    {
      "name": "glass-finder-left-crop",
      "api": "rdk:component:camera",
      "model": "erh:vmodutils:pc-detect-crop-camera",
      "attributes": {
        "src": "left-cam",
        "service": "glass-finder-first-service"
      },
      "ui_folder": { "name": "cropped-cams" }
    }
  ],
  "modules": [
    {
      "type": "registry",
      "name": "viam_tflite_cpu",
      "module_id": "viam:tflite_cpu",
      "version": "latest"
    },
    {
      "type": "registry",
      "name": "viam_obstacles-pointcloud",
      "module_id": "viam:obstacles-pointcloud",
      "version": "latest"
    }
  ],
  "packages": [
    {
      "name": "pour-glass-find-model",
      "package": "YOUR_ORG_ID/pour-glass-find-model",
      "type": "ml_model",
      "version": "latest"
    },
    {
      "name": "pour-glass-locate-on-top-model",
      "package": "YOUR_ORG_ID/pour-glass-locate-on-top-model",
      "type": "ml_model",
      "version": "latest"
    }
  ]
}
```
