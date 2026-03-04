---
linkTitle: "Localize Objects in 3D"
title: "Localize Objects in 3D"
weight: 60
layout: "docs"
type: "docs"
description: "Use the detector_3d_segmenter to project 2D detections into 3D point clouds and retrieve real-world object positions."
date: "2025-01-30"
---

## What Problem This Solves

A 2D detection tells you that there is a cup at pixel coordinates (300, 200)-(400, 350). But if your robot arm needs to pick up that cup, it needs to know the cup's position in 3D space. The `detector_3d_segmenter` automates this conversion by combining 2D detections with depth data to produce 3D point clouds for each detected object.

## Concepts

### From 2D to 3D

A 2D detection gives you a bounding box in pixel space. To get the 3D position of the object, you need two additional pieces of information:

1. **Depth data** from a depth camera, which provides the distance from the camera to the surface at each pixel.
2. **Camera intrinsic parameters** (focal length, principal point), which define how the camera projects 3D space onto the 2D image plane.

With these three inputs (bounding box, depth, intrinsics), the segmenter can project each pixel within the bounding box into 3D space, producing a point cloud that represents the physical shape and location of the detected object.

### The detector_3d_segmenter

The `detector_3d_segmenter` is a vision service model built into Viam. It works as a pipeline:

1. Runs your 2D detector to get bounding boxes.
2. For each bounding box, extracts the corresponding region from the depth image.
3. Projects those depth pixels into 3D points using the camera's intrinsic parameters.
4. Filters noise from the resulting point cloud.
5. Returns the cleaned point cloud with the detection label attached.

You configure it as a vision service that references your existing detector and depth camera. You do not need to write any projection code yourself.

### Noise filtering with mean_k and sigma

Depth data is noisy. Pixels at object edges, on reflective surfaces, or at extreme distances often have incorrect depth values. If these noisy points are included in the object's point cloud, the 3D position estimate will be inaccurate.

The segmenter uses statistical outlier removal to clean the point cloud:

- **`mean_k`**: The number of nearest neighbors to consider when evaluating each point. A higher value considers more neighbors, making the filter more aggressive.
- **`sigma`**: The standard deviation threshold. Points whose distance to their neighbors exceeds this threshold are removed. A lower value removes more points.

Default values of `mean_k=50` and `sigma=2.0` work well for most indoor scenes. Increase `mean_k` or decrease `sigma` if you see noisy points. Decrease `mean_k` or increase `sigma` if too many valid points are being removed.

### Object point clouds

The `GetObjectPointClouds` API returns a list of objects. Each object contains:

- A **label** (the class name from the 2D detector)
- A **point cloud** (the 3D points belonging to this object)
- A **geometry** (a bounding shape, such as a box, enclosing the point cloud)

The point cloud coordinates are in the camera's coordinate frame: x is right, y is down, z is forward (away from the camera), all in millimeters.

### Frame system integration

The 3D positions returned by the segmenter are in the camera's coordinate frame. If your robot has an arm, a base, or other components, you need to transform these positions into the appropriate coordinate frame before using them for navigation or manipulation.

Viam's frame system handles these transformations. Once your camera is registered in the frame system with its position and orientation relative to the robot, you can transform any point from camera coordinates to robot coordinates (or any other frame) automatically.

## Steps

### 1. Configure the detector_3d_segmenter

Add a new vision service that uses the `detector_3d_segmenter` model.

1. Go to [app.viam.com](https://app.viam.com) and navigate to your machine.
2. Click the **+** button in the left sidebar.
3. Select **Service**, then **vision**.
4. Choose **detector_3d_segmenter** as the model.
5. Name it `my-segmenter`.
6. Click **Create**.

### 2. Configure segmenter attributes

Set the segmenter to use your existing detector and depth camera:

```json
{
  "name": "my-segmenter",
  "api": "rdk:service:vision",
  "model": "detector_3d_segmenter",
  "attributes": {
    "detector_name": "my-detector",
    "confidence_threshold_pct": 0.5,
    "mean_k": 50,
    "sigma": 2.0
  }
}
```

| Attribute | Description |
|-----------|-------------|
| `detector_name` | Name of the 2D detector vision service to use |
| `confidence_threshold_pct` | Minimum confidence (0.0-1.0) for a detection to be projected to 3D |
| `mean_k` | Number of neighbors for statistical outlier removal |
| `sigma` | Standard deviation threshold for outlier removal |

The `detector_name` must match the name of an existing vision service configured with an `mlmodel` or `color_detector` model.

### 3. Save and verify

Click **Save**. The segmenter initializes and is ready to use. You can verify it is working by checking the `viam-server` logs for any error messages.

### 4. Get 3D object point clouds

Use the `GetObjectPointClouds` API to retrieve 3D objects from the segmenter.

{{< tabs >}}
{{% tab name="Python" %}}

```python
import asyncio
from viam.robot.client import RobotClient
from viam.services.vision import VisionClient


async def main():
    opts = RobotClient.Options.with_api_key(
        api_key="YOUR-API-KEY",
        api_key_id="YOUR-API-KEY-ID"
    )
    robot = await RobotClient.at_address("YOUR-MACHINE-ADDRESS", opts)

    segmenter = VisionClient.from_robot(robot, "my-segmenter")

    # Get 3D object point clouds
    objects = await segmenter.get_object_point_clouds("my-depth-camera")

    print(f"Found {len(objects)} objects in 3D:")
    for obj in objects:
        print(f"  Object: {obj.geometry.label}")
        center = obj.geometry.center
        print(f"    Center: x={center.x:.0f} y={center.y:.0f} "
              f"z={center.z:.0f} mm")

    await robot.close()

if __name__ == "__main__":
    asyncio.run(main())
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
package main

import (
	"context"
	"fmt"

	"go.viam.com/rdk/logging"
	"go.viam.com/rdk/robot/client"
	"go.viam.com/rdk/services/vision"
	"go.viam.com/rdk/utils"
)

func main() {
	ctx := context.Background()
	logger := logging.NewLogger("localize")

	robot, err := client.New(ctx, "YOUR-MACHINE-ADDRESS", logger,
		client.WithCredentials(utils.Credentials{
			Type:    utils.CredentialsTypeAPIKey,
			Payload: "YOUR-API-KEY",
		}),
		client.WithAPIKeyID("YOUR-API-KEY-ID"),
	)
	if err != nil {
		logger.Fatal(err)
	}
	defer robot.Close(ctx)

	segmenter, err := vision.FromRobot(robot, "my-segmenter")
	if err != nil {
		logger.Fatal(err)
	}

	// Get 3D object point clouds
	objects, err := segmenter.GetObjectPointClouds(
		ctx, "my-depth-camera", nil,
	)
	if err != nil {
		logger.Fatal(err)
	}

	fmt.Printf("Found %d objects in 3D:\n", len(objects))
	for _, obj := range objects {
		label := obj.Geometry.Label()
		center := obj.Geometry.Pose().Point()
		fmt.Printf("  Object: %s\n", label)
		fmt.Printf("    Center: x=%.0f y=%.0f z=%.0f mm\n",
			center.X, center.Y, center.Z)
	}
}
```

{{% /tab %}}
{{< /tabs >}}

### 5. Extract distance and position information

Once you have 3D objects, you can calculate distances and positions for practical use.

{{< tabs >}}
{{% tab name="Python" %}}

```python
import math
from viam.services.vision import VisionClient


segmenter = VisionClient.from_robot(robot, "my-segmenter")
objects = await segmenter.get_object_point_clouds("my-depth-camera")

for obj in objects:
    center = obj.geometry.center

    # Euclidean distance from camera to object center
    distance_mm = math.sqrt(
        center.x**2 + center.y**2 + center.z**2
    )
    distance_m = distance_mm / 1000.0

    # Forward distance (depth only, ignoring lateral offset)
    forward_m = center.z / 1000.0

    # Lateral offset (positive = right of camera center)
    lateral_m = center.x / 1000.0

    print(f"{obj.geometry.label}:")
    print(f"  Euclidean distance: {distance_m:.2f} m")
    print(f"  Forward distance:   {forward_m:.2f} m")
    print(f"  Lateral offset:     {lateral_m:.2f} m")
    print(f"  Vertical offset:    {center.y / 1000.0:.2f} m")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
import (
    "fmt"
    "math"

    "go.viam.com/rdk/services/vision"
)

segmenter, err := vision.FromRobot(robot, "my-segmenter")
if err != nil {
    logger.Fatal(err)
}

objects, err := segmenter.GetObjectPointClouds(
    ctx, "my-depth-camera", nil,
)
if err != nil {
    logger.Fatal(err)
}

for _, obj := range objects {
    center := obj.Geometry.Pose().Point()

    // Euclidean distance from camera to object center
    distanceMM := math.Sqrt(
        center.X*center.X + center.Y*center.Y + center.Z*center.Z,
    )
    distanceM := distanceMM / 1000.0

    // Forward distance (depth only)
    forwardM := center.Z / 1000.0

    // Lateral offset
    lateralM := center.X / 1000.0

    fmt.Printf("%s:\n", obj.Geometry.Label())
    fmt.Printf("  Euclidean distance: %.2f m\n", distanceM)
    fmt.Printf("  Forward distance:   %.2f m\n", forwardM)
    fmt.Printf("  Lateral offset:     %.2f m\n", lateralM)
    fmt.Printf("  Vertical offset:    %.2f m\n", center.Y/1000.0)
}
```

{{% /tab %}}
{{< /tabs >}}

### 6. Monitor objects continuously

Run the segmenter in a loop to track 3D positions over time.

{{< tabs >}}
{{% tab name="Python" %}}

```python
import asyncio
import math
from viam.services.vision import VisionClient


segmenter = VisionClient.from_robot(robot, "my-segmenter")

while True:
    objects = await segmenter.get_object_point_clouds("my-depth-camera")

    if objects:
        for obj in objects:
            center = obj.geometry.center
            dist = math.sqrt(
                center.x**2 + center.y**2 + center.z**2
            ) / 1000.0
            print(f"{obj.geometry.label}: {dist:.2f}m away "
                  f"at ({center.x:.0f}, {center.y:.0f}, {center.z:.0f}) mm")
    else:
        print("No objects detected")

    await asyncio.sleep(0.5)
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
segmenter, err := vision.FromRobot(robot, "my-segmenter")
if err != nil {
    logger.Fatal(err)
}

for {
    objects, err := segmenter.GetObjectPointClouds(
        ctx, "my-depth-camera", nil,
    )
    if err != nil {
        logger.Error(err)
        time.Sleep(time.Second)
        continue
    }

    if len(objects) > 0 {
        for _, obj := range objects {
            center := obj.Geometry.Pose().Point()
            dist := math.Sqrt(
                center.X*center.X +
                center.Y*center.Y +
                center.Z*center.Z,
            ) / 1000.0
            fmt.Printf("%s: %.2fm away at (%.0f, %.0f, %.0f) mm\n",
                obj.Geometry.Label(), dist,
                center.X, center.Y, center.Z)
        }
    } else {
        fmt.Println("No objects detected")
    }

    time.Sleep(500 * time.Millisecond)
}
```

{{% /tab %}}
{{< /tabs >}}

### 7. Tune noise filtering

Adjust `mean_k` and `sigma` based on your environment:

| Scenario | `mean_k` | `sigma` | Why |
|----------|----------|---------|-----|
| Clean indoor scene, close range | 30 | 2.5 | Less filtering needed, preserve detail |
| Cluttered scene, multiple objects | 50 | 2.0 | Moderate filtering |
| Noisy depth (reflective surfaces) | 80 | 1.5 | Aggressive filtering to remove outliers |
| Outdoor or long range | 100 | 1.0 | Very aggressive filtering |

Update the values in your segmenter configuration and save. The changes take effect immediately without restarting `viam-server`.

### 8. Understand coordinate frames

The 3D positions returned by the segmenter are in the camera's coordinate frame. To use these positions with other parts of your robot (an arm, a base), you need to understand the relationship between coordinate frames.

If your camera is mounted on a robot arm at a known position and orientation, the frame system can transform object positions from camera coordinates to arm-base coordinates or world coordinates. This is configured in the frame system, not in the segmenter itself.

For example, if the camera reports an object at (100, 50, 500) mm in camera coordinates, and the camera is mounted 300 mm above the arm base pointing 30 degrees downward, the frame system computes the object's position in the arm's coordinate frame automatically.

See [Define Your Frame System](/build/work-cell-layout/define-your-frame-system/) for how to set up these transforms.

## Try It

1. Run the basic 3D localization script from step 4. Place objects at known distances from the camera and verify the reported positions are reasonable.
2. Move an object closer and farther from the camera. Verify that the z-coordinate (forward distance) changes proportionally.
3. Move an object left and right. Verify that the x-coordinate (lateral offset) changes in the expected direction.
4. Try different `mean_k` and `sigma` values from step 7. Observe how the point cloud changes -- fewer points with aggressive filtering, more noise with lenient filtering.
5. If you have a reflective or transparent object, observe that the segmenter may produce inaccurate results or no results for that object.

## Troubleshooting

{{< expand "No objects returned even though 2D detections work" >}}

- Verify the depth camera is providing depth data. Run the depth measurement test from [Measure Depth](/build/vision-detection/measure-depth/) to confirm.
- The `confidence_threshold_pct` may be filtering out all detections. Lower it to 0.3 and try again.
- Check that the `detector_name` in the segmenter configuration matches the name of your 2D detector exactly.

{{< /expand >}}

{{< expand "Object positions are wildly inaccurate" >}}

- The camera's intrinsic parameters may be incorrect or missing. Check the camera configuration and verify that `fx`, `fy`, `ppx`, and `ppy` are set correctly. See [Measure Depth](/build/vision-detection/measure-depth/) step 7 for configuration details.
- Depth cameras are most accurate at ranges of 0.5-4 meters. Objects outside this range may have large position errors.
- If the camera is not calibrated, all 3D projections will be systematically wrong. Use the camera manufacturer's calibration tool or Viam's intrinsic parameter configuration.

{{< /expand >}}

{{< expand "Point clouds are very noisy" >}}

- Increase `mean_k` and decrease `sigma` to filter more aggressively.
- Ensure adequate lighting. Depth cameras (especially stereo-based ones) perform poorly in low light.
- Avoid pointing the camera at reflective or transparent surfaces.

{{< /expand >}}

{{< expand "Segmenter is slow" >}}

- The 3D segmentation pipeline involves running the 2D detector, capturing a depth image, projecting pixels to 3D, and filtering noise. This is inherently slower than 2D detection alone.
- Reduce the camera resolution to speed up the pipeline. A 320x240 depth image processes significantly faster than 640x480.
- Reduce `mean_k` to speed up the noise filtering step, at the cost of noisier results.

{{< /expand >}}

{{< expand "Frame system transforms give unexpected results" >}}

- Double-check the camera's position and orientation in the frame system configuration. A common mistake is getting the rotation direction wrong (e.g., specifying a 30-degree downward tilt as a 30-degree upward tilt).
- Verify units. The frame system uses millimeters for positions and degrees for rotations.
- Use the Viam app's frame system visualizer to verify your configuration visually.

{{< /expand >}}

## What's Next

- [Define Your Frame System](/build/work-cell-layout/define-your-frame-system/) -- set up coordinate frame transforms so 3D positions are usable by arms, bases, and other components.
- [Pick an Object](/build/arm-manipulation/pick-an-object/) -- use 3D object positions to plan and execute grasp motions with a robot arm.
- [Detect While Moving](/build/mobile-vision/detect-while-moving/) -- run 3D localization while the robot is in motion for dynamic obstacle avoidance.
