# Part 1: Hardware Setup & Arm Configuration

In this first part, you'll set up the physical hardware and configure your first Viam machine with two robotic arms. By the end, you'll be able to control both arms from the Viam app.

**What you'll learn:**
- Creating a machine in Viam
- Installing viam-server on Linux
- Configuring UFactory xArm 6 arms as components
- Understanding modules and the Viam Registry
- Testing arm movements safely

**Time required:** 1-2 hours

## Step 1: Physical Hardware Setup

### 1.1 Arm Placement and Mounting

Mount both xArm 6 robots to a stable surface. The arms should be:
- Positioned to have overlapping reach zones for handoff operations
- Spaced approximately 830mm apart (center to center)
- Mounted at the same height

For the wine pouring demo, we use this layout:
- **Left arm** (at 192.168.1.200): Handles glass pickup and positioning
- **Right arm** (at 192.168.1.208): Handles bottle grip and pouring

### 1.2 Network Configuration

Each xArm has a control box with an Ethernet port. Configure static IPs:

1. Connect a laptop directly to each arm's control box
2. Access the xArm Studio software at the default IP (192.168.1.xxx)
3. Navigate to **Settings → Network** and configure:
   - Left arm: `192.168.1.200`, subnet `255.255.255.0`
   - Right arm: `192.168.1.208`, subnet `255.255.255.0`
4. Connect both arms to your network switch
5. Connect your Linux computer to the same switch

Verify connectivity:
```bash
ping 192.168.1.200
ping 192.168.1.208
```

### 1.3 Camera Connections

Connect the cameras to your Linux computer:
- **Left RealSense D435**: USB 3.0 port
- **Right RealSense D435**: USB 3.0 port  
- **Glass detection webcam**: USB port

Note the serial numbers of your RealSense cameras (printed on the device, or find them with `rs-enumerate-devices`). You'll need these for configuration.

## Step 2: Create Your Machine in Viam

### 2.1 Sign Up and Create Organization

1. Go to [app.viam.com](https://app.viam.com) and create an account
2. Create an organization for your project (e.g., "Wine Robot Lab")
3. Create a location within that organization (e.g., "Demo Station 1")

### 2.2 Create a New Machine

1. Click **+ Add machine**
2. Name it `wine-pouring-robot` (or your preferred name)
3. Click **Create**

You'll see setup instructions with an installation command. Keep this page open.

## Step 3: Install viam-server

On your Linux computer, run the installation command shown in the Viam app. It looks like:

```bash
curl -fsSL https://storage.googleapis.com/packages.viam.com/apps/viam-server/viam-server-stable-aarch64.AppImage -o viam-server
chmod 755 viam-server
sudo ./viam-server --aio-install
```

The exact command varies based on your architecture. Use the command from your machine's setup page.

After installation, `viam-server` runs as a system service. Verify it's running:

```bash
sudo systemctl status viam-server
```

Your machine should now show as **Connected** in the Viam app.

## Step 4: Configure the Left Arm

Now we'll add the first arm as a component.

### 4.1 Add the xArm Module

1. In the Viam app, go to your machine's **CONFIGURE** tab
2. Click the **+** button in the left panel
3. Select **Component**
4. Search for `xArm6` and select `arm / ufactory:xArm6`
5. Click **Add module** when prompted
6. Name the component `left-arm`
7. Click **Create**

### 4.2 Configure the Arm Attributes

In the configuration panel for `left-arm`, set these attributes:

```json
{
  "host": "192.168.1.200",
  "speed_degs_per_sec": 30,
  "acceleration_degs_per_sec_sec": 50
}
```

**Attribute explanations:**
- `host`: The IP address of the arm's control box
- `speed_degs_per_sec`: Maximum joint speed (start low for safety!)
- `acceleration_degs_per_sec_sec`: Maximum joint acceleration

### 4.3 Save and Test

1. Click **Save** in the top right
2. Wait for the configuration to apply (watch the logs)
3. Go to the **CONTROL** tab
4. Find the `left-arm` panel
5. You should see the current joint positions

**First test - Get current position:**
The arm's current joint positions display in radians. Note these values - they represent where your arm physically is right now.

**Second test - Small movement:**

> ⚠️ **SAFETY WARNING**: Before moving the arm:
> - Clear the area around the arm
> - Keep the emergency stop accessible
> - Start with very small movements
> - Have someone ready to cut power if needed

In the TEST panel, try a small joint movement:
1. Note the current J1 (base rotation) value
2. Change it by 0.1 radians (about 5.7 degrees)
3. Click **Move**
4. The arm should rotate slightly at the base

If the arm moves as expected, your first arm is configured correctly!

## Step 5: Configure the Right Arm

Repeat the process for the second arm:

1. Click **+** → **Component**
2. Search for `xArm6` and select `arm / ufactory:xArm6`
3. Name it `right-arm`
4. Click **Create**

Configure attributes:
```json
{
  "host": "192.168.1.208",
  "speed_degs_per_sec": 30,
  "acceleration_degs_per_sec_sec": 50
}
```

Save, then test in the CONTROL tab.

## Step 6: Understanding What You've Built

Let's look at the JSON configuration you've created. Click **JSON** in the top left of the CONFIGURE tab:

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
      }
    }
  ],
  "modules": [
    {
      "type": "registry",
      "name": "ufactory_xArm",
      "module_id": "ufactory:xArm",
      "version": "latest"
    }
  ]
}
```

### Key Concepts

**Components** are the resources your machine controls - motors, cameras, arms, sensors, etc. Each component has:
- `name`: A unique identifier you choose
- `api`: The type of component (arm, camera, motor, etc.)
- `model`: The specific driver implementation
- `attributes`: Configuration specific to that model

**Modules** are packages that provide component models. The xArm module:
- Lives in the Viam Registry
- Downloads automatically when you add the component
- Provides the `ufactory:xArm:xArm6` model
- Communicates with the physical arm hardware

**The Viam Registry** ([app.viam.com/registry](https://app.viam.com/registry)) is a package repository for:
- Hardware drivers (camera modules, motor controllers, arm drivers)
- Software services (ML inference, SLAM, vision processing)
- Complete applications

Anyone can publish modules to the registry - this is how the robotics community shares reusable components.

## Step 7: Configure Grippers (Optional but Recommended)

If your xArms have UFactory grippers attached, you can control them through DoCommand. However, for explicit gripper control, you may want to add gripper components.

The xArm module supports gripper operations through the arm's DoCommand:

```python
# Example: Open gripper on left arm
await left_arm.do_command({
    "setup_gripper": True,
    "move_gripper": 850  # 0=closed, 850=fully open
})
```

We'll configure explicit gripper components in Part 3 when we set up the frame system.

## Step 8: Basic Arm Control via SDK

Let's write a simple Python script to control the arms programmatically.

### 8.1 Install the Viam SDK

```bash
pip install viam-sdk
```

### 8.2 Get Connection Credentials

1. In the Viam app, go to the **CONNECT** tab
2. Select **Python** 
3. Copy the connection code

### 8.3 Create a Test Script

Create `test_arms.py`:

```python
import asyncio
from viam.robot.client import RobotClient
from viam.components.arm import Arm

async def connect():
    opts = RobotClient.Options.with_api_key(
        api_key='YOUR_API_KEY',
        api_key_id='YOUR_API_KEY_ID'
    )
    return await RobotClient.at_address('YOUR_ROBOT_ADDRESS', opts)

async def main():
    robot = await connect()
    
    # Get arm components
    left_arm = Arm.from_robot(robot, "left-arm")
    right_arm = Arm.from_robot(robot, "right-arm")
    
    # Read current positions
    left_positions = await left_arm.get_joint_positions()
    right_positions = await right_arm.get_joint_positions()
    
    print(f"Left arm joints (radians): {left_positions.values}")
    print(f"Right arm joints (radians): {right_positions.values}")
    
    # Get end effector pose
    left_pose = await left_arm.get_end_position()
    print(f"Left arm end effector: x={left_pose.x:.1f}, y={left_pose.y:.1f}, z={left_pose.z:.1f}")
    
    await robot.close()

if __name__ == "__main__":
    asyncio.run(main())
```

Run it:
```bash
python test_arms.py
```

You should see the joint positions and end effector location for both arms.

## Checkpoint: What You've Accomplished

✅ Physically set up two xArm 6 robots with network connectivity  
✅ Created a Viam machine and installed viam-server  
✅ Configured both arms as components using the xArm module  
✅ Tested arm movements through the Viam app  
✅ Connected to the arms programmatically via SDK  

## Troubleshooting

### Arm shows as disconnected
- Verify network connectivity: `ping 192.168.1.200`
- Check that the arm's control box is powered on
- Ensure the arm is not in an error state (check xArm Studio)
- Review viam-server logs: `sudo journalctl -u viam-server -f`

### Module fails to load
- Check internet connectivity (modules download from the registry)
- Verify the module name and version are correct
- Look for error messages in the LOGS tab

### Arm doesn't move when commanded
- Check the arm is not in manual mode (xArm Studio)
- Verify no protective stops are active
- Try re-enabling the arm: power cycle the control box
- Ensure `speed_degs_per_sec` is not set to 0

### "Connection refused" errors
- The arm's TCP port (502) might be blocked
- Another application might be connected to the arm
- Check firewall settings on your Linux computer

## Next Steps

In [Part 2: Depth Cameras & Point Clouds](./02-cameras-point-clouds.md), you'll add the Intel RealSense cameras and learn how to work with depth data for object detection.

---

## Reference: Full Configuration So Far

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
      }
    }
  ],
  "modules": [
    {
      "type": "registry",
      "name": "ufactory_xArm",
      "module_id": "ufactory:xArm",
      "version": "latest"
    }
  ]
}
```
