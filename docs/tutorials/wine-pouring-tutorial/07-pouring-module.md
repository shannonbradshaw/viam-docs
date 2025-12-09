# Part 7: The Pouring Demo Module Deep Dive

In this part, you'll explore the `viam:pouring-demo:vinocart` module - the orchestration brain that ties together all the components into a working demo. You'll understand its architecture and learn patterns for building your own complex robotics applications.

**What you'll learn:**
- How the vinocart service orchestrates the demo
- Using DoCommand for custom operations
- State machine patterns for robotics
- Configuration-driven choreography
- Debugging and extending modules

**Time required:** 1 hour

**Prerequisites:** Complete [Part 6: Motion Planning](./06-motion-planning.md)

## Module Overview

The `viam:pouring-demo` module is a Go-based Viam module that provides the `vinocart` generic service. It:

- Coordinates two arms, cameras, and vision services
- Implements the complete pour workflow
- Provides DoCommand operations for manual control
- Handles error recovery and safety

**Source code:** [github.com/viam-modules/viam-pouring-demo](https://github.com/viam-modules/viam-pouring-demo)

## Step 1: Add the Vinocart Service

### 1.1 Add the Module

```json
{
  "modules": [
    {
      "type": "registry",
      "name": "viam_pouring-demo",
      "module_id": "viam:pouring-demo",
      "version": "latest-with-prerelease"
    }
  ]
}
```

### 1.2 Configure the Service

The vinocart service has extensive configuration:

```json
{
  "name": "cart",
  "api": "rdk:service:generic",
  "model": "viam:pouring-demo:vinocart",
  "attributes": {
    "arm_name": "left-arm",
    "bottle_arm": "right-arm",
    "gripper_name": "left-gripper",
    "bottle_gripper": "right-gripper",
    "camera_name": "left-cam",
    "cropped_cup_camera": "cam-merged-cup",
    "cup_finder_service": "cup-finder-segment",
    "glass_pour_cam": "cam-glass",
    "pour_glass_find_service": "pour-glass-find-service",
    "cup_height": 109,
    "cup_width": 100,
    "bottle_height": 298,
    "glass_pour_motion_threshold": 3,
    "loop": false,
    "handoff": false,
    "positions": {
      "touch": { ... },
      "pour": { ... },
      "pour_prep": { ... },
      "put-back": { ... },
      "reset": { ... }
    }
  },
  "ui_folder": { "name": "app" }
}
```

### 1.3 Configuration Attributes Explained

| Attribute | Type | Description |
|-----------|------|-------------|
| `arm_name` | string | Left arm (handles glass) |
| `bottle_arm` | string | Right arm (handles bottle) |
| `gripper_name` | string | Left gripper component |
| `bottle_gripper` | string | Right gripper component |
| `camera_name` | string | Primary depth camera |
| `cropped_cup_camera` | string | Merged point cloud camera |
| `cup_finder_service` | string | Vision service for 3D cup detection |
| `glass_pour_cam` | string | Camera for pour monitoring |
| `pour_glass_find_service` | string | ML vision service for pour monitoring |
| `cup_height` | int | Glass height in mm |
| `cup_width` | int | Glass width in mm |
| `bottle_height` | int | Bottle height in mm |
| `glass_pour_motion_threshold` | int | Motion detection sensitivity |
| `loop` | bool | Whether to repeat the demo continuously |
| `handoff` | bool | Whether arms hand off the glass |
| `positions` | object | Choreography position sequences |

## Step 2: Understanding DoCommand

The vinocart service is controlled via `DoCommand` - Viam's mechanism for custom operations on generic services.

### 2.1 Available Commands

| Command | Description |
|---------|-------------|
| `{"demo": true}` | Run the full demo sequence |
| `{"touch": true}` | Execute touch/pickup phase only |
| `{"pour-prep": true}` | Execute pour preparation phase |
| `{"pour": true}` | Execute pouring phase |
| `{"put-back": true}` | Execute return phase |
| `{"reset": true}` | Reset arms to starting positions |
| `{"stop": true}` | Emergency stop |
| `{"status": true}` | Get current status |

### 2.2 Calling DoCommand via SDK

```python
from viam.services.generic import Generic

async def run_demo(robot):
    cart = Generic.from_robot(robot, "cart")
    
    # Run the full demo
    result = await cart.do_command({"demo": True})
    print(f"Demo result: {result}")

async def run_phase(robot, phase):
    cart = Generic.from_robot(robot, "cart")
    
    # Run a specific phase
    result = await cart.do_command({phase: True})
    print(f"{phase} result: {result}")

# Usage
await run_phase(robot, "touch")
await run_phase(robot, "pour-prep")
await run_phase(robot, "pour")
await run_phase(robot, "put-back")
```

### 2.3 Calling DoCommand via Control Tab

1. Go to **CONTROL** tab
2. Find the `cart` service
3. In the DoCommand section, enter:
   ```json
   {"demo": true}
   ```
4. Click **Execute**

## Step 3: The Position Choreography System

The `positions` attribute defines all movement sequences:

```json
{
  "positions": {
    "touch": {
      "prep": [
        ["arm-left-watch-pose", "arm-right-watch-pose"]
      ],
      "bad-pick-a": [
        ["arm-left-put-back1"]
      ],
      "bad-pick-b": [
        ["arm-left-put-back2"],
        ["arm-left-watch-pose"]
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
    },
    "pour_prep": {
      "prep-grab": [
        ["arm-left-post-pick", "arm-right-bottle-prep2"]
      ],
      "right-grab": [
        ["arm-right-bottle-right-prep3"],
        ["arm-right-bottle-right-prep4"]
      ],
      "post-grab": [
        ["arm-pour-right-prep", "arm-pour-left-prep"]
      ]
    },
    "put-back": {
      "before-open": [
        ["arm-pour-right-prep", "arm-pour-left-prep"],
        ["arm-right-bottle-right-prep4", "arm-left-put-back1"]
      ],
      "post-open": [
        ["arm-right-bottle-right-prep3", "arm-left-put-back2"],
        ["arm-right-bottle-prep2", "arm-left-put-back3"],
        ["arm-left-watch-pose", "arm-right-watch-pose"]
      ]
    },
    "reset": {
      "right-holding-pre": [["arm-right-bottle-right-prep4"]],
      "right-holding-post": [
        ["arm-right-bottle-right-prep3"],
        ["arm-right-bottle-prep2"],
        ["arm-right-watch-pose"]
      ],
      "left-holding-pre": [["arm-left-put-back1"]],
      "left-holding-post": [
        ["arm-left-put-back2"],
        ["arm-left-put-back3"],
        ["arm-left-watch-pose"]
      ]
    }
  }
}
```

### 3.1 How Sequences Execute

The module processes positions like this:

```go
// Pseudocode from the module
func executeSequence(positions [][]string) {
    for _, step := range positions {
        // Each step is an array of position names
        // All positions in a step execute in PARALLEL
        var wg sync.WaitGroup
        for _, posName := range step {
            wg.Add(1)
            go func(name string) {
                defer wg.Done()
                moveToPosition(name)
            }(posName)
        }
        wg.Wait()  // Wait for all parallel moves to complete
        // Then proceed to next step (SEQUENTIAL)
    }
}
```

### 3.2 Designing Safe Sequences

When creating position sequences:

1. **Parallel moves** must not cause collisions
2. **Sequential steps** provide checkpoints
3. **Recovery positions** handle failure cases

Example safe pattern:
```json
[
  ["arm-left-safe", "arm-right-safe"],     // Both to safe positions
  ["arm-right-approach"],                   // Right approaches alone
  ["arm-left-approach"],                    // Left approaches alone
  ["arm-left-grasp", "arm-right-grasp"]    // Both grasp together
]
```

## Step 4: Module Architecture

### 4.1 Code Structure

```
viam-pouring-demo/
├── cmd/
│   └── module/
│       └── main.go          # Module entry point
├── pour/
│   ├── vinocart.go          # Main service implementation
│   ├── config.go            # Configuration parsing
│   ├── positions.go         # Position execution
│   ├── vision.go            # Vision integration
│   └── pour.go              # Pour logic
├── meta.json                # Module metadata
└── go.mod
```

### 4.2 Key Code Patterns

**Service Registration (main.go):**
```go
func main() {
    utils.ContextualMain(mainWithArgs, module.NewLoggerFromArgs("pouring-demo"))
}

func mainWithArgs(ctx context.Context, args []string, logger logging.Logger) error {
    mod, err := module.NewModuleFromArgs(ctx, logger)
    if err != nil {
        return err
    }
    
    // Register the vinocart model
    err = mod.AddModelFromRegistry(ctx, generic.API, vinocart.Model)
    if err != nil {
        return err
    }
    
    return mod.Start(ctx)
}
```

**DoCommand Handler (vinocart.go):**
```go
func (vc *Vinocart) DoCommand(ctx context.Context, cmd map[string]interface{}) (map[string]interface{}, error) {
    if _, ok := cmd["demo"]; ok {
        return vc.runDemo(ctx)
    }
    if _, ok := cmd["touch"]; ok {
        return vc.runTouch(ctx)
    }
    if _, ok := cmd["pour"]; ok {
        return vc.runPour(ctx)
    }
    if _, ok := cmd["stop"]; ok {
        return vc.stop(ctx)
    }
    // ... more commands
    
    return nil, errors.New("unknown command")
}
```

**Vision Integration (vision.go):**
```go
func (vc *Vinocart) findCups(ctx context.Context) ([]r3.Vector, error) {
    // Get 3D detections from the segmentation service
    objects, err := vc.cupFinder.GetObjectPointClouds(ctx, vc.croppedCupCamera, nil)
    if err != nil {
        return nil, err
    }
    
    var positions []r3.Vector
    for _, obj := range objects {
        if len(obj.Geometries) > 0 {
            center := obj.Geometries[0].Pose().Point()
            positions = append(positions, center)
        }
    }
    
    return positions, nil
}
```

### 4.3 State Management

The module tracks state for recovery:

```go
type Vinocart struct {
    // Components
    leftArm      arm.Arm
    rightArm     arm.Arm
    leftGripper  gripper.Gripper
    rightGripper gripper.Gripper
    
    // State
    leftHolding  bool    // Is left arm holding something?
    rightHolding bool    // Is right arm holding something?
    stopped      bool    // Emergency stop active?
    currentPhase string  // Current demo phase
    
    // Config
    positions    map[string]map[string][][]string
    cupHeight    float64
    cupWidth     float64
}
```

## Step 5: Running the Demo

### 5.1 Full Demo Sequence

```python
async def run_full_demo(robot):
    cart = Generic.from_robot(robot, "cart")
    
    print("Starting wine pouring demo...")
    
    try:
        result = await cart.do_command({"demo": True})
        print(f"Demo completed: {result}")
    except Exception as e:
        print(f"Demo failed: {e}")
        # Attempt reset
        await cart.do_command({"reset": True})
```

### 5.2 Step-by-Step Execution

For debugging, run each phase separately:

```python
async def run_demo_stepwise(robot):
    cart = Generic.from_robot(robot, "cart")
    
    phases = ["touch", "pour-prep", "pour", "put-back"]
    
    for phase in phases:
        input(f"Press Enter to execute '{phase}'...")
        try:
            result = await cart.do_command({phase: True})
            print(f"{phase}: {result}")
        except Exception as e:
            print(f"{phase} failed: {e}")
            break
    
    print("Demo complete or stopped")
```

### 5.3 Monitoring Status

```python
async def monitor_demo(robot):
    cart = Generic.from_robot(robot, "cart")
    
    while True:
        status = await cart.do_command({"status": True})
        print(f"Phase: {status.get('phase', 'unknown')}")
        print(f"Left holding: {status.get('left_holding', False)}")
        print(f"Right holding: {status.get('right_holding', False)}")
        await asyncio.sleep(1)
```

## Step 6: Error Handling and Recovery

### 6.1 The Reset Command

When things go wrong, the reset command attempts to safely return to a known state:

```python
async def safe_reset(robot):
    cart = Generic.from_robot(robot, "cart")
    
    # First, stop any motion
    await cart.do_command({"stop": True})
    
    # Wait for motion to cease
    await asyncio.sleep(1)
    
    # Execute reset sequence
    result = await cart.do_command({"reset": True})
    print(f"Reset result: {result}")
```

### 6.2 Recovery Positions

The `reset` section of positions handles different states:

- `right-holding-pre/post`: Right arm has bottle, needs to return it
- `left-holding-pre/post`: Left arm has glass, needs to return it

### 6.3 Common Failure Modes

| Failure | Symptom | Recovery |
|---------|---------|----------|
| Glass not detected | Touch phase fails | Check lighting, camera alignment |
| Grip failure | Arm moves but glass stays | Adjust gripper force, glass position |
| Motion planning failure | Arm doesn't move | Check obstacles, try different approach |
| Pour detection failure | Pours too long/short | Adjust `glass_pour_motion_threshold` |

## Step 7: Extending the Module

### 7.1 Adding New Commands

To add a custom command, modify `DoCommand`:

```go
func (vc *Vinocart) DoCommand(ctx context.Context, cmd map[string]interface{}) (map[string]interface{}, error) {
    // Add your custom command
    if _, ok := cmd["calibrate"]; ok {
        return vc.runCalibration(ctx)
    }
    // ... existing commands
}

func (vc *Vinocart) runCalibration(ctx context.Context) (map[string]interface{}, error) {
    // Your calibration logic
    return map[string]interface{}{"calibrated": true}, nil
}
```

### 7.2 Adding New Position Phases

Add to the config:
```json
{
  "positions": {
    "custom-phase": {
      "step1": [["position-a", "position-b"]],
      "step2": [["position-c"]]
    }
  }
}
```

Then reference in code or call via DoCommand with the phase name.

### 7.3 Building and Testing Locally

```bash
# Clone the repo
git clone https://github.com/viam-modules/viam-pouring-demo.git
cd viam-pouring-demo

# Build
make build

# Test locally
./bin/pouring-demo --config /path/to/test-config.json
```

## Checkpoint: What You've Accomplished

✅ Added and configured the vinocart service  
✅ Understood DoCommand for custom operations  
✅ Learned the position choreography system  
✅ Explored the module's code architecture  
✅ Ran the demo in full and step-by-step modes  
✅ Understood error handling and recovery  

## Troubleshooting

### Service doesn't start
- Check all referenced components exist
- Verify position saver names match exactly
- Review logs for configuration errors

### DoCommand returns error
- Check command spelling
- Verify prerequisites (e.g., arms in right positions)
- Look for detailed error in logs

### Demo stops mid-sequence
- Check for vision detection failures
- Verify arm positions are reachable
- Look for motion planning errors

### Arms don't coordinate properly
- Verify position sequences are safe
- Check parallel moves don't collide
- Test positions individually first

## Next Steps

In [Part 8: Fragments for Reusable Configuration](./08-fragments.md), you'll learn how to package this entire configuration for reuse and deployment across multiple machines.

---

## Reference: Complete Vinocart Configuration

```json
{
  "services": [
    {
      "name": "cart",
      "api": "rdk:service:generic",
      "model": "viam:pouring-demo:vinocart",
      "attributes": {
        "arm_name": "left-arm",
        "bottle_arm": "right-arm",
        "gripper_name": "left-gripper",
        "bottle_gripper": "right-gripper",
        "camera_name": "left-cam",
        "cropped_cup_camera": "cam-merged-cup",
        "cup_finder_service": "cup-finder-segment",
        "glass_pour_cam": "cam-glass",
        "pour_glass_find_service": "pour-glass-find-service",
        "cup_height": 109,
        "cup_width": 100,
        "bottle_height": 298,
        "glass_pour_motion_threshold": 3,
        "loop": false,
        "handoff": false,
        "positions": {
          "touch": {
            "prep": [["arm-left-watch-pose", "arm-right-watch-pose"]],
            "bad-pick-a": [["arm-left-put-back1"]],
            "bad-pick-b": [["arm-left-put-back2"], ["arm-left-watch-pose"]]
          },
          "pour": {
            "prep": [
              ["arm-pour-left-prep", "arm-pour-right-prep"],
              ["arm-pour-right-pos0"]
            ],
            "finish": [["arm-pour-left-prep", "arm-pour-right-prep"]]
          },
          "pour_prep": {
            "prep-grab": [["arm-left-post-pick", "arm-right-bottle-prep2"]],
            "right-grab": [
              ["arm-right-bottle-right-prep3"],
              ["arm-right-bottle-right-prep4"]
            ],
            "post-grab": [["arm-pour-right-prep", "arm-pour-left-prep"]]
          },
          "put-back": {
            "before-open": [
              ["arm-pour-right-prep", "arm-pour-left-prep"],
              ["arm-right-bottle-right-prep4", "arm-left-put-back1"]
            ],
            "post-open": [
              ["arm-right-bottle-right-prep3", "arm-left-put-back2"],
              ["arm-right-bottle-prep2", "arm-left-put-back3"],
              ["arm-left-watch-pose", "arm-right-watch-pose"]
            ]
          },
          "reset": {
            "right-holding-pre": [["arm-right-bottle-right-prep4"]],
            "right-holding-post": [
              ["arm-right-bottle-right-prep3"],
              ["arm-right-bottle-prep2"],
              ["arm-right-watch-pose"]
            ],
            "left-holding-pre": [["arm-left-put-back1"]],
            "left-holding-post": [
              ["arm-left-put-back2"],
              ["arm-left-put-back3"],
              ["arm-left-watch-pose"]
            ]
          }
        }
      },
      "ui_folder": { "name": "app" }
    }
  ],
  "modules": [
    {
      "type": "registry",
      "name": "viam_pouring-demo",
      "module_id": "viam:pouring-demo",
      "version": "latest-with-prerelease"
    }
  ]
}
```
