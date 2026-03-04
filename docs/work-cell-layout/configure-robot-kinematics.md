---
linkTitle: "Configure Robot Kinematics"
title: "Configure Robot Kinematics"
weight: 20
layout: "docs"
type: "docs"
description: "Set up the kinematic model that describes how your arm's joints and links create motion."
date: "2025-01-30"
aliases:
  - /build/work-cell-layout/configure-robot-kinematics/
---

## What Problem This Solves

When you command a robot arm to move its end effector to a position in 3D space, something needs to figure out what angle each joint should be set to in order to reach that position.
This is the inverse kinematics problem, and solving it requires a mathematical model of the arm's physical structure.
This how-to covers the kinematics concepts, shows you how to inspect and verify your arm's kinematic configuration, and demonstrates how to read joint and end effector data programmatically.

## Concepts

### What kinematics are

Kinematics is the mathematical description of how a robot's joints and links
create motion. It defines the geometry of the arm -- link lengths, joint axes,
joint types -- without considering forces or torques. The kinematic model is what
allows Viam to compute where the end effector is (forward kinematics) and how to
get it to a target position (inverse kinematics).

### Forward kinematics

Forward kinematics answers the question: given the current angle of every joint,
where is the end effector? This is a straightforward calculation. You start at
the base, apply each joint angle and link length in sequence, and arrive at the
end effector's position and orientation in space.

Every time you call `GetEndPosition`, the arm uses forward kinematics to compute
the result from the current joint angles.

### Inverse kinematics

Inverse kinematics answers the reverse question: given a target position and
orientation for the end effector, what joint angles will get the arm there? This
is a harder problem because there may be zero, one, or many solutions. The arm
might be able to reach the same point with the elbow up or elbow down, or it
might not be able to reach the point at all.

Viam's motion planner uses inverse kinematics internally when you call
`MoveToPosition` or use the motion service's `Move` method.

### Links and joints

A robot arm is modeled as a chain of rigid bodies (links) connected by joints:

- **Links** are the rigid structural segments. Each link has a length and
  optionally a geometry (for collision checking). Links do not move on their
  own -- they are moved by the joints that connect them.
- **Joints** connect two links and define how they can move relative to each
  other. The two most common joint types are:
  - **Revolute**: rotates around an axis (like an elbow or shoulder). Measured in
    degrees.
  - **Prismatic**: slides along an axis (like a linear actuator). Measured in
    millimeters.

### Joint limits

Every joint has limits that define its range of motion:

- **Min/max angle** (revolute joints): the minimum and maximum rotation in
  degrees. For example, a shoulder joint might allow -180 to 180 degrees.
- **Min/max position** (prismatic joints): the minimum and maximum extension in
  millimeters.

Joint limits prevent the motion planner from computing solutions that would
require the arm to bend past its physical limits.

### Kinematics file formats

Viam supports three formats for describing arm kinematics:

| Format | Description | When to use |
|--------|-------------|-------------|
| **SVA** (Spatial Vector Algebra) | Viam's native JSON format | Preferred for new arms, most detailed |
| **DH** (Denavit-Hartenberg) | Standard robotics convention, four parameters per joint | When converting from textbook DH parameters |
| **URDF** | XML format used by ROS and many manufacturers | When the manufacturer provides a URDF file |

Most registry arm modules use SVA internally. You rarely need to write a
kinematics file from scratch unless you are building a custom arm.

### Tool center point (TCP)

The tool center point is the reference point on the end effector. By default,
it is at the last link's origin. If you attach a gripper or tool, you may need
to define an offset so the TCP reflects the actual point of interaction (for
example, the tip of a gripper or the center of a suction cup). This offset is
configured through the frame system, not the kinematics file.

## Steps

### 1. Check if your arm has a built-in kinematics file

Most arm modules in the Viam registry -- including modules for UR5e, xArm6, and
Viam Arm -- ship with a kinematics file built into the module. The module loads
and applies the kinematics automatically when `viam-server` starts.

You can verify this by calling `GetKinematics`:

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.components.arm import Arm

arm = Arm.from_robot(machine, "my-arm")
kinematics = await arm.get_kinematics()
print(f"Kinematics format: {kinematics[0]}")
# kinematics[1] contains the raw kinematics data
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
myArm, err := arm.FromRobot(machine, "my-arm")
if err != nil {
    logger.Fatal(err)
}

kinematicsType, kinematicsData, err := myArm.Kinematics(ctx, nil)
if err != nil {
    logger.Fatal(err)
}
fmt.Printf("Kinematics format: %v\n", kinematicsType)
fmt.Printf("Kinematics data length: %d bytes\n", len(kinematicsData))
```

{{% /tab %}}
{{< /tabs >}}

If this call succeeds and returns data, your arm module has kinematics built in.
You do not need to provide a kinematics file. Proceed to step 4 to read joint
and end effector data.

If the call fails or returns empty data, the module does not include kinematics.
You will need to provide a kinematics file (steps 2-3).

### 2. Understand the SVA kinematics format

The SVA format describes the arm as a sequence of links and joints in JSON. Here
is a simplified example for a two-joint arm:

```json
{
  "name": "MyArm",
  "kinematic_param_type": "SVA",
  "links": [
    {
      "id": "base_link",
      "parent": "world",
      "translation": { "x": 0, "y": 0, "z": 162.5 },
      "geometry": {
        "x": 120,
        "y": 120,
        "z": 260,
        "translation": { "x": 0, "y": 0, "z": 130 }
      }
    },
    {
      "id": "upper_arm_link",
      "parent": "shoulder_pan_joint",
      "translation": { "x": 0, "y": 0, "z": 245.0 }
    }
  ],
  "joints": [
    {
      "id": "shoulder_pan_joint",
      "type": "revolute",
      "parent": "base_link",
      "axis": { "x": 0, "y": 0, "z": 1 },
      "min": -360,
      "max": 360
    },
    {
      "id": "shoulder_lift_joint",
      "type": "revolute",
      "parent": "upper_arm_link",
      "axis": { "x": 0, "y": 1, "z": 0 },
      "min": -360,
      "max": 360
    }
  ]
}
```

Each field in the SVA format:

- **`links[].id`**: unique name for the link
- **`links[].parent`**: the joint or frame this link attaches to
- **`links[].translation`**: offset in mm from the parent's origin to this
  link's origin
- **`links[].geometry`**: optional collision shape dimensions in mm
- **`joints[].id`**: unique name for the joint
- **`joints[].type`**: `"revolute"` (rotates) or `"prismatic"` (slides)
- **`joints[].parent`**: the link this joint attaches to
- **`joints[].axis`**: the axis of rotation or translation (unit vector)
- **`joints[].min`** / **`joints[].max`**: joint limits in degrees (revolute)
  or mm (prismatic)

### 3. Import a URDF file

If your arm manufacturer provides a URDF file instead of SVA parameters, you can
reference it in your arm module's configuration. URDF (Unified Robot Description
Format) is an XML format that describes links, joints, visual meshes, and
collision geometry.

A typical URDF structure looks like this:

```xml
<robot name="my_arm">
  <link name="base_link">
    <visual>
      <geometry><cylinder length="0.1" radius="0.05"/></geometry>
    </visual>
    <collision>
      <geometry><cylinder length="0.1" radius="0.05"/></geometry>
    </collision>
  </link>
  <joint name="shoulder_pan" type="revolute">
    <parent link="base_link"/>
    <child link="upper_arm"/>
    <axis xyz="0 0 1"/>
    <limit lower="-3.14" upper="3.14" velocity="1.0"/>
  </joint>
</robot>
```

To use a URDF file with your arm module, place the file in a location accessible
to `viam-server` and reference it in the module's configuration. The exact
configuration depends on the module. Consult the module's documentation for the
specific attribute name.

### 4. Configure joint limits

Joint limits prevent the arm from attempting to reach configurations that are
physically impossible or dangerous. Review and adjust the limits for your
specific arm:

- **Revolute joints**: limits are in degrees. A limit of `"min": -180, "max":
  180` allows a full rotation in either direction. Some joints may have
  mechanical stops at smaller angles.
- **Prismatic joints**: limits are in millimeters. A limit of `"min": 0, "max":
  500` allows 500 mm of extension.

If your arm is mounted near a wall or other obstruction, you can tighten the
joint limits to prevent motion in restricted directions. This is separate from
obstacle avoidance -- joint limits are absolute constraints, while obstacles are
checked during path planning.

### 5. Read joint positions and end effector pose

Once kinematics are configured, you can read the arm's current state at any
time.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.components.arm import Arm

arm = Arm.from_robot(machine, "my-arm")

# Read current joint positions (in degrees for revolute joints)
joint_positions = await arm.get_joint_positions()
print(f"Joint positions: {joint_positions.values}")

# Read end effector position (in mm, relative to arm base)
end_position = await arm.get_end_position()
print(f"End effector position:")
print(f"  x={end_position.x:.1f} mm")
print(f"  y={end_position.y:.1f} mm")
print(f"  z={end_position.z:.1f} mm")
print(f"  orientation: o_x={end_position.o_x:.3f}, "
      f"o_y={end_position.o_y:.3f}, "
      f"o_z={end_position.o_z:.3f}, "
      f"theta={end_position.theta:.1f}")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
myArm, err := arm.FromRobot(machine, "my-arm")
if err != nil {
    logger.Fatal(err)
}

// Read current joint positions (in degrees for revolute joints)
jointPositions, err := myArm.JointPositions(ctx, nil)
if err != nil {
    logger.Fatal(err)
}
fmt.Printf("Joint positions: %v\n", jointPositions.Values)

// Read end effector position (in mm, relative to arm base)
endPosition, err := myArm.EndPosition(ctx, nil)
if err != nil {
    logger.Fatal(err)
}
pt := endPosition.Point()
fmt.Printf("End effector position:\n")
fmt.Printf("  x=%.1f mm\n", pt.X)
fmt.Printf("  y=%.1f mm\n", pt.Y)
fmt.Printf("  z=%.1f mm\n", pt.Z)
```

{{% /tab %}}
{{< /tabs >}}

The joint positions are reported in the order defined by the kinematics file,
starting from the base joint. For a 6-axis arm, you get 6 values.

### 6. Move the arm to specific joint positions

You can command the arm to move to a specific set of joint angles. This bypasses
inverse kinematics and directly sets each joint.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.component.arm import JointPositions

# Move all joints to their zero positions (home position)
home = JointPositions(values=[0, 0, 0, 0, 0, 0])
await arm.move_to_joint_positions(home)
print("Arm moved to home position")

# Read back the joint positions to confirm
current = await arm.get_joint_positions()
print(f"Current joint positions: {current.values}")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
// Move all joints to their zero positions (home position)
home := &armapi.JointPositions{Values: []float64{0, 0, 0, 0, 0, 0}}
err = myArm.MoveToJointPositions(ctx, home, nil)
if err != nil {
    logger.Fatal(err)
}
fmt.Println("Arm moved to home position")

// Read back the joint positions to confirm
current, err := myArm.JointPositions(ctx, nil)
if err != nil {
    logger.Fatal(err)
}
fmt.Printf("Current joint positions: %v\n", current.Values)
```

{{% /tab %}}
{{< /tabs >}}

The number of values must match the number of joints in the kinematics model.
Each value must be within the joint's configured limits.

### 7. Verify kinematics in the VISUALIZE tab

The Viam app can render a 3D visualization of your arm based on its kinematic
model:

1. Navigate to your machine in the Viam app.
2. Click the **VISUALIZE** tab.
3. The arm should appear as a 3D model with joints and links. If the arm module
   includes visual mesh data, the arm appears as a realistic model. Otherwise,
   it appears as a simplified wireframe.

Verify the visualization by comparing it to the physical arm:

1. Move a joint using the **CONTROL** tab.
2. Switch to the **VISUALIZE** tab and confirm the visualization updated.
3. Check that the joint rotated in the correct direction and by the correct
   amount.
4. Repeat for each joint.

If the visualization does not match the physical arm, the kinematics file may
have incorrect link lengths, joint axes, or joint limits. Contact the module
maintainer or consult the manufacturer's specifications.

### 8. Monitor joint positions over time

For applications that need to track the arm's state continuously, read joint
positions in a loop.

{{< tabs >}}
{{% tab name="Python" %}}

```python
import asyncio
from viam.components.arm import Arm

arm = Arm.from_robot(machine, "my-arm")

while True:
    joints = await arm.get_joint_positions()
    end_pos = await arm.get_end_position()

    joint_str = ", ".join(f"{v:.1f}" for v in joints.values)
    print(f"Joints: [{joint_str}] | "
          f"TCP: ({end_pos.x:.1f}, {end_pos.y:.1f}, {end_pos.z:.1f})")

    await asyncio.sleep(0.1)
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
myArm, err := arm.FromRobot(machine, "my-arm")
if err != nil {
    logger.Fatal(err)
}

for {
    joints, err := myArm.JointPositions(ctx, nil)
    if err != nil {
        logger.Error(err)
        time.Sleep(time.Second)
        continue
    }

    endPos, err := myArm.EndPosition(ctx, nil)
    if err != nil {
        logger.Error(err)
        time.Sleep(time.Second)
        continue
    }

    pt := endPos.Point()
    fmt.Printf("Joints: %v | TCP: (%.1f, %.1f, %.1f)\n",
        joints.Values, pt.X, pt.Y, pt.Z)

    time.Sleep(100 * time.Millisecond)
}
```

{{% /tab %}}
{{< /tabs >}}

## Try It

1. Run the kinematics check from step 1 to confirm your arm module has a
   built-in kinematics file.
2. Read joint positions and end effector pose (step 5). Move the arm using the
   CONTROL tab and run the code again to see the values change.
3. Move to the home position using joint positions (step 6), then read back the
   end effector position. This is your arm's home TCP location.
4. Open the VISUALIZE tab and compare the rendered arm to the physical arm. Move
   individual joints and verify the visualization matches.

## Troubleshooting

{{< expand "GetKinematics returns an error or empty data" >}}

- The arm module may not include a built-in kinematics file. Check the module's
  documentation or registry page for information about kinematics support.
- Verify the arm component is configured correctly and `viam-server` has no
  errors for this component. Check the logs tab in the Viam app.
- Ensure you are using the correct component name. The name in your code must
  match the name in the CONFIGURE tab exactly.

{{< /expand >}}

{{< expand "Joint limits are too restrictive" >}}

- The motion planner may fail to find paths because joint limits prevent it from
  exploring valid configurations. Check the kinematics file and widen limits if
  the physical arm allows greater range of motion.
- If the arm is a commercial model, use the manufacturer's documented joint
  limits. Do not exceed these values.

{{< /expand >}}

{{< expand "VISUALIZE tab does not match the physical arm" >}}

- The kinematics file may have incorrect link lengths. Measure the physical arm
  segments and compare to the values in the kinematics file.
- Joint axes may be swapped or inverted. Move one joint at a time in the CONTROL
  tab and verify the correct joint moves in the visualizer.
- If the arm model appears offset from the frame axes, check the arm's frame
  configuration in the CONFIGURE tab. The frame translation and orientation
  affect the arm's position in the visualizer.

{{< /expand >}}

{{< expand "URDF import fails" >}}

- Verify the URDF file is valid XML. Use an XML validator to check for syntax
  errors.
- Ensure all referenced mesh files (STL, DAE) are accessible at the paths
  specified in the URDF. Missing mesh files may cause import failures.
- Check that joint types in the URDF are supported (`revolute`, `prismatic`,
  `fixed`). Unsupported joint types may be ignored or cause errors.

{{< /expand >}}

## What's Next

- [Calibrate Camera to Robot](/work-cell-layout/calibrate-camera-to-robot/) -- calibrate your
  camera's intrinsic parameters for accurate 3D position estimation.
- [Define Obstacles](/work-cell-layout/define-obstacles/) -- add collision geometry to your
  workspace so the motion planner avoids collisions.
- [Move to Pose](/build/arm-manipulation/move-to-pose/) -- use inverse kinematics
  to move the arm's end effector to a target position in 3D space.
