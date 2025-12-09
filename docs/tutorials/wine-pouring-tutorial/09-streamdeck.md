# Part 9: StreamDeck Integration (Optional)

In this optional part, you'll add physical button controls to your wine pouring robot using an Elgato StreamDeck. This provides a tactile interface for running demos without needing the Viam app.

**What you'll learn:**
- Configuring the StreamDeck module
- Mapping buttons to DoCommand operations
- Customizing button appearance
- Creating intuitive control panels

**Time required:** 30 minutes

**Prerequisites:** 
- Complete [Part 7: Pouring Module Deep Dive](./07-pouring-module.md)
- Elgato StreamDeck (Original, Mini, or XL)

## Why StreamDeck?

For demo situations, a physical control panel offers:
- **Immediate access**: No need to navigate the app
- **Visual feedback**: Button colors indicate state
- **Reliability**: Works even if network is slow
- **Professional appearance**: Clean demo presentation

## Step 1: Hardware Setup

### 1.1 Connect StreamDeck

1. Connect the StreamDeck to your robot's computer via USB
2. Verify detection:
   ```bash
   lsusb | grep Elgato
   ```
   
You should see something like:
```
Bus 001 Device 005: ID 0fd9:0060 Elgato Systems GmbH Stream Deck
```

### 1.2 Install Prerequisites

The StreamDeck module requires hidapi:
```bash
sudo apt-get install libhidapi-hidraw0 libhidapi-libusb0
```

Set up udev rules for non-root access:
```bash
sudo tee /etc/udev/rules.d/99-streamdeck.rules << EOF
SUBSYSTEM=="usb", ATTRS{idVendor}=="0fd9", MODE="0666"
EOF
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Unplug and reconnect the StreamDeck.

## Step 2: Add the StreamDeck Module

### 2.1 Add Module to Configuration

```json
{
  "modules": [
    {
      "type": "registry",
      "name": "erh_viam-streamdeck",
      "module_id": "erh:viam-streamdeck",
      "version": "latest-with-prerelease"
    }
  ]
}
```

### 2.2 Add the StreamDeck Service

```json
{
  "services": [
    {
      "name": "buttons",
      "api": "rdk:service:generic",
      "model": "erh:viam-streamdeck:streamdeck-original",
      "attributes": {
        "brightness": 100,
        "keys": []
      },
      "ui_folder": { "name": "app" }
    }
  ]
}
```

**Model options:**
- `streamdeck-original`: 15 buttons (3×5)
- `streamdeck-mini`: 6 buttons (2×3)
- `streamdeck-xl`: 32 buttons (4×8)

## Step 3: Configure Button Mappings

### 3.1 Button Layout (StreamDeck Original)

```
┌─────┬─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │  4  │
├─────┼─────┼─────┼─────┼─────┤
│  5  │  6  │  7  │  8  │  9  │
├─────┼─────┼─────┼─────┼─────┤
│ 10  │ 11  │ 12  │ 13  │ 14  │
└─────┴─────┴─────┴─────┴─────┘
```

### 3.2 Demo Button Configuration

Here's the button layout for the wine demo:

```json
{
  "keys": [
    {
      "key": 0,
      "text": "full demo",
      "color": "green",
      "component": "cart",
      "method": "do_command",
      "args": [{ "demo": true }]
    },
    {
      "key": 5,
      "text": "touch",
      "color": "purple",
      "component": "cart",
      "method": "do_command",
      "args": [{ "touch": true }]
    },
    {
      "key": 6,
      "text": "prep",
      "color": "purple",
      "component": "cart",
      "method": "do_command",
      "args": [{ "pour-prep": true }]
    },
    {
      "key": 7,
      "text": "pour",
      "color": "purple",
      "component": "cart",
      "method": "do_command",
      "args": [{ "pour": true }]
    },
    {
      "key": 8,
      "text": "put-back",
      "color": "purple",
      "component": "cart",
      "method": "do_command",
      "args": [{ "put-back": true }]
    },
    {
      "key": 10,
      "text": "reset",
      "color": "orange",
      "component": "cart",
      "method": "do_command",
      "args": [{ "reset": true }]
    },
    {
      "key": 14,
      "image": "stopsign.jpg",
      "component": "cart",
      "method": "do_command",
      "args": [{ "stop": true }]
    }
  ]
}
```

### 3.3 Visual Layout

```
┌─────────┬─────────┬─────────┬─────────┬─────────┐
│  FULL   │         │         │         │         │
│  DEMO   │         │         │         │         │
│ (green) │         │         │         │         │
├─────────┼─────────┼─────────┼─────────┼─────────┤
│  TOUCH  │  PREP   │  POUR   │PUT-BACK │         │
│(purple) │(purple) │(purple) │(purple) │         │
├─────────┼─────────┼─────────┼─────────┼─────────┤
│  RESET  │         │         │         │  STOP   │
│(orange) │         │         │         │  🛑     │
└─────────┴─────────┴─────────┴─────────┴─────────┘
```

## Step 4: Button Configuration Options

### 4.1 Basic Button

```json
{
  "key": 0,
  "text": "button label",
  "color": "green",
  "component": "component-name",
  "method": "do_command",
  "args": [{ "command": true }]
}
```

### 4.2 Available Colors

- `red`
- `green`
- `blue`
- `yellow`
- `orange`
- `purple`
- `white`
- `black`

### 4.3 Custom Images

Instead of text and color, use an image:

```json
{
  "key": 14,
  "image": "stopsign.jpg",
  "component": "cart",
  "method": "do_command",
  "args": [{ "stop": true }]
}
```

Images should be:
- 72×72 pixels (StreamDeck Original)
- JPEG or PNG format
- Placed in a location the module can access

### 4.4 Methods

| Method | Description | Args Format |
|--------|-------------|-------------|
| `do_command` | Call DoCommand on service | `[{command_object}]` |
| `set_position` | Activate a switch | `[position_number]` |

## Step 5: Advanced Button Configurations

### 5.1 Arm Position Buttons

Add buttons to move arms to specific positions:

```json
{
  "key": 1,
  "text": "left\\nstow",
  "color": "blue",
  "component": "left-stow",
  "method": "set_position",
  "args": [1]
}
```

### 5.2 Multi-line Text

Use `\\n` for line breaks:
```json
{
  "text": "full\\ndemo"
}
```

### 5.3 Complete Demo Layout

```json
{
  "keys": [
    {
      "key": 0,
      "text": "FULL\\nDEMO",
      "color": "green",
      "component": "cart",
      "method": "do_command",
      "args": [{ "demo": true }]
    },
    {
      "key": 1,
      "text": "left\\nstow",
      "color": "blue",
      "component": "left-stow",
      "method": "set_position",
      "args": [1]
    },
    {
      "key": 2,
      "text": "right\\nstow",
      "color": "blue",
      "component": "right-stow",
      "method": "set_position",
      "args": [1]
    },
    {
      "key": 5,
      "text": "touch",
      "color": "purple",
      "component": "cart",
      "method": "do_command",
      "args": [{ "touch": true }]
    },
    {
      "key": 6,
      "text": "prep",
      "color": "purple",
      "component": "cart",
      "method": "do_command",
      "args": [{ "pour-prep": true }]
    },
    {
      "key": 7,
      "text": "pour",
      "color": "purple",
      "component": "cart",
      "method": "do_command",
      "args": [{ "pour": true }]
    },
    {
      "key": 8,
      "text": "put\\nback",
      "color": "purple",
      "component": "cart",
      "method": "do_command",
      "args": [{ "put-back": true }]
    },
    {
      "key": 10,
      "text": "RESET",
      "color": "orange",
      "component": "cart",
      "method": "do_command",
      "args": [{ "reset": true }]
    },
    {
      "key": 14,
      "text": "STOP",
      "color": "red",
      "component": "cart",
      "method": "do_command",
      "args": [{ "stop": true }]
    }
  ]
}
```

## Step 6: Test the StreamDeck

### 6.1 Verify Configuration

1. Save your configuration
2. Check viam-server logs for StreamDeck initialization:
   ```bash
   sudo journalctl -u viam-server -f | grep -i streamdeck
   ```

### 6.2 Test Buttons

1. The StreamDeck should light up with your configured buttons
2. Press the **RESET** button first (arms to safe position)
3. Test individual phases before running full demo

### 6.3 Button Press Feedback

When a button is pressed:
- The module calls the configured method
- Log output shows the operation
- No visual feedback on the button (it stays the same color)

## Step 7: Demo Operation Flow

### 7.1 Starting a Demo

1. Ensure robot is powered and connected
2. Press **RESET** to initialize arm positions
3. Place a glass on the table
4. Press **FULL DEMO** to run automatically

### 7.2 Manual Phase Control

For step-by-step operation:
1. Press **RESET**
2. Place glass
3. Press **TOUCH** (picks up glass)
4. Press **PREP** (positions for pour)
5. Press **POUR** (tilts bottle)
6. Press **PUT BACK** (returns glass)

### 7.3 Emergency Stop

Press **STOP** at any time to:
- Halt all arm motion
- Cancel current operation
- Require **RESET** before continuing

## Step 8: Creating Custom Images

### 8.1 Image Requirements

For StreamDeck Original:
- Size: 72×72 pixels
- Format: JPEG or PNG
- Location: Accessible path on the robot's filesystem

### 8.2 Example: Stop Sign Image

Create a simple stop sign:

```python
from PIL import Image, ImageDraw

# Create 72x72 red image
img = Image.new('RGB', (72, 72), color='red')
draw = ImageDraw.Draw(img)

# Add white border
draw.rectangle([4, 4, 67, 67], outline='white', width=3)

# Save
img.save('/home/user/robot-images/stopsign.jpg')
```

Reference in config:
```json
{
  "image": "/home/user/robot-images/stopsign.jpg"
}
```

## Checkpoint: What You've Accomplished

✅ Connected and configured StreamDeck hardware  
✅ Mapped buttons to DoCommand operations  
✅ Created an intuitive control panel layout  
✅ Learned button customization options  
✅ Tested physical control of the demo  

## Troubleshooting

### StreamDeck not detected
- Check USB connection
- Verify udev rules are installed
- Check `lsusb` output
- Try different USB port

### Buttons don't light up
- Check module loaded (view logs)
- Verify model matches your StreamDeck type
- Check configuration syntax

### Button press doesn't work
- Verify component name is correct
- Check method spelling
- Ensure args format matches expected input
- Review viam-server logs for errors

### Wrong button layout
- Key numbers start at 0
- Top-left is key 0
- Verify your StreamDeck model in config

## Alternative Control Options

If you don't have a StreamDeck, consider:

### Keyboard Shortcuts
Use a Python script with keyboard library:
```python
import keyboard

keyboard.add_hotkey('f1', lambda: run_demo())
keyboard.add_hotkey('f2', lambda: run_touch())
keyboard.add_hotkey('esc', lambda: emergency_stop())
```

### Web Interface
Create a simple web UI using the TypeScript SDK:
```typescript
const button = document.getElementById('demo-btn');
button.onclick = async () => {
  await cart.doCommand({ demo: true });
};
```

### Mobile App
Use the Flutter SDK to build a custom mobile control app.

---

## 🎉 Tutorial Complete!

Congratulations! You've built a complete dual-arm wine pouring robot with:

- ✅ Two coordinated xArm 6 robotic arms
- ✅ Intel RealSense depth cameras for 3D perception
- ✅ ML-powered glass detection
- ✅ Collision-free motion planning
- ✅ Choreographed pour sequences
- ✅ Reusable fragment-based configuration
- ✅ Physical StreamDeck controls

## Where to Go From Here

### Extend the Demo
- Add multiple pour levels (half glass, full glass)
- Detect when bottle is empty
- Support multiple bottle types
- Add speech feedback

### Learn More Viam
- [SLAM and Navigation](https://docs.viam.com/services/slam/)
- [Fleet Management](https://docs.viam.com/fleet/)
- [Create Your Own Module](https://docs.viam.com/registry/create/)

### Join the Community
- [Viam Discord](https://discord.gg/viam)
- [GitHub Discussions](https://github.com/viamrobotics/rdk/discussions)
- [Viam YouTube](https://youtube.com/@viamrobotics)

---

## Reference: Complete StreamDeck Configuration

```json
{
  "services": [
    {
      "name": "buttons",
      "api": "rdk:service:generic",
      "model": "erh:viam-streamdeck:streamdeck-original",
      "attributes": {
        "brightness": 100,
        "keys": [
          {
            "key": 0,
            "text": "FULL\\nDEMO",
            "color": "green",
            "component": "cart",
            "method": "do_command",
            "args": [{ "demo": true }]
          },
          {
            "key": 1,
            "text": "left\\nstow",
            "color": "blue",
            "component": "left-stow",
            "method": "set_position",
            "args": [1]
          },
          {
            "key": 2,
            "text": "right\\nstow",
            "color": "blue",
            "component": "right-stow",
            "method": "set_position",
            "args": [1]
          },
          {
            "key": 5,
            "text": "touch",
            "color": "purple",
            "component": "cart",
            "method": "do_command",
            "args": [{ "touch": true }]
          },
          {
            "key": 6,
            "text": "prep",
            "color": "purple",
            "component": "cart",
            "method": "do_command",
            "args": [{ "pour-prep": true }]
          },
          {
            "key": 7,
            "text": "pour",
            "color": "purple",
            "component": "cart",
            "method": "do_command",
            "args": [{ "pour": true }]
          },
          {
            "key": 8,
            "text": "put\\nback",
            "color": "purple",
            "component": "cart",
            "method": "do_command",
            "args": [{ "put-back": true }]
          },
          {
            "key": 10,
            "text": "RESET",
            "color": "orange",
            "component": "cart",
            "method": "do_command",
            "args": [{ "reset": true }]
          },
          {
            "key": 14,
            "text": "STOP",
            "color": "red",
            "component": "cart",
            "method": "do_command",
            "args": [{ "stop": true }]
          }
        ]
      },
      "ui_folder": { "name": "app" }
    }
  ],
  "modules": [
    {
      "type": "registry",
      "name": "erh_viam-streamdeck",
      "module_id": "erh:viam-streamdeck",
      "version": "latest-with-prerelease"
    }
  ]
}
```
