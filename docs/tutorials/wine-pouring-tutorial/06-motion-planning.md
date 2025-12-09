# Part 6: Motion Planning & Choreography

In this part, you'll program the arm movements for the wine pouring sequence. You'll learn how to create named positions, plan collision-free paths, and coordinate dual-arm movements.

**What you'll learn:**
- Saving and recalling arm positions
- Using the motion service for path planning
- Creating position sequences (choreography)
- Coordinating movements between two arms
- Building reusable motion primitives

**Time required:** 1.5-2 hours

**Prerequisites:** Complete [Part 5: Vision Pipelines](./05-vision-pipelines.md)

## The Pour Sequence

The complete wine pouring demo follows this sequence:

```
1. PREP       → Both arms move to watch positions
2. TOUCH      → Left arm touches/picks up the glass
3. POUR-PREP  → Right arm grabs bottle, both arms position for pour
4. POUR       → Right arm tilts bottle, liquid flows
5. PUT-BACK   → Both arms return to start positions
```

Each step involves precise coordination between the arms, with collision avoidance active throughout.

## Step 1: Position Saver Components

The demo uses "position saver" components - switches that move an arm to a predefined joint configuration.

### 1.1 Understand the Pattern

Each position saver is a switch component:
```json
{
  "name": "left-stow",
  "api": "rdk:component:switch",
  "model": "erh:vmodutils:arm-position-saver",
  "attributes": {
    "arm": "left-arm",
    "joints": [0, 0, 0, 0, 0, 0],
    "motion": ""
  }
}
```

**Attributes:**
- `arm`: Which arm to move
- `joints`: Array of 6 joint positions (radians) for the xArm 6
- `motion`: Optional motion service name for path planning

### 1.2 Create Key Positions

You'll need to teach the robot positions by:
1. Manually moving the arm to the desired pose
2. Reading the joint values
3. Creating a position saver with those values

#### Teaching a Position

```python
from viam.components.arm import Arm

async def teach_position(robot, arm_name):
    arm = Arm.from_robot(robot, arm_name)
    
    # Read current joint positions
    positions = await arm.get_joint_positions()
    
    print(f"Current {arm_name} joints (radians):")
    print(f"  [{', '.join([f'{p:.6f}' for p in positions.values])}]")
    
    # You can now copy these values to a position saver config
```

### 1.3 Define the Core Positions

Here are the key positions for the demo:

**Stow Positions** (arms tucked away safely):
```json
{
  "name": "left-stow",
  "api": "rdk:component:switch",
  "model": "erh:vmodutils:arm-position-saver",
  "attributes": {
    "arm": "left-arm",
    "joints": [0, 0, 0, 0, 0, 0],
    "motion": ""
  },
  "ui_folder": { "name": "admin-poses" }
}
```

```json
{
  "name": "right-stow",
  "api": "rdk:component:switch",
  "model": "erh:vmodutils:arm-position-saver",
  "attributes": {
    "arm": "right-arm",
    "joints": [0, 0, 0, 0, 0, 0],
    "motion": ""
  },
  "ui_folder": { "name": "admin-poses" }
}
```

**Watch Positions** (arms ready, observing the table):
```json
{
  "name": "arm-left-watch-pose",
  "api": "rdk:component:switch",
  "model": "erh:vmodutils:arm-position-saver",
  "attributes": {
    "arm": "left-arm",
    "joints": [-0.5, -0.8, -1.2, -1.5, 1.57, 0],
    "motion": ""
  }
}
```

**End-Effector Assembly Positions** (for attaching grippers):
```json
{
  "name": "left-ee-assemble",
  "api": "rdk:component:switch",
  "model": "erh:vmodutils:arm-position-saver",
  "attributes": {
    "arm": "left-arm",
    "joints": [0.672, 0.528, -0.357, 2.786, 2.521, -0.102],
    "motion": ""
  },
  "ui_folder": { "name": "admin-poses" }
}
```

## Step 2: Testing Position Savers

### 2.1 Via the Control Tab

1. Go to **CONTROL** tab
2. Find the position saver (e.g., `left-stow`) under switches
3. Click the toggle to activate
4. The arm moves to the saved position

### 2.2 Via SDK

```python
from viam.components.switch import Switch

async def go_to_position(robot, position_name):
    switch = Switch.from_robot(robot, position_name)
    
    # Get current state
    positions = await switch.get_number_of_positions()
    current = await switch.get_position()
    print(f"{position_name}: {current}/{positions}")
    
    # Activate the position (move arm)
    await switch.set_position(1)
    print(f"Moved to {position_name}")
```

## Step 3: Using the Motion Service

For more complex movements - especially those requiring obstacle avoidance - use the motion service.

### 3.1 Basic Motion Planning

```python
from viam.services.motion import Motion
from viam.proto.common import Pose, PoseInFrame

async def move_arm_to_pose(robot, target_x, target_y, target_z):
    motion = Motion.from_robot(robot, "builtin")
    
    # Define target pose in world coordinates
    target = Pose(
        x=target_x,
        y=target_y, 
        z=target_z,
        o_x=0, o_y=1, o_z=0, theta=0  # Pointing down
    )
    
    goal = PoseInFrame(
        reference_frame="world",
        pose=target
    )
    
    # Plan and execute
    await motion.move(
        component_name="left-arm",
        destination=goal
    )
```

### 3.2 Motion with Constraints

The motion service supports constraints:

```python
from viam.proto.service.motion import Constraints, LinearConstraint

async def move_with_constraints(robot):
    motion = Motion.from_robot(robot, "builtin")
    
    # Keep the gripper level during motion
    constraints = Constraints(
        linear_constraint=[
            LinearConstraint(
                line_tolerance_mm=1.0,
                orientation_tolerance_degs=5.0
            )
        ]
    )
    
    await motion.move(
        component_name="left-arm",
        destination=goal,
        constraints=constraints
    )
```

## Step 4: Position Sequences

The `vinocart` service uses named position sequences for choreography. Each sequence is a list of poses to hit in order.

### 4.1 Sequence Structure

From the demo config:
```json
{
  "positions": {
    "prep": {
      "prep": [
        ["arm-left-watch-pose", "arm-right-watch-pose"]
      ]
    },
    "pour": {
      "prep": [
        ["arm-pour-left-prep", "arm-pour-right-prep"],
        ["arm-pour-right-pos0"]
      ],
      "finish": [
        ["arm-pour-left-prep", "arm-pour-right-prep"]
      ]
    }
  }
}
```

**Structure:**
- Top level: Phase names (`prep`, `pour`, `pour_prep`, `put-back`, `reset`)
- Second level: Sub-phases within each phase
- Arrays: Each inner array is executed in parallel
- Sequence: Arrays are executed sequentially

### 4.2 Parallel vs Sequential

```
["arm-left-watch-pose", "arm-right-watch-pose"]
```
Both arms move simultaneously to their watch poses.

```
[
  ["arm-pour-left-prep", "arm-pour-right-prep"],
  ["arm-pour-right-pos0"]
]
```
First: Both arms move to prep positions (parallel)
Then: Right arm moves to pos0 (sequential)

## Step 5: Creating Pour Positions

The pour sequence requires careful positioning:

### 5.1 Bottle Grab Positions

```json
{
  "name": "arm-right-bottle-prep2",
  "api": "rdk:component:switch",
  "model": "erh:vmodutils:arm-position-saver",
  "attributes": {
    "arm": "right-arm",
    "joints": [-0.721, 0.559, -0.337, 3.134, 2.508, 0.0]
  }
}
```

### 5.2 Pour Tilt Positions

Create a sequence of tilt angles for smooth pouring:

```json
{
  "name": "arm-pour-right-pos0",
  "attributes": {
    "arm": "right-arm",
    "joints": [/* bottle upright */]
  }
}
```

```json
{
  "name": "arm-pour-right-pos1",
  "attributes": {
    "arm": "right-arm", 
    "joints": [/* bottle tilted 30° */]
  }
}
```

```json
{
  "name": "arm-pour-right-pos2",
  "attributes": {
    "arm": "right-arm",
    "joints": [/* bottle tilted 60° */]
  }
}
```

### 5.3 Glass Positioning

The left arm positions the glass under the bottle:

```python
async def position_glass_for_pour(robot, glass_position):
    """Move left arm to hold glass under pour point."""
    motion = Motion.from_robot(robot, "builtin")
    
    # Glass should be slightly forward and below bottle tip
    pour_x = glass_position.x + 50  # Offset for bottle tip
    pour_y = glass_position.y
    pour_z = glass_position.z + 150  # Lift glass up
    
    target = Pose(x=pour_x, y=pour_y, z=pour_z, ...)
    await motion.move(component_name="left-arm", destination=...)
```

## Step 6: Gripper Control

### 6.1 Opening and Closing Grippers

The xArm gripper is controlled via DoCommand:

```python
from viam.components.arm import Arm

async def control_gripper(robot, arm_name, position):
    """
    Control xArm gripper.
    position: 0 = closed, 850 = fully open
    """
    arm = Arm.from_robot(robot, arm_name)
    
    await arm.do_command({
        "setup_gripper": True,
        "move_gripper": position
    })

# Usage
await control_gripper(robot, "left-arm", 850)  # Open
await control_gripper(robot, "left-arm", 400)  # Close on glass
```

### 6.2 Glass Pickup Sequence

```python
async def pickup_glass(robot, glass_x, glass_y, glass_z):
    motion = Motion.from_robot(robot, "builtin")
    left_arm = Arm.from_robot(robot, "left-arm")
    
    # 1. Open gripper
    await left_arm.do_command({"setup_gripper": True, "move_gripper": 850})
    
    # 2. Move above glass
    approach = Pose(x=glass_x, y=glass_y, z=glass_z + 100, ...)
    await motion.move(component_name="left-arm", destination=PoseInFrame("world", approach))
    
    # 3. Move down to glass
    grasp = Pose(x=glass_x, y=glass_y, z=glass_z + 20, ...)
    await motion.move(component_name="left-arm", destination=PoseInFrame("world", grasp))
    
    # 4. Close gripper
    await left_arm.do_command({"setup_gripper": True, "move_gripper": 300})
    
    # 5. Lift glass
    lift = Pose(x=glass_x, y=glass_y, z=glass_z + 150, ...)
    await motion.move(component_name="left-arm", destination=PoseInFrame("world", lift))
```

## Step 7: Complete Pour Choreography

Here's a simplified version of the pour sequence:

```python
async def execute_pour(robot):
    # Services and components
    motion = Motion.from_robot(robot, "builtin")
    vision = Vision.from_robot(robot, "cup-finder-segment")
    left_arm = Arm.from_robot(robot, "left-arm")
    right_arm = Arm.from_robot(robot, "right-arm")
    
    # Phase 1: PREP - Move to watch positions
    print("Phase 1: Prep")
    await go_to_positions(robot, ["arm-left-watch-pose", "arm-right-watch-pose"])
    
    # Phase 2: DETECT - Find glass
    print("Phase 2: Detect glass")
    objects = await vision.get_object_point_clouds("cam-merged-cup")
    if not objects:
        print("No glass found!")
        return
    glass_pos = objects[0].geometries[0].center
    print(f"Glass at: {glass_pos.x}, {glass_pos.y}, {glass_pos.z}")
    
    # Phase 3: TOUCH - Pick up glass
    print("Phase 3: Touch/pickup")
    await pickup_glass(robot, glass_pos.x, glass_pos.y, glass_pos.z)
    
    # Phase 4: POUR-PREP - Position both arms
    print("Phase 4: Pour prep")
    await go_to_positions(robot, ["arm-pour-left-prep", "arm-pour-right-prep"])
    
    # Phase 5: POUR - Tilt bottle
    print("Phase 5: Pour")
    await go_to_position(robot, "arm-pour-right-pos1")
    await asyncio.sleep(2)  # Pour duration
    await go_to_position(robot, "arm-pour-right-pos0")  # Upright
    
    # Phase 6: PUT-BACK - Return everything
    print("Phase 6: Put back")
    await place_glass(robot, glass_pos)
    await go_to_positions(robot, ["arm-left-watch-pose", "arm-right-watch-pose"])
    
    print("Pour complete!")
```

## Step 8: Safety Considerations

### 8.1 Speed Limits

Always start with conservative speeds:
```json
{
  "speed_degs_per_sec": 30,
  "acceleration_degs_per_sec_sec": 50
}
```

### 8.2 Emergency Stop

Implement an e-stop check:
```python
async def check_estop(robot):
    # Check for stop flag in DoCommand
    cart = Generic.from_robot(robot, "cart")
    result = await cart.do_command({"status": True})
    return result.get("stopped", False)
```

### 8.3 Collision Monitoring

The motion service plans around obstacles, but also monitor for:
- Unexpected obstacles (humans in workspace)
- Arm-to-arm collisions during parallel moves
- Gripper collisions with objects

## Checkpoint: What You've Accomplished

✅ Created position saver components for key arm poses  
✅ Taught positions by reading joint values  
✅ Used the motion service for path planning  
✅ Built position sequences for choreography  
✅ Implemented gripper control  
✅ Created a complete pour sequence  

## Tips for Position Teaching

### Use Freedrive Mode
Many arms have a "freedrive" or "teach" mode where you can physically move the arm:
```python
# Enable freedrive on xArm (check xArm documentation)
await arm.do_command({"set_mode": 2})  # Manual mode
# Move arm by hand
# Record position
# Return to normal mode
await arm.do_command({"set_mode": 0})
```

### Test Incrementally
1. Test each position individually
2. Test pairs of positions
3. Test full sequences at slow speed
4. Gradually increase speed

### Document Your Positions
Keep notes on what each position is for:
```json
{
  "name": "arm-left-watch-pose",
  // Position: Left arm tucked back, looking at table center
  // Used for: Initial state, monitoring during bottle operations
  // Clearance: Clears right arm in all positions
}
```

## Troubleshooting

### Arm doesn't reach position
- Joint limits may prevent the configuration
- Try an alternative approach angle
- Check for obstacles in the path

### Motion planning fails
- Obstacles may block all paths
- Try intermediate waypoints
- Temporarily widen obstacle tolerances

### Arms collide during parallel move
- Sequence the moves instead of parallel
- Add intermediate positions
- Adjust timing with delays

### Gripper doesn't grip properly
- Adjust gripper close position
- Check object dimensions
- Verify gripper force settings

## Next Steps

In [Part 7: The Pouring Demo Module Deep Dive](./07-pouring-module.md), you'll explore the `viam:pouring-demo:vinocart` module that orchestrates all these components into a production-ready demo.

---

## Reference: Position Saver Configuration

```json
{
  "components": [
    {
      "name": "left-stow",
      "api": "rdk:component:switch",
      "model": "erh:vmodutils:arm-position-saver",
      "attributes": {
        "arm": "left-arm",
        "joints": [0, 0, 0, 0, 0, 0],
        "motion": ""
      },
      "ui_folder": { "name": "admin-poses" }
    },
    {
      "name": "right-stow",
      "api": "rdk:component:switch",
      "model": "erh:vmodutils:arm-position-saver",
      "attributes": {
        "arm": "right-arm",
        "joints": [0, 0, 0, 0, 0, 0],
        "motion": ""
      },
      "ui_folder": { "name": "admin-poses" }
    },
    {
      "name": "new-stow-left",
      "api": "rdk:component:switch",
      "model": "erh:vmodutils:arm-position-saver",
      "attributes": {
        "arm": "left-arm",
        "joints": [-1.162, -1.791, -3.176, -2.285, 2.807, 0.727],
        "motion": ""
      },
      "ui_folder": { "name": "admin-poses" }
    },
    {
      "name": "new-stow-right",
      "api": "rdk:component:switch",
      "model": "erh:vmodutils:arm-position-saver",
      "attributes": {
        "arm": "right-arm",
        "joints": [-1.311, 1.927, -3.155, 4.916, 1.990, -1.295],
        "motion": ""
      },
      "ui_folder": { "name": "admin-poses" }
    }
  ]
}
```
