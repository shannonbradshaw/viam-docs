---
linkTitle: "Arm"
title: "Add an arm"
weight: 5
layout: "docs"
type: "docs"
description: "Add and configure a robotic arm, verify joint motion, and test end-effector positioning."
date: "2025-03-07"
aliases:
  - /hardware-components/add-an-arm/
---

You have a robotic arm and you need Viam to control it. The arm API supports moving joints,
positioning the end effector, and integrating with motion planning.

## Concepts

An arm component controls a multi-jointed robotic arm. The API provides:

- **Joint-level control**: move individual joints to specific angles.
- **End-effector positioning**: move the end effector to a pose in 3D space
  (Viam handles the inverse kinematics).
- **Motion planning integration**: the motion service can plan
  collision-free paths for the arm.

Arm models almost always come from **modules in the registry** because each
arm manufacturer has its own communication protocol. Common models include:

| Model | Arms supported |
|-------|---------------|
| `viam:ufactory:xArm6` | UFactory xArm 5, xArm 6, xArm 7, Lite 6 |
| `viam:universalrobots:ur5e` | Universal Robots UR3, UR5, UR10, UR16 |

The `fake` built-in model is useful for testing code without physical hardware.
It supports kinematics for several arm models (ur5e, xarm6, xarm7, lite6).

## Steps

### 1. Add an arm component

1. Click the **+** button.
2. Select **Configuration block**.
3. Search for the model that matches your arm manufacturer and model:
   - For a UFactory xArm, search for **xArm6** (or xArm5, xArm7, Lite6).
   - For a Universal Robots arm, search for **ur5e** (or ur3e, ur10e, ur16e).
4. Click **Add component**, name your arm (e.g., `my-arm`), and click
   **Add component** again to confirm.

### 2. Configure arm attributes

Attributes vary by arm module. Most network-connected arms need a host
address:

**UFactory xArm 6:**

```json
{
  "host": "192.168.1.100",
  "speed_degs_per_sec": 30,
  "acceleration_degs_per_sec_per_sec": 100
}
```

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `host` | string | Yes | IP address of the arm on your network. |
| `speed_degs_per_sec` | float | No | Default joint speed. |
| `acceleration_degs_per_sec_per_sec` | float | No | Default joint acceleration. |

Check your module's documentation in the registry for the full list of
attributes.

### 3. Configure a frame (recommended)

If you plan to use motion planning or coordinate with other components (like a
camera or gripper), add a frame to define the arm's position in your workspace. The frame definition below is typical for a single are configuration:

```json
{
  "frame": {
    "parent": "world",
    "translation": { "x": 0, "y": 0, "z": 0 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

See [Frame System](/motion-planning/frame-system/) for details on configuring frames.

### 4. Save and test

Click **Save**, then expand the **TEST** section.

- Use the joint position controls to move individual joints.
- Verify each joint moves in the expected direction.
- Start with slow speeds until you're confident in the configuration.

{{< alert title="Safety" color="caution" >}}

Robotic arms can move with significant force. Keep the work area clear, start
with low speeds, and have an e-stop accessible. Never reach into the arm's
workspace while it's in motion.

{{< /alert >}}

## Testing with a fake arm

To develop and test code without physical hardware, use the `fake` built-in
model:

```json
{
  "arm-model": "xarm6"
}
```

The fake arm simulates kinematics for the specified model. It responds to all
API calls and reports joint positions, but doesn't move anything physical.

## Try it

Read the arm's current joint positions and move a single joint.

{{< tabs >}}
{{% tab name="Python" %}}

```bash
pip install viam-sdk
```

Save this as `arm_test.py`:

```python
import asyncio
from viam.robot.client import RobotClient
from viam.components.arm import Arm


async def main():
    opts = RobotClient.Options.with_api_key(
        api_key="YOUR-API-KEY",
        api_key_id="YOUR-API-KEY-ID"
    )
    robot = await RobotClient.at_address("YOUR-MACHINE-ADDRESS", opts)

    arm = Arm.from_robot(robot, "my-arm")

    # Get current joint positions
    joints = await arm.get_joint_positions()
    print(f"Current joint positions: {joints.values}")

    # Get end effector pose
    pose = await arm.get_end_position()
    print(f"End effector at: x={pose.x:.1f}, y={pose.y:.1f}, z={pose.z:.1f}")

    # Check if arm is moving
    moving = await arm.is_moving()
    print(f"Arm is moving: {moving}")

    await robot.close()

if __name__ == "__main__":
    asyncio.run(main())
```

Run it:

```bash
python arm_test.py
```

You should see the current joint angles (in degrees) and the end effector's
position in millimeters.

{{% /tab %}}
{{% tab name="Go" %}}

```bash
mkdir arm-test && cd arm-test
go mod init arm-test
go get go.viam.com/rdk
```

Save this as `main.go`:

```go
package main

import (
	"context"
	"fmt"

	"go.viam.com/rdk/components/arm"
	"go.viam.com/rdk/logging"
	"go.viam.com/rdk/robot/client"
	"go.viam.com/rdk/utils"
)

func main() {
	ctx := context.Background()
	logger := logging.NewLogger("arm-test")

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

	a, err := arm.FromRobot(robot, "my-arm")
	if err != nil {
		logger.Fatal(err)
	}

	// Get current joint positions
	joints, err := a.JointPositions(ctx, nil)
	if err != nil {
		logger.Fatal(err)
	}
	fmt.Printf("Current joint positions: %v\n", joints.Values)

	// Get end effector pose
	pose, err := a.EndPosition(ctx, nil)
	if err != nil {
		logger.Fatal(err)
	}
	fmt.Printf("End effector at: x=%.1f, y=%.1f, z=%.1f\n",
		pose.Point().X, pose.Point().Y, pose.Point().Z)

	// Check if arm is moving
	moving, err := a.IsMoving(ctx)
	if err != nil {
		logger.Fatal(err)
	}
	fmt.Printf("Arm is moving: %v\n", moving)
}
```

Run it:

```bash
go run main.go
```

{{% /tab %}}
{{< /tabs >}}

Replace the placeholder values with your API key, API key ID, and machine
address from the Viam app's **CONNECT** tab.

## Troubleshooting

{{< expand "Cannot connect to arm" >}}

- Verify the arm is powered on and connected to the same network as your
  machine.
- Ping the arm's IP address from the machine to confirm network connectivity.
- Check that no other software (manufacturer's own control software) has an
  exclusive connection to the arm.

{{< /expand >}}

{{< expand "Arm moves but positions are wrong" >}}

- Verify the frame configuration matches the arm's physical placement.
- If using motion planning, check that the kinematics model matches your
  arm. Using the wrong model causes incorrect inverse kinematics.

{{< /expand >}}

{{< expand "Motion planning fails" >}}

- Ensure the arm has a frame configured with `parent: "world"`.
- Check that the target pose is within the arm's reachable workspace.
- See [Obstacles](/motion-planning/obstacles/) for configuring obstacles and
  [Constraints](/motion-planning/constraints/) for motion constraints.

{{< /expand >}}

## What's next

- [Arm API reference](/dev/reference/apis/components/arm/): full method documentation.
- [Fragments](/hardware/fragments/): save and reuse working
  hardware configurations.
- [Motion Planning](/motion-planning/): configure frames, kinematics, and
  obstacles for motion planning.
- [What is a module?](/build-modules/from-hardware-to-logic/): write a module that
  coordinates the arm with sensors.
