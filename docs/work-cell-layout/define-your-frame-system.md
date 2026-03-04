---
linkTitle: "Define Your Frame System"
title: "Define Your Frame System"
weight: 10
layout: "docs"
type: "docs"
description: "Configure a unified coordinate tree so all components share a consistent spatial model."
date: "2025-01-30"
aliases:
  - /build/work-cell-layout/define-your-frame-system/
---

## What Problem This Solves

When your machine has multiple components -- an arm, a camera, a gripper, sensors -- each reports positions in its own local coordinate system.
The frame system stores the position and orientation of every component in a single, consistent coordinate tree, letting you convert between any two frames with a single API call.
Getting the frame system right is a prerequisite for motion planning, obstacle avoidance, and any task where multiple components need to agree on where things are in physical space.

## Concepts

### What the frame system is

The frame system is a tree of coordinate frames. Each component on your machine
can have a frame that describes its position and orientation. The frame system
tracks how all these frames relate to each other, so you can convert a position
from one component's coordinate system to another.

Every frame has a parent. The default parent is the world frame, which sits at
the root of the tree. You build the tree by setting each component's frame
parent to either the world frame or another component's frame.

### The world frame

The world frame is the fixed root reference point for your entire frame system.
You choose what physical location it corresponds to -- the corner of a table,
the center of a work surface, the base of an arm. You do not explicitly
configure the world frame itself. Instead, you define it implicitly by
configuring other frames relative to it.

Pick a point that is easy to measure from and that will not move. For a
table-mounted arm, the arm base or a table corner are good choices. For a mobile
robot, the center of the base is typical.

### Parent-child hierarchy

Each component's frame has a parent frame. The parent defaults to the world
frame if you do not specify one. When you attach a camera to an arm, you set the
camera's frame parent to the arm. This means the camera's position is defined
relative to the arm's end, and when the arm moves, the camera frame moves with
it automatically.

The hierarchy forms a tree rooted at the world frame:

```
world
├── my-arm
│   └── my-camera (mounted on arm)
├── my-sensor (mounted on table)
└── table-surface
```

### Translation

Translation is the offset in millimeters from the parent frame's origin to the
component's origin. It is specified as an (x, y, z) vector:

- **x**: right/left offset
- **y**: forward/backward offset
- **z**: up/down offset

The exact meaning of each axis depends on the parent frame's orientation. If the
parent is the world frame with the standard orientation, +z points up.

### Orientation

Orientation describes how a component's axes are rotated relative to its parent
frame. Viam supports several orientation formats:

| Type | Fields | Description |
|------|--------|-------------|
| `ov_degrees` | `x, y, z, th` | Orientation vector (axis) with angle in degrees |
| `ov_radians` | `x, y, z, th` | Orientation vector (axis) with angle in radians |
| `euler_angles` | `roll, pitch, yaw` | Rotation around x, y, z axes |
| `quaternion` | `w, x, y, z` | Unit quaternion |

The default orientation is `ov_degrees` with values `(0, 0, 1), 0`, which means
the component's axes are aligned with its parent frame. The orientation vector
(x, y, z) defines the axis of rotation, and `th` defines the rotation angle
around that axis.

### Geometry

Each frame can optionally include collision geometry -- a simple shape that
approximates the component's physical footprint. The motion planner uses these
shapes to avoid collisions. Supported types:

- **box**: defined by x, y, z dimensions in mm
- **sphere**: defined by a radius in mm
- **capsule**: defined by a radius and length in mm

Geometry is centered on the frame's origin by default. You can add a translation
offset to shift the geometry relative to the frame.

## Steps

### 1. Choose your world frame

Pick a fixed, easy-to-measure point in your workspace. Good choices:

- **Table-mounted arm:** the arm base or a table corner
- **Mobile robot:** the center of the base
- **Work cell:** a corner of the work surface

You do not configure the world frame explicitly. It exists automatically as the
root of the frame tree. All other frames reference it (directly or through
parent chains).

Write down your choice and mark the physical location. All measurements you take
in the following steps are relative to this point.

### 2. Add a frame to your arm

In the Viam app:

1. Go to [app.viam.com](https://app.viam.com) and navigate to your machine.
2. Click the **CONFIGURE** tab.
3. Find your arm component in the component list.
4. Click **+ Add frame** in the arm's configuration panel.

The default frame configuration places the arm's origin at the world frame
origin with no rotation:

```json
{
  "parent": "world",
  "translation": { "x": 0, "y": 0, "z": 0 },
  "orientation": {
    "type": "ov_degrees",
    "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
  }
}
```

Here is what each field means:

- **`parent`**: the frame this component is positioned relative to. `"world"` is
  the root frame.
- **`translation`**: the offset in millimeters from the parent frame's origin to
  this component's origin. `(0, 0, 0)` means the origins coincide.
- **`orientation.type`**: the rotation format. `"ov_degrees"` uses an orientation
  vector with the angle in degrees.
- **`orientation.value`**: the rotation itself. `(0, 0, 1)` is the rotation
  axis (pointing along +z), and `th: 0` means zero rotation. This is the
  identity orientation -- no rotation from the parent frame.

Click **Save** after adding the frame.

### 3. Figure out which way the axes point

Before you set offsets, verify that the arm's coordinate axes match your
intended world frame directions.

1. Go to the **CONTROL** tab and find your arm.
2. Use the arm's **TEST** panel to jog the end effector in small increments.
3. Command a move in the +z direction and observe which way the arm moves
   physically. If +z moves the arm up, the z axis matches the standard
   convention.
4. Repeat for +x and +y to confirm all three axes.

If the directions do not match your intended world frame orientation, adjust the
`orientation` field. For example, if the arm's +x points in the opposite
direction from your intended +x, set `th: 180` with the z axis as the rotation
axis:

```json
{
  "orientation": {
    "type": "ov_degrees",
    "value": { "x": 0, "y": 0, "z": 1, "th": 180 }
  }
}
```

This rotates the arm's frame 180 degrees around the z axis, flipping the x and
y axes.

### 4. Add a frame offset

If your world frame is not at the arm's base (for example, the world frame is at
a table corner), measure the distance from the world frame origin to the arm
base along each axis.

For an arm base that sits 100 mm to the right and 200 mm forward from the table
corner:

```json
{
  "parent": "world",
  "translation": { "x": 100, "y": 200, "z": 0 },
  "orientation": {
    "type": "ov_degrees",
    "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
  }
}
```

Use a tape measure for these values. Precision to the nearest millimeter is
sufficient for most applications. If the arm base is elevated above the world
frame origin, set the z translation accordingly.

### 5. Add frames for other components

Each component that participates in spatial reasoning needs a frame. Add frames
for cameras, grippers, sensors, and other components.

The parent of each frame determines what it is positioned relative to:

- A camera bolted to the table: parent = `"world"`
- A camera mounted on the arm: parent = `"my-arm"`
- A gripper attached to the arm: parent = `"my-arm"`

**Example: a camera mounted on the arm**, offset 50 mm in y and 100 mm in z from
the arm's end effector:

```json
{
  "parent": "my-arm",
  "translation": { "x": 0, "y": 50, "z": 100 },
  "orientation": {
    "type": "ov_degrees",
    "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
  }
}
```

**Example: a sensor mounted on the table**, 300 mm to the right and 400 mm
forward from the world frame origin:

```json
{
  "parent": "world",
  "translation": { "x": 300, "y": 400, "z": 0 },
  "orientation": {
    "type": "ov_degrees",
    "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
  }
}
```

If the camera is tilted (for example, angled 30 degrees downward), set the
orientation to reflect this. A 30-degree tilt around the x axis:

```json
{
  "orientation": {
    "type": "ov_degrees",
    "value": { "x": 1, "y": 0, "z": 0, "th": -30 }
  }
}
```

Add geometry to each frame if you want the motion planner to account for the
component's physical size. See [Define Obstacles](/work-cell-layout/define-obstacles/) for
details on geometry configuration.

### 6. Visualize the frame system

Use the Viam app to verify your configuration visually:

1. Navigate to your machine in the Viam app.
2. Click the **VISUALIZE** tab.
3. The viewer renders all configured frames in 3D space. Each frame appears as a
   set of colored axes (red = x, green = y, blue = z).
4. Verify that each frame's position and orientation match the physical layout of
   your workspace.

If a frame appears in the wrong position, go back to the **CONFIGURE** tab and
adjust the translation values. If axes point the wrong way, adjust the
orientation.

### 7. Query the frame system programmatically

Use the robot client API to read frame configurations and transform poses
between frames.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.common import PoseInFrame, Pose

# Get the current frame system configuration
frame_system = await machine.get_frame_system_config()
for frame in frame_system:
    print(f"Frame: {frame.frame.reference_frame}, "
          f"Parent: {frame.frame.parent_reference_frame}")

# Transform a pose from one frame to another
pose_in_camera = PoseInFrame(
    reference_frame="my-camera",
    pose=Pose(x=100, y=0, z=200)
)
pose_in_world = await machine.transform_pose(pose_in_camera, "world")
print(f"Position in world: "
      f"({pose_in_world.pose.x}, "
      f"{pose_in_world.pose.y}, "
      f"{pose_in_world.pose.z})")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
// Get the current frame system configuration
fsConfig, err := machine.FrameSystemConfig(ctx)
if err != nil {
    logger.Fatal(err)
}
for _, frame := range fsConfig {
    fmt.Printf("Frame: %s\n", frame.Frame().Name())
}

// Transform a pose from one frame to another
poseInCamera := referenceframe.NewPoseInFrame("my-camera",
    spatialmath.NewPoseFromPoint(r3.Vector{X: 100, Y: 0, Z: 200}))
poseInWorld, err := machine.TransformPose(ctx, poseInCamera, "world", nil)
if err != nil {
    logger.Fatal(err)
}
fmt.Printf("Position in world: (%.1f, %.1f, %.1f)\n",
    poseInWorld.Pose().Point().X,
    poseInWorld.Pose().Point().Y,
    poseInWorld.Pose().Point().Z)
```

{{% /tab %}}
{{< /tabs >}}

The `TransformPose` call takes a pose in one frame and returns the equivalent
pose in another frame. The frame system uses the parent-child hierarchy and all
configured translations and orientations to compute the transform. This works
for any two frames in the tree, regardless of how many levels apart they are.

### 8. Transform poses in application logic

A common pattern is to detect something in a camera frame and convert it to
world coordinates for use by an arm or motion planner.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.common import PoseInFrame, Pose

# Suppose a vision service reports an object at (150, -20, 400) in camera frame
object_in_camera = PoseInFrame(
    reference_frame="my-camera",
    pose=Pose(x=150, y=-20, z=400)
)

# Convert to world frame for the motion planner
object_in_world = await machine.transform_pose(object_in_camera, "world")
print(f"Object in world: "
      f"x={object_in_world.pose.x:.1f}, "
      f"y={object_in_world.pose.y:.1f}, "
      f"z={object_in_world.pose.z:.1f}")

# Convert to arm frame for direct arm movement
object_in_arm = await machine.transform_pose(object_in_camera, "my-arm")
print(f"Object in arm frame: "
      f"x={object_in_arm.pose.x:.1f}, "
      f"y={object_in_arm.pose.y:.1f}, "
      f"z={object_in_arm.pose.z:.1f}")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
// Suppose a vision service reports an object at (150, -20, 400) in camera frame
objectInCamera := referenceframe.NewPoseInFrame("my-camera",
    spatialmath.NewPoseFromPoint(r3.Vector{X: 150, Y: -20, Z: 400}))

// Convert to world frame for the motion planner
objectInWorld, err := machine.TransformPose(ctx, objectInCamera, "world", nil)
if err != nil {
    logger.Fatal(err)
}
pt := objectInWorld.Pose().Point()
fmt.Printf("Object in world: x=%.1f, y=%.1f, z=%.1f\n", pt.X, pt.Y, pt.Z)

// Convert to arm frame for direct arm movement
objectInArm, err := machine.TransformPose(ctx, objectInCamera, "my-arm", nil)
if err != nil {
    logger.Fatal(err)
}
armPt := objectInArm.Pose().Point()
fmt.Printf("Object in arm frame: x=%.1f, y=%.1f, z=%.1f\n",
    armPt.X, armPt.Y, armPt.Z)
```

{{% /tab %}}
{{< /tabs >}}

## Try It

1. Add frames to at least two components on your machine (an arm and a camera,
   or any two components).
2. Open the **VISUALIZE** tab and confirm both frames appear in the correct
   positions relative to each other.
3. Run the frame system query code from step 7 and verify that all configured
   frames are listed.
4. Pick a known physical point (such as the camera's position), express it in
   the camera's frame as `(0, 0, 0)`, transform it to the world frame, and
   verify the result matches your measured translation values.

## Troubleshooting

{{< expand "Frames not appearing in the VISUALIZE tab" >}}

- Confirm you clicked **Save** after adding the frame configuration.
- Verify that `viam-server` is running and the machine is online (green dot in
  the Viam app).
- Check that the `parent` field references a valid frame name. If you
  misspell the parent name, the frame will not appear in the tree.

{{< /expand >}}

{{< expand "Axes point the wrong direction" >}}

- Use the arm's TEST panel to jog in each axis direction and observe the
  physical movement.
- Adjust the `orientation` field. A 180-degree rotation around z flips x and y.
  A 180-degree rotation around x flips y and z.
- If using `euler_angles`, double-check the order of rotations. Euler angle
  conventions vary between systems.

{{< /expand >}}

{{< expand "Translation offsets look wrong in the visualizer" >}}

- Verify your measurements. Translations are in millimeters, not centimeters or
  meters. A 10 cm offset is `100` mm.
- Check which axis is which. If x and y are swapped compared to your
  expectation, the parent frame's orientation may differ from what you assumed.
- Confirm the parent frame is correct. An offset of `(100, 0, 0)` from
  `"world"` and `(100, 0, 0)` from `"my-arm"` will produce different world
  positions.

{{< /expand >}}

{{< expand "TransformPose returns unexpected values" >}}

- Print the full frame system configuration (step 7) and verify all frames have
  the correct parent, translation, and orientation.
- Check that you are using the correct frame names in the `TransformPose` call.
  Frame names are case-sensitive and must match the component names exactly.
- If the arm has moved, joint positions affect the transform. The frame system
  accounts for the arm's current joint configuration when computing transforms.

{{< /expand >}}

## What's Next

- [Configure Robot Kinematics](/work-cell-layout/configure-robot-kinematics/) -- set up the
  kinematic model that describes how your arm's joints create motion.
- [Calibrate Camera to Robot](/work-cell-layout/calibrate-camera-to-robot/) -- determine your
  camera's intrinsic parameters for accurate 2D-to-3D projection.
- [Define Obstacles](/work-cell-layout/define-obstacles/) -- add collision geometry so the
  motion planner can plan safe paths around physical objects.
