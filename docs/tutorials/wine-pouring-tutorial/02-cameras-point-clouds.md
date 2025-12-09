# Part 2: Depth Cameras & Point Clouds

In this part, you'll add Intel RealSense depth cameras to your robot and learn how to work with 3D point cloud data. These cameras will enable your robot to detect and locate glasses in 3D space.

**What you'll learn:**
- Configuring Intel RealSense D435 cameras
- Understanding depth vs. color sensors
- Working with point clouds
- Cropping and merging point cloud data
- Debugging camera issues

**Time required:** 45 minutes - 1 hour

**Prerequisites:** Complete [Part 1: Hardware Setup](./01-hardware-setup.md)

## Why Depth Cameras?

Regular cameras give you 2D images - useful for recognizing *what* objects are, but they can't tell you *where* objects are in 3D space. For a robot to pick up a glass, it needs to know:
- X, Y, Z coordinates of the glass
- The glass's orientation
- How far to reach

Depth cameras solve this by measuring distance at every pixel. The RealSense D435 uses stereo infrared sensors to create a "depth map" that can be combined with the color image to produce a **point cloud** - a 3D representation of the scene where every point has both color and position data.

## Step 1: Install RealSense Prerequisites

Before configuring the cameras in Viam, ensure your system can communicate with them.

### 1.1 Install librealsense

On Ubuntu:
```bash
# Add Intel's package repository
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE
sudo add-apt-repository "deb https://librealsense.intel.com/Debian/apt-repo $(lsb_release -cs) main"

# Install the library and tools
sudo apt update
sudo apt install librealsense2-dkms librealsense2-utils
```

### 1.2 Verify Camera Detection

Connect your RealSense cameras and run:
```bash
rs-enumerate-devices
```

You should see output like:
```
Device info:
    Name                          : Intel RealSense D435
    Serial Number                 : 022422071219
    ...

Device info:
    Name                          : Intel RealSense D435
    Serial Number                 : 243222071262
    ...
```

**Note the serial numbers** - you'll need these to distinguish between your left and right cameras.

### 1.3 Test with RealSense Viewer (Optional)

```bash
realsense-viewer
```

This opens a GUI where you can see the camera streams. Verify both cameras work independently. Close the viewer before proceeding (it will conflict with Viam).

## Step 2: Add the RealSense Module

### 2.1 Add the First Camera

1. In the Viam app, go to **CONFIGURE**
2. Click **+** → **Component**
3. Search for `realsense` and select `camera / viam:realsense`
4. Click **Add module** when prompted
5. Name it `left-cam`
6. Click **Create**

### 2.2 Configure Camera Attributes

In the configuration panel:

```json
{
  "sensors": ["color", "depth"],
  "width_px": 640,
  "height_px": 480,
  "serial": "022422071219"
}
```

**Attribute explanations:**
- `sensors`: Which data streams to enable. Order matters - the first is the "primary" sensor
- `width_px`, `height_px`: Resolution (lower = faster, higher = more detail)
- `serial`: The camera's serial number (required when multiple RealSense cameras are connected)

### 2.3 Add the Second Camera

Repeat for the right camera:
1. Click **+** → **Component** → `camera / viam:realsense`
2. Name it `right-cam`
3. Configure:

```json
{
  "sensors": ["color", "depth"],
  "width_px": 640,
  "height_px": 480,
  "serial": "243222071262"
}
```

### 2.4 Save and Test

1. Click **Save**
2. Go to the **CONTROL** tab
3. Find the camera panels
4. Click the camera icon to view the stream

You should see color images from both cameras. Toggle the sensor dropdown to view depth data (displayed as a colorized depth map).

## Step 3: Add the Glass Detection Webcam

For ML-based glass detection during pouring, we use a standard webcam positioned to view the pour zone.

### 3.1 Find the Webcam Video Path

```bash
# List video devices
v4l2-ctl --list-devices
```

Look for your webcam and note its path (e.g., `/dev/video0` or `video12`).

### 3.2 Add the Webcam Component

1. Click **+** → **Component**
2. Search for `webcam` and select `camera / rdk:builtin:webcam`
3. Name it `cam-glass`
4. Configure:

```json
{
  "video_path": "video12",
  "width_px": 1920,
  "height_px": 1080
}
```

> **Note**: The `video_path` may be just the number (e.g., `"video12"`) or a full path (e.g., `"/dev/video12"`). Try both if one doesn't work.

Save and verify the webcam stream in the CONTROL tab.

## Step 4: Understanding Point Clouds

### 4.1 What is a Point Cloud?

A point cloud is a collection of 3D points, where each point has:
- **Position**: X, Y, Z coordinates (typically in millimeters from the camera)
- **Color**: RGB values from the color camera
- **Optional metadata**: Confidence scores, normals, etc.

The RealSense creates point clouds by:
1. Capturing a depth image (distance at each pixel)
2. Capturing a color image  
3. Aligning them using camera intrinsics
4. Converting 2D pixels + depth → 3D points

### 4.2 View Point Clouds in Viam

In the CONTROL tab, switch the camera view mode to **Point Cloud**. You'll see a 3D visualization you can rotate and zoom.

### 4.3 Accessing Point Clouds Programmatically

```python
from viam.components.camera import Camera

async def get_point_cloud(robot):
    camera = Camera.from_robot(robot, "left-cam")
    
    # Get point cloud data
    pcd, _ = await camera.get_point_cloud()
    
    # pcd is binary PCD data
    # You can save it or process it with libraries like Open3D
    with open("scene.pcd", "wb") as f:
        f.write(pcd)
    
    print(f"Captured point cloud with {len(pcd)} bytes")
```

## Step 5: Point Cloud Cropping

For the wine pouring demo, we don't want to process the entire scene - only the region where glasses might be. The `erh:vmodutils` module provides camera models that crop and transform point clouds.

### 5.1 Add the vmodutils Module

1. Go to **CONFIGURE** → **+** → **Local module** (or search in registry)
2. Search for `vmodutils` 
3. Add the module `erh:vmodutils`

Or add it directly in JSON:
```json
{
  "modules": [
    {
      "type": "registry",
      "name": "erh_vmodutils",
      "module_id": "erh:vmodutils",
      "version": "latest"
    }
  ]
}
```

### 5.2 Create a Cropped Camera

The `pc-crop-camera` model filters a point cloud to only include points within a 3D bounding box.

Add a new component:
1. Click **+** → **Component**
2. Select `camera` type
3. Enter model: `erh:vmodutils:pc-crop-camera`
4. Name it `cam-left-cup-crop`
5. Configure:

```json
{
  "src": "left-cam",
  "min": {
    "X": 225,
    "Y": 120,
    "Z": 40
  },
  "max": {
    "X": 410,
    "Y": 740,
    "Z": 130
  }
}
```

**What this does:**
- Takes the point cloud from `left-cam`
- Keeps only points where:
  - X is between 225 and 410 mm
  - Y is between 120 and 740 mm
  - Z (height above surface) is between 40 and 130 mm
- This region corresponds to where glasses sit on the table

### 5.3 Understanding the Crop Coordinates

The coordinates are relative to the camera's reference frame:
- **X**: Horizontal (left-right from camera's perspective)
- **Y**: Horizontal (forward-back from camera's perspective)  
- **Z**: Vertical (up-down, where the camera looks down)

You'll likely need to adjust these values based on:
- Your camera mounting position
- Your table height
- Where glasses will be placed

**Tuning tip:** Use the point cloud viewer in Viam to identify coordinates. Move a glass to different positions and note its XYZ coordinates, then set your crop bounds accordingly.

### 5.4 Create Right Camera Crop

Add another cropped camera for the right side:

```json
{
  "name": "cam-right-cup-crop",
  "api": "rdk:component:camera",
  "model": "erh:vmodutils:pc-crop-camera",
  "attributes": {
    "src": "right-cam",
    "min": {
      "X": 225,
      "Y": 120,
      "Z": 40
    },
    "max": {
      "X": 410,
      "Y": 740,
      "Z": 130
    }
  }
}
```

## Step 6: Merging Point Clouds

The demo setup has cameras on both sides. To get a complete view of the glass placement area, we merge their point clouds.

### 6.1 Add a Merged Camera

Add a component with model `erh:vmodutils:pc-merge`:

```json
{
  "name": "cam-merged-cup",
  "api": "rdk:component:camera",
  "model": "erh:vmodutils:pc-merge",
  "attributes": {
    "cameras": ["cam-left-cup-crop", "cam-right-cup-crop"]
  }
}
```

This creates a virtual camera that:
1. Gets point clouds from both cropped cameras
2. Transforms them into a common reference frame
3. Combines them into a single point cloud

> **Important:** For point cloud merging to work correctly, the cameras must have properly configured frames in the frame system. We'll cover this in Part 3.

## Step 7: The Camera Pipeline

Here's what we've built:

```
┌─────────────┐     ┌─────────────────────┐     ┌─────────────────┐
│  left-cam   │────►│  cam-left-cup-crop  │────►│                 │
│ (RealSense) │     │  (crop to table)    │     │ cam-merged-cup  │
└─────────────┘     └─────────────────────┘     │ (combined view) │
                                                │                 │
┌─────────────┐     ┌─────────────────────┐     │                 │
│  right-cam  │────►│  cam-right-cup-crop │────►│                 │
│ (RealSense) │     │  (crop to table)    │     └─────────────────┘
└─────────────┘     └─────────────────────┘
```

This pipeline:
1. Captures depth data from physical cameras
2. Crops to the region of interest
3. Merges for a unified view

The merged camera will be used by the vision service to detect glasses.

## Step 8: Verify Your Configuration

Test each camera in the CONTROL tab:
1. `left-cam` - Should show color image, switch to depth to verify
2. `right-cam` - Same verification
3. `cam-glass` - Should show webcam feed
4. `cam-left-cup-crop` - Point cloud view should show only the cropped region
5. `cam-right-cup-crop` - Same verification
6. `cam-merged-cup` - Should combine both cropped views (may need frame system setup first)

## Checkpoint: What You've Accomplished

✅ Installed RealSense prerequisites on your Linux system  
✅ Configured two RealSense depth cameras with specific serial numbers  
✅ Added a webcam for ML-based glass detection  
✅ Created cropped camera views to focus on the glass placement area  
✅ Set up point cloud merging for combined 3D perception  
✅ Understood the camera pipeline architecture  

## Troubleshooting

### Camera not detected
- Check USB connections (use USB 3.0 ports)
- Run `rs-enumerate-devices` to verify detection
- Install udev rules:
  ```bash
  sudo wget -O /etc/udev/rules.d/99-realsense-libusb.rules \
    https://raw.githubusercontent.com/IntelRealSense/librealsense/master/config/99-realsense-libusb.rules
  sudo udevadm control --reload-rules
  sudo udevadm trigger
  ```

### "Permission denied" or "failed to set power state"
- udev rules not installed (see above)
- Try unplugging and reconnecting the camera
- Reboot the computer

### Wrong camera assigned to wrong name
- Double-check serial numbers in configuration
- Use `rs-enumerate-devices` to confirm which is which
- Physical labels on cameras help!

### Cropped camera shows nothing
- Crop bounds may not include any points
- Verify coordinates by viewing the raw camera's point cloud first
- Adjust min/max values based on actual scene geometry

### Point cloud merge fails
- Cameras need frame definitions (covered in Part 3)
- Check that both source cameras are working
- Review error logs in the LOGS tab

## Next Steps

In [Part 3: The Frame System & Obstacle Definition](./03-frame-system-obstacles.md), you'll learn how Viam models spatial relationships between components and how to define obstacles for safe motion planning.

---

## Reference: Camera Configuration

```json
{
  "components": [
    {
      "name": "left-cam",
      "api": "rdk:component:camera",
      "model": "viam:camera:realsense",
      "attributes": {
        "sensors": ["color", "depth"],
        "width_px": 640,
        "height_px": 480,
        "serial": "022422071219"
      }
    },
    {
      "name": "right-cam",
      "api": "rdk:component:camera",
      "model": "viam:camera:realsense",
      "attributes": {
        "sensors": ["color", "depth"],
        "width_px": 640,
        "height_px": 480,
        "serial": "243222071262"
      }
    },
    {
      "name": "cam-glass",
      "api": "rdk:component:camera",
      "model": "rdk:builtin:webcam",
      "attributes": {
        "video_path": "video12",
        "width_px": 1920,
        "height_px": 1080
      }
    },
    {
      "name": "cam-left-cup-crop",
      "api": "rdk:component:camera",
      "model": "erh:vmodutils:pc-crop-camera",
      "attributes": {
        "src": "left-cam",
        "min": { "X": 225, "Y": 120, "Z": 40 },
        "max": { "X": 410, "Y": 740, "Z": 130 }
      }
    },
    {
      "name": "cam-right-cup-crop",
      "api": "rdk:component:camera",
      "model": "erh:vmodutils:pc-crop-camera",
      "attributes": {
        "src": "right-cam",
        "min": { "X": 225, "Y": 120, "Z": 40 },
        "max": { "X": 410, "Y": 740, "Z": 130 }
      }
    },
    {
      "name": "cam-merged-cup",
      "api": "rdk:component:camera",
      "model": "erh:vmodutils:pc-merge",
      "attributes": {
        "cameras": ["cam-left-cup-crop", "cam-right-cup-crop"]
      }
    }
  ],
  "modules": [
    {
      "type": "registry",
      "name": "viam_realsense",
      "module_id": "viam:realsense",
      "version": "latest"
    },
    {
      "type": "registry",
      "name": "erh_vmodutils",
      "module_id": "erh:vmodutils",
      "version": "latest"
    }
  ]
}
```
