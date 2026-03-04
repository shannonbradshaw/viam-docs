---
linkTitle: "Calibrate Camera to Robot"
title: "Calibrate Camera to Robot"
weight: 30
layout: "docs"
type: "docs"
description: "Compute camera intrinsic and distortion parameters for accurate 2D-to-3D projection."
date: "2025-01-30"
aliases:
  - /build/work-cell-layout/calibrate-camera-to-robot/
---

## What Problem This Solves

A camera captures 2D images, but your machine operates in 3D space.
To convert pixel coordinates into real-world positions, you need to know the camera's intrinsic parameters: focal length, principal point, and lens distortion characteristics.
Without accurate intrinsics, every 2D-to-3D conversion will be wrong, detected objects will appear shifted, and the arm will miss its targets.

## Concepts

### Camera intrinsic parameters

Intrinsic parameters describe the camera's internal optical properties:

| Parameter | Description |
|-----------|-------------|
| `fx` | Focal length in the x direction (pixels) |
| `fy` | Focal length in the y direction (pixels) |
| `ppx` | Principal point x coordinate (pixels) -- the optical center |
| `ppy` | Principal point y coordinate (pixels) -- the optical center |
| `width_px` | Image width in pixels |
| `height_px` | Image height in pixels |

The focal length determines how much the camera "zooms in" on the scene. The
principal point is where the optical axis intersects the image sensor. For a
perfect camera, the principal point would be at the exact center of the image,
but manufacturing tolerances mean it is usually slightly off-center.

### Distortion parameters

Real camera lenses bend light imperfectly, causing straight lines in the world
to appear curved in the image. Distortion parameters model this bending so
software can correct for it:

| Parameter | Description |
|-----------|-------------|
| `rk1` | First radial distortion coefficient |
| `rk2` | Second radial distortion coefficient |
| `rk3` | Third radial distortion coefficient |
| `tp1` | First tangential distortion coefficient |
| `tp2` | Second tangential distortion coefficient |

Radial distortion causes barrel or pincushion effects (straight lines bowing
outward or inward). Tangential distortion occurs when the lens is not perfectly
parallel to the sensor. For most cameras, radial distortion dominates and
tangential distortion is small.

### Why calibration matters

Accurate intrinsic parameters are required for:

- **2D-to-3D projection**: converting a pixel coordinate and depth value into a
  3D position in space
- **Depth estimation**: computing distances from stereo camera pairs
- **Visual servoing**: using camera feedback to guide robot motion in real time
- **Point cloud generation**: projecting depth images into 3D point clouds

If the intrinsic parameters are wrong by even a few percent, 3D position errors
compound with distance. An object 1 meter away might appear 5 cm off. At 3
meters, the error grows to 15 cm.

### Eye-in-hand vs eye-to-hand

Camera mounting affects how you configure the camera's frame:

- **Eye-in-hand**: the camera is mounted on the robot arm. Its frame parent is
  the arm. When the arm moves, the camera moves with it. This setup is common
  for close-range inspection and manipulation tasks.
- **Eye-to-hand**: the camera is mounted on a fixed structure (a pole, a wall,
  or overhead). Its frame parent is the world frame. The camera stays
  stationary while the arm moves. This setup provides a broader view of the
  workspace.

The calibration process (computing intrinsics and distortion) is the same for
both setups. Only the frame configuration differs.

### The chessboard method

Camera calibration uses images of a known pattern -- typically a chessboard
printed on paper and mounted on a flat, rigid surface. The calibration algorithm
detects the corners of the chessboard squares, compares their detected pixel
positions to their known physical positions, and solves for the camera parameters
that best explain the observed projections.

For good calibration results, you need 10-15 images of the chessboard from
different angles, distances, and positions within the camera's field of view.

## Steps

### 1. Print a calibration target

Print a standard chessboard calibration pattern. You can find printable patterns
online by searching for "camera calibration chessboard PDF." Use a pattern with
at least 8x6 inner corners.

Mount the printed pattern on a flat, rigid surface:

- **Good:** acrylic plate, rigid cardboard, flat wooden board
- **Bad:** holding the paper by hand (it will curl), taping it to a textured
  surface (the surface may warp the pattern)

Ensure the printed squares are exactly the specified size. Measure a few squares
with a ruler to confirm. Common square sizes are 20 mm or 25 mm per side.

### 2. Capture calibration images

Take 10-15 images of the chessboard from various positions and angles:

1. Go to [app.viam.com](https://app.viam.com) and navigate to your machine.
2. Click the **CONTROL** tab and find your camera.
3. Position the chessboard in the camera's field of view.
4. Click the **Export screenshot** button to save an image.
5. Repeat from different angles and distances.

Follow these guidelines for good calibration images:

- **Cover the entire field of view.** Take images with the chessboard in the
  center, top-left, top-right, bottom-left, and bottom-right of the frame.
- **Vary the distance.** Take images at the closest and farthest working
  distances you expect to use.
- **Tilt the chessboard.** Tilt it 15-30 degrees in different directions. This
  helps the algorithm separate focal length from distance effects.
- **Show the full chessboard.** Every image must include all corners of the
  chessboard pattern. If any corners are cut off, the image cannot be used.
- **Ensure good lighting.** Avoid shadows on the chessboard and avoid
  overexposure (bright glare).
- **Hold still.** The image must be sharp, not blurry from motion.

Save all images to a single directory on your computer.

### 3. Run the calibration script

Install the required Python packages and run the calibration:

```sh
pip3 install numpy opencv-python
```

Download or create the calibration script (`cameraCalib.py`), then run it on
your image directory:

```sh
python3 cameraCalib.py YOUR_PICTURES_DIRECTORY
```

The script detects chessboard corners in each image, computes the camera matrix
and distortion coefficients, and outputs the results. A successful calibration
produces output like this:

```json
{
  "intrinsic_parameters": {
    "fx": 939.27,
    "fy": 940.29,
    "ppx": 320.61,
    "ppy": 239.14,
    "width_px": 640,
    "height_px": 480
  },
  "distortion_parameters": {
    "rk1": 0.0465,
    "rk2": 0.8003,
    "rk3": -5.408,
    "tp1": -0.000009,
    "tp2": -0.002829
  }
}
```

The reprojection error should be less than 1.0 pixel. If it is above 2.0,
your calibration is poor -- retake images following the guidelines in step 2.

### 4. Add parameters to camera config

Copy the calibration output into your camera's configuration:

1. Go to [app.viam.com](https://app.viam.com) and navigate to your machine.
2. Click the **CONFIGURE** tab.
3. Find your camera component in the component list.
4. Add the `intrinsic_parameters` and `distortion_parameters` to the camera's
   attributes.

The full camera configuration with calibration parameters looks like this:

```json
{
  "name": "my-camera",
  "api": "rdk:component:camera",
  "model": "webcam",
  "attributes": {
    "video_path": "video0",
    "width_px": 640,
    "height_px": 480,
    "intrinsic_parameters": {
      "fx": 939.27,
      "fy": 940.29,
      "ppx": 320.61,
      "ppy": 239.14,
      "width_px": 640,
      "height_px": 480
    },
    "distortion_parameters": {
      "rk1": 0.0465,
      "rk2": 0.8003,
      "rk3": -5.408,
      "tp1": -0.000009,
      "tp2": -0.002829
    }
  }
}
```

Click **Save** to apply the configuration. The camera will immediately use the
new parameters for any operation that requires intrinsics (depth projection,
point cloud generation, etc.).

### 5. Configure the camera frame

Add a frame to the camera that describes its physical position and orientation
relative to the robot. The frame configuration depends on how the camera is
mounted.

**Eye-in-hand (camera mounted on the arm):**

Set the parent to the arm component. Measure the translation from the arm's end
effector origin to the camera's optical center.

```json
{
  "parent": "my-arm",
  "translation": { "x": 50, "y": 0, "z": 80 },
  "orientation": {
    "type": "ov_degrees",
    "value": { "x": 0, "y": 1, "z": 0, "th": -30 }
  }
}
```

In this example, the camera is mounted 50 mm to the right and 80 mm above the
arm's end effector, tilted 30 degrees downward around the y axis.

**Eye-to-hand (camera on a fixed mount):**

Set the parent to the world frame. Measure the translation from your world frame
origin to the camera's position.

```json
{
  "parent": "world",
  "translation": { "x": 500, "y": 300, "z": 800 },
  "orientation": {
    "type": "ov_degrees",
    "value": { "x": 0, "y": 0, "z": 1, "th": 180 }
  }
}
```

In this example, the camera is mounted 500 mm to the right, 300 mm forward, and
800 mm above the world frame origin, rotated 180 degrees (pointing back toward
the workspace).

Click **Save** after adding the frame configuration.

### 6. Verify calibration accuracy

Test your calibration by checking if the camera produces accurate 3D positions
for objects at known locations.

1. Place an object at a measured position in your workspace (for example, 400 mm
   to the right and 300 mm forward from the world frame origin).
2. Use a vision service to detect the object, or use the depth camera to get a
   point cloud.
3. Use `TransformPose` to convert the detected position from the camera frame
   to the world frame.
4. Compare the computed position to the measured position.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.common import PoseInFrame, Pose

# Suppose the vision service detected an object at these coordinates
# in the camera's frame (from a 3D segmenter or depth projection)
detected_in_camera = PoseInFrame(
    reference_frame="my-camera",
    pose=Pose(x=50, y=30, z=400)
)

# Transform to world frame
detected_in_world = await machine.transform_pose(detected_in_camera, "world")
print(f"Detected position in world frame:")
print(f"  x={detected_in_world.pose.x:.1f} mm")
print(f"  y={detected_in_world.pose.y:.1f} mm")
print(f"  z={detected_in_world.pose.z:.1f} mm")
print("Compare these values to your physical measurements.")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
detectedInCamera := referenceframe.NewPoseInFrame("my-camera",
    spatialmath.NewPoseFromPoint(r3.Vector{X: 50, Y: 30, Z: 400}))

detectedInWorld, err := machine.TransformPose(ctx, detectedInCamera, "world", nil)
if err != nil {
    logger.Fatal(err)
}

pt := detectedInWorld.Pose().Point()
fmt.Printf("Detected position in world frame:\n")
fmt.Printf("  x=%.1f mm\n", pt.X)
fmt.Printf("  y=%.1f mm\n", pt.Y)
fmt.Printf("  z=%.1f mm\n", pt.Z)
fmt.Println("Compare these values to your physical measurements.")
```

{{% /tab %}}
{{< /tabs >}}

If the computed position is within 10-20 mm of the measured position at a
distance of 500-1000 mm, your calibration is good. Larger errors indicate a
problem with the intrinsic parameters, the frame configuration, or both.

### 7. Fine-tune the frame offset

If the 3D positions are consistently off by a fixed amount, adjust the camera
frame's translation values:

1. Place an object at a known position and measure the error in each axis.
2. Adjust the translation in the camera's frame config by the measured error.
3. Save and re-test.

For rotational errors (object appears shifted more at farther distances),
adjust the orientation values. A small angular error in the frame configuration
compounds with distance.

Repeat the place-and-measure cycle until the error is within your application's
tolerance.

### 8. Run a multi-point accuracy check

For a thorough verification, test calibration accuracy at several known points
across your workspace. This reveals systematic errors that a single-point check
might miss.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.common import PoseInFrame, Pose
import math

# Define known test points (measured positions in world frame, in mm)
test_points = [
    {"name": "near-center", "world": (200, 150, 0), "camera": (30, 20, 350)},
    {"name": "far-left",    "world": (-100, 300, 0), "camera": (-80, 10, 550)},
    {"name": "far-right",   "world": (400, 300, 0),  "camera": (120, 15, 520)},
    {"name": "close-up",    "world": (150, 50, 0),   "camera": (10, 40, 200)},
]

for point in test_points:
    detected = PoseInFrame(
        reference_frame="my-camera",
        pose=Pose(x=point["camera"][0],
                  y=point["camera"][1],
                  z=point["camera"][2])
    )
    result = await machine.transform_pose(detected, "world")

    ex = result.pose.x - point["world"][0]
    ey = result.pose.y - point["world"][1]
    ez = result.pose.z - point["world"][2]
    error_mm = math.sqrt(ex**2 + ey**2 + ez**2)

    print(f"{point['name']}: error = {error_mm:.1f} mm "
          f"(dx={ex:.1f}, dy={ey:.1f}, dz={ez:.1f})")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
type testPoint struct {
    name    string
    worldX, worldY, worldZ    float64
    cameraX, cameraY, cameraZ float64
}

testPoints := []testPoint{
    {"near-center", 200, 150, 0, 30, 20, 350},
    {"far-left", -100, 300, 0, -80, 10, 550},
    {"far-right", 400, 300, 0, 120, 15, 520},
    {"close-up", 150, 50, 0, 10, 40, 200},
}

for _, tp := range testPoints {
    detected := referenceframe.NewPoseInFrame("my-camera",
        spatialmath.NewPoseFromPoint(
            r3.Vector{X: tp.cameraX, Y: tp.cameraY, Z: tp.cameraZ}))

    result, err := machine.TransformPose(ctx, detected, "world", nil)
    if err != nil {
        logger.Fatal(err)
    }

    pt := result.Pose().Point()
    ex := pt.X - tp.worldX
    ey := pt.Y - tp.worldY
    ez := pt.Z - tp.worldZ
    errorMM := math.Sqrt(ex*ex + ey*ey + ez*ez)

    fmt.Printf("%s: error = %.1f mm (dx=%.1f, dy=%.1f, dz=%.1f)\n",
        tp.name, errorMM, ex, ey, ez)
}
```

{{% /tab %}}
{{< /tabs >}}

If errors are consistent in one direction across all points, adjust the
translation offset. If errors grow with distance, adjust the orientation. If
errors are random and large, re-run the calibration with better images.

## Try It

1. Print a chessboard target and capture 10-15 calibration images from your
   camera using the CONTROL tab.
2. Run the calibration script and record the intrinsic and distortion
   parameters.
3. Add the parameters to your camera configuration and save.
4. Configure the camera frame (eye-in-hand or eye-to-hand) with measured
   translation and orientation values.
5. Place an object at a known position and verify the calibration accuracy
   using `TransformPose` (step 6). Confirm the error is within 20 mm at a
   working distance of 500-1000 mm.

## Troubleshooting

{{< expand "Calibration script fails to find chessboard corners" >}}

- Verify the chessboard is fully visible in every image. All inner corners must
  be within the frame.
- Check the lighting. Shadows or glare on the chessboard prevent corner
  detection.
- Ensure the chessboard is flat. A curled or bent printout produces inaccurate
  corners or prevents detection entirely.
- Verify the chessboard dimensions match the script's expected pattern size. If
  the script expects 9x7 inner corners but your board has 8x6, it will not find
  corners.

{{< /expand >}}

{{< expand "Calibration parameters look wrong (very large or very small values)" >}}

- Re-run calibration with more images (at least 15) from a wider variety of
  angles and distances.
- Remove any images where the chessboard is blurry, partially occluded, or
  poorly lit.
- Verify the chessboard square size parameter is correct. The calibration script
  needs the physical size of each square (in mm) to produce correct focal length
  values.

{{< /expand >}}

{{< expand "3D positions are consistently offset" >}}

- Check the camera frame translation. Measure the physical offset from the
  parent frame origin to the camera lens center and update the translation
  values.
- Check the camera frame orientation. If the camera is tilted and the
  orientation does not reflect this, all positions will be systematically
  shifted.
- Verify the parent frame. If the camera frame parent is set to `"world"` but
  the camera is mounted on the arm, the frame will not move with the arm and
  all positions will be wrong when the arm moves.

{{< /expand >}}

{{< expand "Calibration accuracy varies with distance" >}}

- This is expected to some degree. Depth errors grow with distance for both
  stereo and structured-light depth cameras.
- If accuracy is poor at close range, re-run calibration with more close-range
  images.
- If accuracy is poor at far range, the depth camera may be at the edge of its
  operating range. Check the camera's specifications for supported depth range.

{{< /expand >}}

## What's Next

- [Define Obstacles](/work-cell-layout/define-obstacles/) -- add collision geometry to your
  workspace so the motion planner avoids physical objects.
- [Localize Objects in 3D](/vision-detection/localize-objects-in-3d/) --
  combine 2D detections with depth data to get real-world 3D object positions.
- [Pick an Object](/build/arm-manipulation/pick-an-object/) -- use calibrated
  camera data and 3D positions to plan and execute grasps with a robot arm.
