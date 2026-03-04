---
linkTitle: "Define Obstacles"
title: "Define Obstacles"
weight: 40
layout: "docs"
type: "docs"
description: "Add collision geometry so the motion planner can compute safe, collision-free paths."
date: "2025-01-30"
aliases:
  - /build/work-cell-layout/define-obstacles/
---

## What Problem This Solves

A robot arm that does not know about its surroundings will plan the shortest path to a target, even if that path goes through a table, a wall, or another piece of equipment.
By defining obstacles, you give the motion planner the information it needs to plan collision-free paths.
This how-to covers both static obstacles configured in the Viam app and dynamic obstacles defined programmatically at runtime.

## Concepts

### Why define obstacles

The motion service plans paths for your arm using the frame system and the arm's
kinematic model. By default, it only knows about the arm itself. Any other
physical object in the workspace is invisible to the planner unless you define
it as an obstacle.

Defining obstacles does two things:

1. **Prevents collisions.** The planner checks every candidate path against all
   defined obstacles and rejects paths that intersect them.
2. **Informs path planning.** The planner can find paths around obstacles that
   would otherwise be unreachable with a straight-line motion.

### Geometry types

Viam supports three primitive geometry types for defining obstacles:

| Type | Parameters | Best for |
|------|-----------|----------|
| **Box** (RectangularPrism) | x, y, z dimensions in mm | Tables, walls, shelves, rectangular equipment |
| **Sphere** | radius in mm | Balls, rounded objects, approximate keep-out zones |
| **Capsule** | radius + length in mm | Posts, pipes, cylindrical objects |

Each geometry has a center point (pose) that defines where it sits in space. The
pose is relative to a reference frame -- typically the world frame.

You do not need to model every surface detail of an obstacle. Approximate shapes
that fully enclose the real object are safer and work better with the motion
planner. A table can be modeled as a single box. A support post can be modeled
as a capsule.

### WorldState

WorldState is the container that holds obstacle and transform information passed
to the motion service at call time. It contains:

- **obstacles**: a list of `GeometriesInFrame`, where each entry specifies a
  reference frame and one or more geometries in that frame
- **transforms**: additional frame transforms (used less commonly)

You construct a WorldState in your code and pass it to the `Move` call. This
allows you to add, remove, or modify obstacles between motion calls without
reconfiguring the machine.

### Static vs dynamic obstacles

**Static obstacles** are permanent fixtures in your workspace. You configure
them by adding geometry to a component's frame configuration in the Viam app.
They are always present and do not require any code changes. Examples: the table
the arm is mounted on, walls, permanent fixtures.

**Dynamic obstacles** are objects that you define at runtime and pass to the
motion service via WorldState. They can change between calls. Examples: a box
that was just placed on the table, a temporary keep-out zone, objects detected
by a vision system.

Both types use the same geometry primitives. The difference is only in where
they are defined (configuration vs code).

### Keep-out zones

A keep-out zone is a region of space where the robot should never enter. You
define it the same way as any other obstacle -- as a box, sphere, or capsule
positioned in the workspace. The motion planner treats it identically to a
physical obstacle: it routes around it.

Common keep-out zones include areas near people, fragile equipment, or
sensor blind spots.

## Steps

### 1. Add geometry to a component frame

For permanent fixtures, add geometry directly to a component's frame
configuration. This is the simplest way to define static obstacles.

**Example: a table under the arm.**

1. Go to [app.viam.com](https://app.viam.com) and navigate to your machine.
2. Click the **CONFIGURE** tab.
3. Add a new component to represent the table obstacle (or add the geometry to
   an existing component's frame).
4. Configure the frame with a geometry:

```json
{
  "parent": "world",
  "translation": { "x": 0, "y": 0, "z": -20 },
  "orientation": {
    "type": "ov_degrees",
    "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
  },
  "geometry": {
    "type": "box",
    "x": 800,
    "y": 600,
    "z": 40
  }
}
```

This defines a table as a box 800 mm wide, 600 mm deep, and 40 mm thick. The
center of the box is at z = -20 mm (20 mm below the world frame origin), so
the top surface of the 40 mm thick table aligns with z = 0.

Click **Save**. The table geometry is now part of the frame system and the
motion planner will avoid it automatically.

### 2. Define obstacles programmatically with WorldState

For obstacles that change at runtime, build a WorldState in your code.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.common import (
    Pose, Vector3, RectangularPrism, Capsule, Sphere,
    Geometry, GeometriesInFrame, WorldState
)

# Define a table as a box
table_origin = Pose(x=-202.5, y=-546.5, z=-19.0)
table_dims = Vector3(x=635.0, y=1271.0, z=38.0)
table_obstacle = Geometry(
    center=table_origin,
    box=RectangularPrism(dims_mm=table_dims),
    label="table"
)

# Define a post as a capsule
post_origin = Pose(x=200, y=0, z=150)
post_obstacle = Geometry(
    center=post_origin,
    capsule=Capsule(radius_mm=25, length_mm=300),
    label="post"
)

# Define a ball as a sphere
ball_origin = Pose(x=100, y=100, z=50)
ball_obstacle = Geometry(
    center=ball_origin,
    sphere=Sphere(radius_mm=40),
    label="ball"
)

# Combine into WorldState
obstacles_in_frame = GeometriesInFrame(
    reference_frame="world",
    geometries=[table_obstacle, post_obstacle, ball_obstacle]
)
world_state = WorldState(obstacles=[obstacles_in_frame])
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
import (
    "github.com/golang/geo/r3"
    "go.viam.com/rdk/referenceframe"
    "go.viam.com/rdk/spatialmath"
)

obstacles := make([]spatialmath.Geometry, 0)

// Define a table as a box
tableOrigin := spatialmath.NewPose(
    r3.Vector{X: -202.5, Y: -546.5, Z: -19.0},
    &spatialmath.OrientationVectorDegrees{OX: 0, OY: 0, OZ: 1, Theta: 0},
)
tableDims := r3.Vector{X: 635.0, Y: 1271.0, Z: 38.0}
tableObj, err := spatialmath.NewBox(tableOrigin, tableDims, "table")
if err != nil {
    logger.Fatal(err)
}
obstacles = append(obstacles, tableObj)

// Define a post as a capsule
postOrigin := spatialmath.NewPose(
    r3.Vector{X: 200, Y: 0, Z: 150},
    &spatialmath.OrientationVectorDegrees{OX: 0, OY: 0, OZ: 1, Theta: 0},
)
postObj, err := spatialmath.NewCapsule(postOrigin, 25, 300, "post")
if err != nil {
    logger.Fatal(err)
}
obstacles = append(obstacles, postObj)

// Define a ball as a sphere
ballOrigin := spatialmath.NewPose(
    r3.Vector{X: 100, Y: 100, Z: 50},
    &spatialmath.OrientationVectorDegrees{OX: 0, OY: 0, OZ: 1, Theta: 0},
)
ballObj, err := spatialmath.NewSphere(ballOrigin, 40, "ball")
if err != nil {
    logger.Fatal(err)
}
obstacles = append(obstacles, ballObj)

// Combine into WorldState
obstaclesInFrame := referenceframe.NewGeometriesInFrame(
    referenceframe.World, obstacles)
worldState, err := referenceframe.NewWorldState(
    []*referenceframe.GeometriesInFrame{obstaclesInFrame}, nil)
if err != nil {
    logger.Fatal(err)
}
```

{{% /tab %}}
{{< /tabs >}}

Each geometry has a `center` pose that positions it in the specified reference
frame. All dimensions are in millimeters. Labels are optional but help with
debugging.

### 3. Use WorldState with the motion service

Pass the WorldState to the motion service's `Move` method. The planner uses the
obstacles to compute a collision-free path.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.services.motion import MotionClient
from viam.proto.common import PoseInFrame, Pose
from viam.proto.common import ResourceName

motion_service = MotionClient.from_robot(machine, "builtin")

arm_name = ResourceName(
    namespace="rdk",
    type="component",
    subtype="arm",
    name="my-arm"
)

destination = PoseInFrame(
    reference_frame="world",
    pose=Pose(x=300, y=200, z=400, o_x=0, o_y=0, o_z=-1, theta=0)
)

# Move with obstacle avoidance
await motion_service.move(
    component_name=arm_name,
    destination=destination,
    world_state=world_state
)
print("Arm moved to destination, avoiding obstacles")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
import (
    "go.viam.com/rdk/services/motion"
    "go.viam.com/rdk/resource"
)

motionService, err := motion.FromRobot(machine, "builtin")
if err != nil {
    logger.Fatal(err)
}

armName := resource.Name{
    API:    resource.NewAPI("rdk", "component", "arm"),
    Remote: "",
    Name:   "my-arm",
}

destination := referenceframe.NewPoseInFrame("world",
    spatialmath.NewPose(
        r3.Vector{X: 300, Y: 200, Z: 400},
        &spatialmath.OrientationVectorDegrees{OX: 0, OY: 0, OZ: -1, Theta: 0},
    ))

moveReq := motion.MoveReq{
    ComponentName: armName,
    Destination:   destination,
    WorldState:    worldState,
}
_, err = motionService.Move(ctx, moveReq)
if err != nil {
    logger.Fatal(err)
}
fmt.Println("Arm moved to destination, avoiding obstacles")
```

{{% /tab %}}
{{< /tabs >}}

If the motion planner cannot find a collision-free path, it returns an error.
This typically means the obstacles are too restrictive, the destination is inside
an obstacle, or the arm physically cannot reach the destination without
colliding.

### 4. Verify collision avoidance

Test that the motion planner actually routes around your obstacles:

1. Define an obstacle between the arm's current position and the target
   position.
2. Call `Move` with the WorldState.
3. Watch the arm move. It should take a curved or indirect path to avoid the
   obstacle.
4. Remove the obstacle from the WorldState and call `Move` again. The arm
   should take a more direct path.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.common import (
    Pose, Vector3, RectangularPrism,
    Geometry, GeometriesInFrame, WorldState, PoseInFrame
)

# Place a box obstacle directly between the arm and the target
blocker_origin = Pose(x=150, y=100, z=300)
blocker_dims = Vector3(x=200, y=200, z=200)
blocker = Geometry(
    center=blocker_origin,
    box=RectangularPrism(dims_mm=blocker_dims),
    label="test-blocker"
)

test_obstacles = GeometriesInFrame(
    reference_frame="world",
    geometries=[blocker]
)
test_world_state = WorldState(obstacles=[test_obstacles])

destination = PoseInFrame(
    reference_frame="world",
    pose=Pose(x=300, y=200, z=400, o_x=0, o_y=0, o_z=-1, theta=0)
)

# Move with the blocker in place
try:
    await motion_service.move(
        component_name=arm_name,
        destination=destination,
        world_state=test_world_state
    )
    print("Arm reached destination while avoiding the blocker")
except Exception as e:
    print(f"Motion planner could not find a path: {e}")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
blockerOrigin := spatialmath.NewPose(
    r3.Vector{X: 150, Y: 100, Z: 300},
    &spatialmath.OrientationVectorDegrees{OX: 0, OY: 0, OZ: 1, Theta: 0},
)
blockerDims := r3.Vector{X: 200, Y: 200, Z: 200}
blockerObj, err := spatialmath.NewBox(blockerOrigin, blockerDims, "test-blocker")
if err != nil {
    logger.Fatal(err)
}

testObstacles := referenceframe.NewGeometriesInFrame(
    referenceframe.World, []spatialmath.Geometry{blockerObj})
testWorldState, err := referenceframe.NewWorldState(
    []*referenceframe.GeometriesInFrame{testObstacles}, nil)
if err != nil {
    logger.Fatal(err)
}

destination := referenceframe.NewPoseInFrame("world",
    spatialmath.NewPose(
        r3.Vector{X: 300, Y: 200, Z: 400},
        &spatialmath.OrientationVectorDegrees{OX: 0, OY: 0, OZ: -1, Theta: 0},
    ))

moveReq := motion.MoveReq{
    ComponentName: armName,
    Destination:   destination,
    WorldState:    testWorldState,
}
_, err = motionService.Move(ctx, moveReq)
if err != nil {
    fmt.Printf("Motion planner could not find a path: %v\n", err)
} else {
    fmt.Println("Arm reached destination while avoiding the blocker")
}
```

{{% /tab %}}
{{< /tabs >}}

### 5. Visualize obstacles

Check the Viam app to see your obstacle geometry rendered in 3D:

1. Navigate to your machine in the Viam app.
2. Click the **VISUALIZE** tab.
3. Obstacles defined in component frame configurations appear as translucent
   shapes in the 3D view.
4. Verify that each obstacle's size, position, and shape match the physical
   objects in your workspace.

Dynamic obstacles (defined via WorldState in code) do not appear in the
VISUALIZE tab because they only exist during the `Move` call. To debug dynamic
obstacles, add them temporarily to a component's frame configuration so you can
see them in the visualizer, then move them back to code once they look correct.

### 6. Build a complete workspace model

For a realistic work cell, define all significant obstacles. Here is an example
that models a table, two walls, and an equipment rack:

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.common import (
    Pose, Vector3, RectangularPrism,
    Geometry, GeometriesInFrame, WorldState
)

# Table surface
table = Geometry(
    center=Pose(x=0, y=0, z=-20),
    box=RectangularPrism(dims_mm=Vector3(x=800, y=600, z=40)),
    label="table"
)

# Back wall
back_wall = Geometry(
    center=Pose(x=0, y=400, z=500),
    box=RectangularPrism(dims_mm=Vector3(x=1200, y=20, z=1000)),
    label="back-wall"
)

# Side wall
side_wall = Geometry(
    center=Pose(x=-500, y=0, z=500),
    box=RectangularPrism(dims_mm=Vector3(x=20, y=800, z=1000)),
    label="side-wall"
)

# Equipment rack
rack = Geometry(
    center=Pose(x=400, y=200, z=300),
    box=RectangularPrism(dims_mm=Vector3(x=200, y=200, z=600)),
    label="equipment-rack"
)

workspace_obstacles = GeometriesInFrame(
    reference_frame="world",
    geometries=[table, back_wall, side_wall, rack]
)
workspace = WorldState(obstacles=[workspace_obstacles])
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
obstacles := make([]spatialmath.Geometry, 0)

// Table surface
tableOrigin := spatialmath.NewPose(
    r3.Vector{X: 0, Y: 0, Z: -20},
    &spatialmath.OrientationVectorDegrees{OX: 0, OY: 0, OZ: 1, Theta: 0},
)
table, _ := spatialmath.NewBox(tableOrigin, r3.Vector{X: 800, Y: 600, Z: 40}, "table")
obstacles = append(obstacles, table)

// Back wall
backWallOrigin := spatialmath.NewPose(
    r3.Vector{X: 0, Y: 400, Z: 500},
    &spatialmath.OrientationVectorDegrees{OX: 0, OY: 0, OZ: 1, Theta: 0},
)
backWall, _ := spatialmath.NewBox(backWallOrigin, r3.Vector{X: 1200, Y: 20, Z: 1000}, "back-wall")
obstacles = append(obstacles, backWall)

// Side wall
sideWallOrigin := spatialmath.NewPose(
    r3.Vector{X: -500, Y: 0, Z: 500},
    &spatialmath.OrientationVectorDegrees{OX: 0, OY: 0, OZ: 1, Theta: 0},
)
sideWall, _ := spatialmath.NewBox(sideWallOrigin, r3.Vector{X: 20, Y: 800, Z: 1000}, "side-wall")
obstacles = append(obstacles, sideWall)

// Equipment rack
rackOrigin := spatialmath.NewPose(
    r3.Vector{X: 400, Y: 200, Z: 300},
    &spatialmath.OrientationVectorDegrees{OX: 0, OY: 0, OZ: 1, Theta: 0},
)
rack, _ := spatialmath.NewBox(rackOrigin, r3.Vector{X: 200, Y: 200, Z: 600}, "equipment-rack")
obstacles = append(obstacles, rack)

workspaceObstacles := referenceframe.NewGeometriesInFrame(
    referenceframe.World, obstacles)
workspace, _ := referenceframe.NewWorldState(
    []*referenceframe.GeometriesInFrame{workspaceObstacles}, nil)
```

{{% /tab %}}
{{< /tabs >}}

Use this WorldState with every `Move` call to ensure the arm always avoids these
obstacles.

## Try It

1. Add a table geometry to a component frame in the CONFIGURE tab (step 1).
   Open the VISUALIZE tab and verify the table appears in the correct position.
2. Write code to define a dynamic obstacle using WorldState (step 2) and pass
   it to a `Move` call (step 3).
3. Place a test obstacle between the arm and a target position (step 4). Verify
   the arm takes an indirect path to avoid it.
4. Remove the test obstacle and move again. Verify the arm takes a more direct
   path.
5. Build a complete workspace model (step 6) with at least three obstacles and
   verify the arm navigates safely among them.

## Troubleshooting

{{< expand "Motion planner cannot find a path" >}}

- The obstacles may be too large or too close together, leaving no valid path
  for the arm. Reduce the obstacle dimensions slightly. Obstacles should be
  slightly larger than the real objects they represent, but not so large that
  they block all paths.
- The destination may be inside or very close to an obstacle. The planner
  requires clearance around both the path and the destination.
- Try a different destination to confirm the planner works in general. If all
  destinations fail, the obstacle configuration may be enclosing the arm.

{{< /expand >}}

{{< expand "Obstacles appear in the wrong position" >}}

- Verify the reference frame. If obstacles are defined in the `"world"` frame,
  their positions are relative to the world frame origin. If you use a different
  reference frame, the positions are relative to that frame's origin.
- Check translation values. Positions are in millimeters, not centimeters or
  meters. A 60 cm table is `600` mm.
- Use the VISUALIZE tab to see where obstacles appear. Adjust the center pose
  until the obstacle aligns with the physical object.

{{< /expand >}}

{{< expand "Arm clips through obstacles" >}}

- The obstacle geometry may be too small to fully enclose the physical object.
  Increase the dimensions to add a safety margin. A good practice is to make
  obstacles 20-50 mm larger than the real objects in each dimension.
- The arm's own collision geometry may be missing or too small. Check the arm's
  kinematics file for link geometry definitions.
- If the arm moves very fast, it may pass through thin obstacles between planner
  checks. Make thin obstacles (walls, panels) at least 20 mm thick.

{{< /expand >}}

{{< expand "WorldState obstacles are not working" >}}

- Confirm you are passing the WorldState to the `Move` call. If you construct
  the WorldState but do not pass it, the planner will not see the obstacles.
- Verify the reference frame name in the GeometriesInFrame matches a frame in
  your frame system. The most common choice is `"world"`.
- Print the WorldState before the `Move` call to verify it contains the expected
  obstacles with correct positions and dimensions.

{{< /expand >}}

## What's Next

- [Move to Pose](/build/arm-manipulation/move-to-pose/) -- use the motion service
  to move the arm's end effector to a target position while avoiding obstacles.
- [Configure Robot Kinematics](/work-cell-layout/configure-robot-kinematics/) -- understand the
  kinematic model that the motion planner uses alongside obstacle definitions.
- [Calibrate Camera to Robot](/work-cell-layout/calibrate-camera-to-robot/) -- calibrate your
  camera so vision-detected objects can be added as dynamic obstacles with
  accurate positions.
