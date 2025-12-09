# Part 3: The Frame System & Obstacle Definition

In this part, you'll learn how Viam models the spatial relationships between robot components and how to define obstacles for collision-free motion planning. This is critical for safe dual-arm operation.

**What you'll learn:**
- Understanding the frame system and transforms
- Defining component positions relative to each other
- Creating obstacle geometries for motion planning
- Building a world model for your robot workspace

**Time required:** 1-1.5 hours

**Prerequisites:** Complete [Part 2: Depth Cameras & Point Clouds](./02-cameras-point-clouds.md)

## Why Frames Matter

Your robot has multiple components - arms, cameras, grippers - each with its own local coordinate system. To coordinate their actions, Viam needs to know:

- Where is the left arm relative to the right arm?
- Where is the camera relative to the arm it's monitoring?
- Where is the table surface relative to the arm bases?

The **frame system** is Viam's way of encoding these spatial relationships. It's a tree structure where:
- Each component has a **frame** (coordinate system)
- Frames are connected by **transforms** (position + orientation offsets)
- The root frame is typically called `world`

## Step 1: Understanding the World Frame

By convention, `world` is the global reference frame. All other frames are defined relative to it or to each other.

For the wine pouring demo, we establish:
- **Origin**: A convenient reference point (e.g., between the two arm bases)
- **X-axis**: Points forward (toward the pouring area)
- **Y-axis**: Points left (parallel to the arm baseline)
- **Z-axis**: Points up

Here's the physical layout with coordinates:

```
                    Y = -90 (left wall)
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
    │  LEFT ARM          │           RIGHT ARM│
    │  (0, 0, 0)         │          (-18, 830)│
    │                    │                    │
    │                  ORIGIN                 │
    │                  (0,0,0)                │
────┼────────────────────┼────────────────────┼──── Y = 920 (right wall)
    │                    │                    │
    │           TABLE SURFACE                 │
    │           (162, 412, 0)                 │
    │                    │                    │
    └────────────────────┼────────────────────┘
                         │
                    X = 500 (front)
```

## Step 2: Configure Arm Frames

Each arm needs a frame definition that specifies where its base is located in world coordinates.

### 2.1 Configure Left Arm Frame

Edit the `left-arm` component to add a frame:

```json
{
  "name": "left-arm",
  "api": "rdk:component:arm",
  "model": "ufactory:xArm:xArm6",
  "attributes": {
    "host": "192.168.1.200",
    "speed_degs_per_sec": 30,
    "acceleration_degs_per_sec_sec": 50
  },
  "frame": {
    "parent": "world",
    "translation": {
      "x": 0,
      "y": 0,
      "z": 15
    },
    "orientation": {
      "type": "ov_degrees",
      "value": {
        "x": 0,
        "y": 0,
        "z": 1,
        "th": 0
      }
    }
  }
}
```

**Frame configuration explained:**
- `parent`: The frame this component is attached to (`world`)
- `translation`: Position offset in mm (X, Y, Z)
- `orientation`: Rotational offset
  - `type`: "ov_degrees" means orientation vector with angle in degrees
  - `x`, `y`, `z`: The axis of rotation (normalized)
  - `th`: The angle of rotation around that axis

The left arm is positioned at the world origin, raised 15mm (to account for its mounting plate).

### 2.2 Configure Right Arm Frame

The right arm is offset from the left:

```json
{
  "name": "right-arm",
  "api": "rdk:component:arm",
  "model": "ufactory:xArm:xArm6",
  "attributes": {
    "host": "192.168.1.208",
    "speed_degs_per_sec": 30,
    "acceleration_degs_per_sec_sec": 50
  },
  "frame": {
    "parent": "world",
    "translation": {
      "x": -18,
      "y": 830,
      "z": 15
    },
    "orientation": {
      "type": "ov_degrees",
      "value": {
        "x": 0,
        "y": 0,
        "z": 1,
        "th": 30
      }
    }
  }
}
```

This positions the right arm:
- 18mm back from the left arm (X = -18)
- 830mm to the right (Y = 830)
- Same height (Z = 15)
- Rotated 30° around the Z-axis (angled slightly toward center)

## Step 3: Define Obstacles

For safe motion planning, the robot needs to know what it might collide with. We define **obstacles** as geometric shapes positioned in the world.

### 3.1 Understanding Obstacle Components

The `erh:vmodutils:obstacle` model creates geometric obstacles that the motion planner will avoid. These are configured as gripper components (a clever use of the API that provides geometry support).

### 3.2 Define the Table Surface

```json
{
  "name": "obstacle-table",
  "api": "rdk:component:gripper",
  "model": "erh:vmodutils:obstacle",
  "attributes": {
    "geometries": [
      {
        "type": "box",
        "x": 705,
        "y": 1045,
        "z": 10,
        "label": "table"
      }
    ]
  },
  "frame": {
    "parent": "world",
    "translation": {
      "x": 162,
      "y": 412,
      "z": 0
    },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

**Geometry attributes:**
- `type`: Shape type ("box", "sphere", "capsule")
- `x`, `y`, `z`: Dimensions in mm (for a box: length, width, height)
- `label`: Human-readable name for debugging

The table is a 705mm × 1045mm × 10mm box, centered at (162, 412, 0).

### 3.3 Define Wall Obstacles

The demo has walls on three sides to define the workspace boundary:

**Back Wall:**
```json
{
  "name": "obstacle-back-wall",
  "api": "rdk:component:gripper",
  "model": "erh:vmodutils:obstacle",
  "attributes": {
    "geometries": [
      {
        "type": "box",
        "x": 40,
        "y": 1045,
        "z": 80,
        "label": "back-wall"
      }
    ]
  },
  "frame": {
    "parent": "world",
    "translation": { "x": -180, "y": 412, "z": 40 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

**Left Wall:**
```json
{
  "name": "obstacle-left-wall",
  "api": "rdk:component:gripper",
  "model": "erh:vmodutils:obstacle",
  "attributes": {
    "geometries": [
      {
        "type": "box",
        "x": 705,
        "y": 40,
        "z": 80,
        "label": "left-wall"
      }
    ]
  },
  "frame": {
    "parent": "world",
    "translation": { "x": 162, "y": -90, "z": 40 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

**Right Wall:**
```json
{
  "name": "obstacle-right-wall",
  "api": "rdk:component:gripper",
  "model": "erh:vmodutils:obstacle",
  "attributes": {
    "geometries": [
      {
        "type": "box",
        "x": 705,
        "y": 40,
        "z": 80,
        "label": "right-wall"
      }
    ]
  },
  "frame": {
    "parent": "world",
    "translation": { "x": 162, "y": 920, "z": 40 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

### 3.4 Corner Obstacles

The table corners are rounded/angled, so we add explicit corner obstacles:

```json
{
  "name": "obstacle-table-corner-left",
  "api": "rdk:component:gripper",
  "model": "erh:vmodutils:obstacle",
  "attributes": {
    "geometries": [
      {
        "type": "box",
        "x": 240,
        "y": 240,
        "z": 80,
        "label": "left-corner"
      }
    ]
  },
  "frame": {
    "parent": "obstacle-left-wall",
    "translation": { "x": 352, "y": -20, "z": 0 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 45 }
    }
  }
}
```

Note that this corner is defined **relative to the left wall** (`parent: obstacle-left-wall`), and rotated 45° to represent a diagonal corner piece.

### 3.5 Ceiling (Safety Boundary)

To prevent the arms from raising too high:

```json
{
  "name": "obstacle-ceiling",
  "api": "rdk:component:gripper",
  "model": "erh:vmodutils:obstacle",
  "attributes": {
    "geometries": [
      {
        "type": "box",
        "x": 1500,
        "y": 1500,
        "z": 10,
        "label": "ceiling"
      }
    ]
  },
  "frame": {
    "parent": "obstacle-table",
    "translation": { "x": 0, "y": 0, "z": 1000 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

### 3.6 Front Safety Wall

Prevents the arms from reaching toward the operator:

```json
{
  "name": "obstacle-front-wall",
  "api": "rdk:component:gripper",
  "model": "erh:vmodutils:obstacle",
  "attributes": {
    "geometries": [
      {
        "type": "box",
        "x": 10,
        "y": 1045,
        "z": 1000,
        "label": "front-wall"
      }
    ]
  },
  "frame": {
    "parent": "obstacle-table",
    "translation": { "x": 960, "y": 0, "z": 500 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

### 3.7 Bottle Obstacle (Dynamic)

The wine bottle is an obstacle that the arms need to avoid (except when gripping it):

```json
{
  "name": "obstacle-right-bottle",
  "api": "rdk:component:gripper",
  "model": "erh:vmodutils:obstacle",
  "attributes": {
    "geometries": [
      {
        "type": "box",
        "x": 85,
        "y": 85,
        "z": 320,
        "label": "right-bottle"
      }
    ]
  },
  "frame": {
    "parent": "world",
    "translation": { "x": -100, "y": 500, "z": 160 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

### 3.8 Arm Cable Obstacles

The xArm has cables that trail from its base. Define these as obstacles to prevent the other arm from hitting them:

```json
{
  "name": "obstacle-arm-left-cord",
  "api": "rdk:component:gripper",
  "model": "erh:vmodutils:obstacle",
  "attributes": {
    "geometries": [
      {
        "type": "box",
        "x": 50,
        "y": 70,
        "z": 50,
        "label": "arm-left-cord"
      }
    ]
  },
  "frame": {
    "parent": "left-arm",
    "translation": { "x": 0, "y": 80, "z": -10 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

Note this is defined **relative to `left-arm`** - the obstacle moves with the arm's base frame.

## Step 4: Visualize the Frame System

The Viam app includes a frame system visualizer:

1. Go to the **CONFIGURE** tab
2. Click the **Frame** subtab (next to Builder/JSON/History)
3. You should see a 3D visualization of all frames and obstacles

Use this to verify:
- Arm positions are correct
- Obstacles are properly placed
- The workspace boundaries make sense

## Step 5: Test Motion Planning

With frames and obstacles defined, the motion service can plan collision-free paths.

### 5.1 Test via SDK

```python
from viam.services.motion import Motion
from viam.proto.common import Pose, PoseInFrame, WorldState

async def test_motion_planning(robot):
    motion = Motion.from_robot(robot, "builtin")
    
    # Define a target pose for the left arm's gripper
    target_pose = Pose(
        x=300,  # mm forward from world origin
        y=200,  # mm to the left
        z=150,  # mm above table
        o_x=0, o_y=0, o_z=0, theta=0  # orientation
    )
    
    # Create a PoseInFrame (pose relative to a reference frame)
    goal = PoseInFrame(
        reference_frame="world",
        pose=target_pose
    )
    
    # Plan and execute motion
    # The motion service will automatically avoid all obstacles
    await motion.move(
        component_name="left-arm",
        destination=goal,
        world_state=WorldState()  # Uses configured obstacles
    )
    
    print("Motion complete!")
```

If the target position would require passing through an obstacle, the motion planner will find an alternative path or report that no valid path exists.

### 5.2 Test via Control Tab

In the CONTROL tab, the arm panels now show motion planning options:
1. Switch to "Pose" mode (instead of joint positions)
2. Enter target X, Y, Z coordinates
3. Click "Move"

The motion service plans the path, avoiding obstacles.

## Step 6: Frame Hierarchy Summary

Here's the complete frame tree for the wine pouring demo:

```
world
├── left-arm
│   └── obstacle-arm-left-cord
├── right-arm
├── obstacle-table
│   ├── obstacle-front-wall
│   └── obstacle-ceiling
├── obstacle-back-wall
├── obstacle-left-wall
│   └── obstacle-table-corner-left
├── obstacle-right-wall
│   └── obstacle-table-corner-right
└── obstacle-right-bottle
```

## Checkpoint: What You've Accomplished

✅ Configured spatial relationships between arms using the frame system  
✅ Defined world coordinates and arm positions  
✅ Created obstacle geometries for the workspace boundary  
✅ Set up collision avoidance for safe motion planning  
✅ Understood parent-child frame relationships  
✅ Visualized the frame system in the Viam app  

## Measuring Your Workspace

To configure frames for your own setup, you'll need to measure:

1. **Arm positions**: Measure from a common reference point to each arm's base mounting plate
2. **Table dimensions**: Length, width, height above floor
3. **Wall positions**: Distance from reference to each workspace boundary
4. **Obstacle positions**: Any equipment, monitors, or other items in the workspace

Use a tape measure and record measurements in millimeters. It's better to over-estimate obstacle sizes for safety.

## Troubleshooting

### Motion planning fails with "no valid path"
- Check that obstacles aren't blocking the entire path
- Verify arm frame positions are correct
- Try a different target pose to see if any motion works
- Temporarily remove obstacles to isolate the problem

### Arm moves through obstacles
- Obstacles may not be properly positioned
- Check that obstacle geometries are large enough
- Verify the motion service is using the configured WorldState

### Frame visualization looks wrong
- Double-check translation values (X, Y, Z order)
- Verify orientation axis and angle
- Check parent frame assignments

### Arms collide with each other
- The arms themselves have built-in geometry from their URDF models
- The motion planner should avoid arm-to-arm collisions automatically
- If collisions occur, ensure both arms have correct frame positions

## Next Steps

In [Part 4: Training ML Models for Glass Detection](./04-ml-training.md), you'll capture training data and build a machine learning model to detect glasses during the pour.

---

## Reference: Complete Frame and Obstacle Configuration

```json
{
  "components": [
    {
      "name": "left-arm",
      "api": "rdk:component:arm",
      "model": "ufactory:xArm:xArm6",
      "attributes": {
        "host": "192.168.1.200",
        "speed_degs_per_sec": 30,
        "acceleration_degs_per_sec_sec": 50
      },
      "frame": {
        "parent": "world",
        "translation": { "x": 0, "y": 0, "z": 15 },
        "orientation": {
          "type": "ov_degrees",
          "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
        }
      }
    },
    {
      "name": "right-arm",
      "api": "rdk:component:arm",
      "model": "ufactory:xArm:xArm6",
      "attributes": {
        "host": "192.168.1.208",
        "speed_degs_per_sec": 30,
        "acceleration_degs_per_sec_sec": 50
      },
      "frame": {
        "parent": "world",
        "translation": { "x": -18, "y": 830, "z": 15 },
        "orientation": {
          "type": "ov_degrees",
          "value": { "x": 0, "y": 0, "z": 1, "th": 30 }
        }
      }
    },
    {
      "name": "obstacle-table",
      "api": "rdk:component:gripper",
      "model": "erh:vmodutils:obstacle",
      "attributes": {
        "geometries": [{
          "type": "box", "x": 705, "y": 1045, "z": 10, "label": "table"
        }]
      },
      "frame": {
        "parent": "world",
        "translation": { "x": 162, "y": 412, "z": 0 },
        "orientation": {
          "type": "ov_degrees",
          "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
        }
      }
    },
    {
      "name": "obstacle-back-wall",
      "api": "rdk:component:gripper",
      "model": "erh:vmodutils:obstacle",
      "attributes": {
        "geometries": [{
          "type": "box", "x": 40, "y": 1045, "z": 80, "label": "back-wall"
        }]
      },
      "frame": {
        "parent": "world",
        "translation": { "x": -180, "y": 412, "z": 40 },
        "orientation": {
          "type": "ov_degrees",
          "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
        }
      }
    },
    {
      "name": "obstacle-left-wall",
      "api": "rdk:component:gripper",
      "model": "erh:vmodutils:obstacle",
      "attributes": {
        "geometries": [{
          "type": "box", "x": 705, "y": 40, "z": 80, "label": "left-wall"
        }]
      },
      "frame": {
        "parent": "world",
        "translation": { "x": 162, "y": -90, "z": 40 },
        "orientation": {
          "type": "ov_degrees",
          "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
        }
      }
    },
    {
      "name": "obstacle-right-wall",
      "api": "rdk:component:gripper",
      "model": "erh:vmodutils:obstacle",
      "attributes": {
        "geometries": [{
          "type": "box", "x": 705, "y": 40, "z": 80, "label": "right-wall"
        }]
      },
      "frame": {
        "parent": "world",
        "translation": { "x": 162, "y": 920, "z": 40 },
        "orientation": {
          "type": "ov_degrees",
          "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
        }
      }
    },
    {
      "name": "obstacle-ceiling",
      "api": "rdk:component:gripper",
      "model": "erh:vmodutils:obstacle",
      "attributes": {
        "geometries": [{
          "type": "box", "x": 1500, "y": 1500, "z": 10, "label": "ceiling"
        }]
      },
      "frame": {
        "parent": "obstacle-table",
        "translation": { "x": 0, "y": 0, "z": 1000 },
        "orientation": {
          "type": "ov_degrees",
          "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
        }
      }
    },
    {
      "name": "obstacle-front-wall",
      "api": "rdk:component:gripper",
      "model": "erh:vmodutils:obstacle",
      "attributes": {
        "geometries": [{
          "type": "box", "x": 10, "y": 1045, "z": 1000, "label": "front-wall"
        }]
      },
      "frame": {
        "parent": "obstacle-table",
        "translation": { "x": 960, "y": 0, "z": 500 },
        "orientation": {
          "type": "ov_degrees",
          "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
        }
      }
    },
    {
      "name": "obstacle-right-bottle",
      "api": "rdk:component:gripper",
      "model": "erh:vmodutils:obstacle",
      "attributes": {
        "geometries": [{
          "type": "box", "x": 85, "y": 85, "z": 320, "label": "right-bottle"
        }]
      },
      "frame": {
        "parent": "world",
        "translation": { "x": -100, "y": 500, "z": 160 },
        "orientation": {
          "type": "ov_degrees",
          "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
        }
      }
    },
    {
      "name": "obstacle-arm-left-cord",
      "api": "rdk:component:gripper",
      "model": "erh:vmodutils:obstacle",
      "attributes": {
        "geometries": [{
          "type": "box", "x": 50, "y": 70, "z": 50, "label": "arm-left-cord"
        }]
      },
      "frame": {
        "parent": "left-arm",
        "translation": { "x": 0, "y": 80, "z": -10 },
        "orientation": {
          "type": "ov_degrees",
          "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
        }
      }
    }
  ]
}
```
