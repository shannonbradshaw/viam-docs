# Part 4: Training ML Models for Glass Detection

In this part, you'll capture training data, label it, and train a machine learning model to detect wine glasses. This model will guide the robot during the pouring operation.

**What you'll learn:**
- Configuring Viam's data capture service
- Labeling images in the Viam app
- Training an object detection model
- Deploying models to your robot
- Understanding ML model packages

**Time required:** 2-3 hours (including model training time)

**Prerequisites:** Complete [Part 3: Frame System & Obstacles](./03-frame-system-obstacles.md)

## Why ML for Glass Detection?

The wine pouring demo needs to locate glasses for two purposes:

1. **Point cloud detection**: Finding glasses on the table using 3D depth data (covered in Part 5)
2. **Pour monitoring**: Detecting when the glass is properly positioned under the bottle during pouring

For pour monitoring, we use the webcam (`cam-glass`) with an ML object detection model. This provides:
- Fast inference (real-time detection)
- Robustness to lighting changes
- Ability to detect partial occlusion (arm in the way)

## Step 1: Configure Data Capture

Viam's data management service captures images from your cameras and syncs them to the cloud for labeling and training.

### 1.1 Add the Data Manager Service

1. Go to **CONFIGURE** → **+** → **Service**
2. Search for `data_manager` and select `data manager / rdk:builtin:builtin`
3. Name it `data_manager`
4. Configure:

```json
{
  "capture_dir": "",
  "sync_interval_mins": 0.1,
  "tags": []
}
```

**Attributes:**
- `capture_dir`: Where to store data locally (empty = default location)
- `sync_interval_mins`: How often to sync to cloud (0.1 = every 6 seconds)
- `tags`: Optional tags for organizing captured data

### 1.2 Enable Capture on the Glass Camera

Edit the `cam-glass` component to add a data capture configuration:

```json
{
  "name": "cam-glass",
  "api": "rdk:component:camera",
  "model": "rdk:builtin:webcam",
  "attributes": {
    "video_path": "video12",
    "width_px": 1920,
    "height_px": 1080
  },
  "service_configs": [
    {
      "type": "data_manager",
      "attributes": {
        "capture_methods": [
          {
            "method": "ReadImage",
            "capture_frequency_hz": 0.5,
            "additional_params": {
              "mime_type": "image/jpeg"
            }
          }
        ]
      }
    }
  ]
}
```

This captures an image every 2 seconds (0.5 Hz).

### 1.3 Save and Verify Capture

1. Click **Save**
2. Go to the **DATA** tab in the Viam app
3. Wait a minute, then refresh
4. You should see images appearing from `cam-glass`

## Step 2: Capture Training Data

Good ML models need diverse training data. For glass detection, capture images with:

### 2.1 Variation in Glass Position

Move the glass to different positions in the camera's view:
- Center, left, right
- Near, far
- Multiple glasses
- No glasses (negative examples)

### 2.2 Variation in Lighting

Capture images under different conditions:
- Bright lighting
- Dim lighting
- Side lighting
- With/without reflections

### 2.3 Variation in Background

Include variety in what's behind/around the glass:
- Empty table
- With bottle in view
- With arm in view
- Different colored surfaces

### 2.4 Target: 100-500 Images

For a robust model, aim for:
- At least 100 images total
- At least 20 negative examples (no glass)
- Even distribution across variations

Let data capture run while you move the glass around. This might take 30-60 minutes.

### 2.5 Stop Capture When Done

Once you have enough images, either:
- Set `capture_frequency_hz` to 0
- Remove the `service_configs` section
- Or leave it running (images are cheap to store)

## Step 3: Label Your Data

Now you'll draw bounding boxes around glasses in your captured images.

### 3.1 Access the Labeling Interface

1. Go to the **DATA** tab
2. Click **Label data** in the top right
3. Create a new label: `wine_glass`

### 3.2 Label Images

For each image:
1. If a glass is visible, draw a tight bounding box around it
2. Assign the label `wine_glass`
3. If multiple glasses, draw a box around each one
4. For images with no glass, skip them (or add a "background" label if you want explicit negatives)

**Tips for good bounding boxes:**
- Include the entire glass (rim to base)
- Keep boxes tight but include a small margin
- Be consistent - if you include reflections for one, include them for all

### 3.3 Label Efficiently

The labeling interface supports keyboard shortcuts:
- `N` - Next image
- `P` - Previous image
- `D` - Draw box
- `1-9` - Select label by number

Aim to label at least 80% of your captured images.

## Step 4: Train the Model

Viam's training pipeline creates TensorFlow Lite models optimized for edge deployment.

### 4.1 Start Training

1. Go to the **DATA** tab
2. Click **Train model**
3. Configure:
   - **Model name**: `pour-glass-find-model`
   - **Model type**: Object Detection
   - **Labels**: Select `wine_glass`
   - **Training data**: Your labeled images

4. Click **Train**

### 4.2 Wait for Training

Training typically takes 15-60 minutes depending on:
- Number of images
- Model complexity
- Current queue

You can monitor progress in the **TRAINING** tab.

### 4.3 Training Results

When complete, you'll see:
- **mAP (mean Average Precision)**: Higher is better (0.7+ is good)
- **Sample predictions**: Visual examples of model output
- **Training graphs**: Loss curves

If mAP is low (<0.5), you may need:
- More training images
- Better labeling (consistent, tight boxes)
- More variation in training data

## Step 5: Deploy the Model

### 5.1 Create a Model Package

After training, the model becomes a **package** in your organization:

1. Go to **FLEET** → **MODELS**
2. Find your trained model
3. Note the package path (e.g., `e76d1b3b-0468-4efd-bb7f-fb1d2b352fcb/pour-glass-find-model`)

### 5.2 Add the Package to Your Machine

In your machine's configuration, add a package reference:

```json
{
  "packages": [
    {
      "name": "pour-glass-find-model",
      "package": "YOUR_ORG_ID/pour-glass-find-model",
      "type": "ml_model",
      "version": "latest"
    }
  ]
}
```

Replace `YOUR_ORG_ID` with your organization ID from the package path.

### 5.3 Add the ML Model Service

The `mlmodel` service loads and runs your trained model:

1. Go to **CONFIGURE** → **+** → **Service**
2. Search for `tflite` and select `ML model / viam:mlmodel-tflite:tflite_cpu`
3. Name it `pour-glass-find-mlmodel`
4. Configure:

```json
{
  "package_reference": "YOUR_ORG_ID/pour-glass-find-model",
  "model_path": "${packages.ml_model.pour-glass-find-model}/pour-glass-find-model.tflite",
  "label_path": "${packages.ml_model.pour-glass-find-model}/labels.txt"
}
```

**Attributes:**
- `package_reference`: Points to the package in the registry
- `model_path`: Path to the TFLite model file (uses variable syntax)
- `label_path`: Path to the labels file

### 5.4 Add the TFLite Module

If not already present, add the TFLite CPU module:

```json
{
  "modules": [
    {
      "type": "registry",
      "name": "viam_tflite_cpu",
      "module_id": "viam:tflite_cpu",
      "version": "latest"
    }
  ]
}
```

Save your configuration.

## Step 6: Test the Model

### 6.1 Test in the Viam App

1. Go to the **CONTROL** tab
2. Find `pour-glass-find-mlmodel` in the services list
3. Upload a test image or select from camera
4. View detection results

You should see bounding boxes around detected glasses with confidence scores.

### 6.2 Test via SDK

```python
from viam.services.mlmodel import MLModel
from viam.components.camera import Camera
from PIL import Image
import io

async def test_detection(robot):
    camera = Camera.from_robot(robot, "cam-glass")
    ml_model = MLModel.from_robot(robot, "pour-glass-find-mlmodel")
    
    # Capture an image
    image = await camera.get_image()
    
    # Run inference (the vision service handles this more elegantly)
    # This is just to verify the model loads
    metadata = await ml_model.metadata()
    print(f"Model inputs: {metadata.inputs}")
    print(f"Model outputs: {metadata.outputs}")
```

## Step 7: Understanding Model Packages

Viam's ML model package system provides:

### Version Control
```json
{
  "version": "2025-12-05T13-20-23"  // Specific version
  // or
  "version": "latest"  // Always use newest
}
```

### Automatic Deployment
When you update a model and use `"version": "latest"`, the robot automatically downloads the new version on next restart.

### Package Structure
A deployed model package contains:
```
pour-glass-find-model/
├── pour-glass-find-model.tflite  # The model weights
├── labels.txt                     # Class labels
└── metadata.json                  # Model info
```

## Step 8: Training a Second Model (Glass Location)

The demo uses two models:

1. **pour-glass-find-model**: Detects glasses in the webcam view during pouring
2. **pour-glass-locate-on-top-model**: Detects glasses from above (bird's eye view) for initial positioning

For the second model, repeat the training process using images from the RealSense cameras looking down at the table.

### Different Perspective, Different Model

The "locate" model sees:
- Top-down view of glasses
- Circular glass rims
- Table surface as background

The "find" model sees:
- Side view during pouring
- Glass profile/silhouette
- Arm and bottle potentially in frame

Train separate models for each perspective for best results.

## Checkpoint: What You've Accomplished

✅ Configured Viam's data capture service  
✅ Captured diverse training images  
✅ Labeled images with bounding boxes  
✅ Trained an object detection model  
✅ Deployed the model to your robot  
✅ Tested model inference  
✅ Understood ML model packages and versioning  

## Troubleshooting

### No images appearing in DATA tab
- Check data manager service is configured
- Verify `service_configs` is set on the camera
- Check `sync_interval_mins` is not 0
- Review logs for sync errors

### Training fails
- Ensure you have at least 10 labeled images
- Check that labels are consistent
- Verify images uploaded successfully

### Low model accuracy
- Add more training data
- Ensure bounding boxes are tight and consistent
- Include more negative examples
- Try different lighting conditions

### Model doesn't load
- Check package path matches exactly
- Verify the model finished training successfully
- Check the TFLite module is configured
- Review error logs

### Slow inference
- TFLite CPU is slower than GPU alternatives
- Reduce image resolution
- The vision service batches requests efficiently

## Best Practices for ML Models

### Label Consistency
- Use the same labeling conventions throughout
- When in doubt, include the object (false positives are better than false negatives for robotics)

### Data Diversity
- More variation = more robust model
- Include edge cases (partial occlusion, unusual angles)

### Versioning Strategy
- Use `"version": "latest"` during development
- Pin to specific versions for production deployments

### Retraining
- When the robot encounters failures, capture those images
- Add them to training data and retrain
- This "active learning" improves the model over time

## Next Steps

In [Part 5: Vision Service Pipelines](./05-vision-pipelines.md), you'll connect your ML model to a vision service that processes camera feeds in real-time and integrates with the point cloud detection system.

---

## Reference: ML Model Configuration

```json
{
  "services": [
    {
      "name": "data_manager",
      "api": "rdk:service:data_manager",
      "model": "rdk:builtin:builtin",
      "attributes": {
        "capture_dir": "",
        "sync_interval_mins": 0.1,
        "tags": []
      }
    },
    {
      "name": "pour-glass-find-mlmodel",
      "api": "rdk:service:mlmodel",
      "model": "viam:mlmodel-tflite:tflite_cpu",
      "attributes": {
        "package_reference": "YOUR_ORG_ID/pour-glass-find-model",
        "model_path": "${packages.ml_model.pour-glass-find-model}/pour-glass-find-model.tflite",
        "label_path": "${packages.ml_model.pour-glass-find-model}/labels.txt"
      }
    }
  ],
  "modules": [
    {
      "type": "registry",
      "name": "viam_tflite_cpu",
      "module_id": "viam:tflite_cpu",
      "version": "latest"
    }
  ],
  "packages": [
    {
      "name": "pour-glass-find-model",
      "package": "YOUR_ORG_ID/pour-glass-find-model",
      "type": "ml_model",
      "version": "latest"
    }
  ]
}
```

Camera with data capture:
```json
{
  "name": "cam-glass",
  "api": "rdk:component:camera",
  "model": "rdk:builtin:webcam",
  "attributes": {
    "video_path": "video12",
    "width_px": 1920,
    "height_px": 1080
  },
  "service_configs": [
    {
      "type": "data_manager",
      "attributes": {
        "capture_methods": [
          {
            "method": "ReadImage",
            "capture_frequency_hz": 0.5,
            "additional_params": {
              "mime_type": "image/jpeg"
            }
          }
        ]
      }
    }
  ]
}
```
