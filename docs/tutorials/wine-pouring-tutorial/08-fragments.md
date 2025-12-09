# Part 8: Fragments for Reusable Configuration

In this part, you'll learn how to package your robot configuration into reusable fragments. This enables you to deploy the wine pouring demo to multiple machines and manage configuration centrally.

**What you'll learn:**
- Creating fragments from machine configurations
- Using fragment variables for site-specific values
- Applying fragment modifications (mods)
- Managing fragment versions
- Deploying fragments to multiple machines

**Time required:** 45 minutes

**Prerequisites:** Complete [Part 7: Pouring Module Deep Dive](./07-pouring-module.md)

## Why Fragments?

As you've seen, the wine pouring demo configuration is complex - arms, cameras, vision services, obstacles, and orchestration. Fragments solve several problems:

1. **Reusability**: Deploy the same configuration to multiple machines
2. **Maintainability**: Update once, deploy everywhere
3. **Customization**: Override specific values per-machine
4. **Versioning**: Track configuration changes over time

## Step 1: Understanding the Fragment Structure

The demo uses multiple fragments:

```
Fragment 1: Hardware Configuration
├── Left arm (with IP variable)
├── Right arm (with IP variable)
├── Left RealSense (with serial variable)
├── Right RealSense (with serial variable)
├── Glass webcam (with video path variable)
└── Basic services

Fragment 2: Application Configuration
├── Obstacle definitions
├── Vision pipeline
├── Position savers
├── Vinocart service
└── ML model packages
```

This separation allows:
- Hardware fragment: Changes per physical setup
- Application fragment: Same for all deployments

## Step 2: Create a Hardware Fragment

### 2.1 Extract Hardware Configuration

From your working machine, copy the hardware-related JSON:

```json
{
  "components": [
    {
      "name": "left-arm",
      "api": "rdk:component:arm",
      "model": "ufactory:xArm:xArm6",
      "attributes": {
        "host": {
          "$variable": { "name": "left-arm-ip-address" }
        },
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
        "host": {
          "$variable": { "name": "right-arm-ip-address" }
        },
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
      "name": "left-cam",
      "api": "rdk:component:camera",
      "model": "viam:camera:realsense",
      "attributes": {
        "sensors": ["color", "depth"],
        "width_px": 640,
        "height_px": 480,
        "serial": {
          "$variable": { "name": "left-cam-serial" }
        }
      }
    },
    {
      "name": "right-cam",
      "api": "rdk:component:camera",
      "model": "viam:camera:realsense",
      "attributes": {
        "sensors": ["color", "depth"],
        "width_px": 640,
        "height_px": 480,
        "serial": {
          "$variable": { "name": "right-cam-serial" }
        }
      }
    },
    {
      "name": "cam-glass",
      "api": "rdk:component:camera",
      "model": "rdk:builtin:webcam",
      "attributes": {
        "height_px": 2160,
        "video_path": {
          "$variable": { "name": "glass-cam-video-path" }
        }
      }
    }
  ],
  "modules": [
    {
      "type": "registry",
      "name": "ufactory_xArm",
      "module_id": "ufactory:xArm",
      "version": "latest"
    },
    {
      "type": "registry",
      "name": "viam_realsense",
      "module_id": "viam:realsense",
      "version": "latest-with-prerelease"
    }
  ]
}
```

### 2.2 Create the Fragment

1. Go to **FLEET** → **FRAGMENTS**
2. Click **Create fragment**
3. Name it `wine-demo-hardware`
4. Paste the JSON configuration
5. Click **Save**

### 2.3 Understanding Variables

Notice the `$variable` syntax:
```json
{
  "host": {
    "$variable": { "name": "left-arm-ip-address" }
  }
}
```

This creates a placeholder that must be filled when the fragment is used. Variables allow:
- Different IP addresses per deployment
- Different camera serial numbers
- Site-specific customization

## Step 3: Create an Application Fragment

### 3.1 Extract Application Configuration

Create a second fragment with:
- Obstacle definitions
- Vision services
- Position savers
- Vinocart service
- Cropped cameras

```json
{
  "components": [
    {
      "name": "obstacle-table",
      "api": "rdk:component:gripper",
      "model": "erh:vmodutils:obstacle",
      "attributes": {
        "geometries": [{
          "type": "box",
          "x": 705,
          "y": 1045,
          "z": 10,
          "label": "table"
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
    // ... more obstacles
    {
      "name": "cam-left-cup-crop",
      "api": "rdk:component:camera",
      "model": "erh:vmodutils:pc-crop-camera",
      "attributes": {
        "src": {
          "$variable": { "name": "left-camera" }
        },
        "min": { "X": 225, "Y": 120, "Z": 40 },
        "max": { "X": 410, "Y": 740, "Z": 130 }
      }
    },
    {
      "name": "cam-right-cup-crop",
      "api": "rdk:component:camera",
      "model": "erh:vmodutils:pc-crop-camera",
      "attributes": {
        "src": {
          "$variable": { "name": "right-camera" }
        },
        "min": { "X": 225, "Y": 120, "Z": 40 },
        "max": { "X": 410, "Y": 740, "Z": 130 }
      }
    },
    {
      "name": "cam-merged-cup",
      "api": "rdk:component:camera",
      "model": "erh:vmodutils:pc-merge",
      "attributes": {
        "cameras": ["cam-left-cup-crop", "cam-right-cup-crop"]
      }
    }
  ],
  "services": [
    {
      "name": "cup-finder-segment",
      "api": "rdk:service:vision",
      "model": "viam:obstacles-pointcloud:obstacles-pointcloud",
      "attributes": {
        "camera_name": "cam-merged-cup",
        "min_points_in_plane": 100000000,
        "min_points_in_segment": 100,
        "clustering_strictness": 0.3,
        "ground_plane_normal_vec": { "z": 1 }
      }
    },
    {
      "name": "cart",
      "api": "rdk:service:generic",
      "model": "viam:pouring-demo:vinocart",
      "attributes": {
        "arm_name": {
          "$variable": { "name": "left-arm-name" }
        },
        "bottle_arm": {
          "$variable": { "name": "right-arm-name" }
        },
        "cup_height": {
          "$variable": { "name": "cup-height" }
        },
        "cup_width": {
          "$variable": { "name": "cup-width" }
        },
        "bottle_height": {
          "$variable": { "name": "bottle-height" }
        },
        // ... more config
      }
    }
  ],
  "modules": [
    {
      "type": "registry",
      "name": "erh_vmodutils",
      "module_id": "erh:vmodutils",
      "version": "latest-with-prerelease"
    },
    {
      "type": "registry",
      "name": "viam_pouring-demo",
      "module_id": "viam:pouring-demo",
      "version": "latest-with-prerelease"
    }
  ]
}
```

### 3.2 Save as Fragment

1. Go to **FLEET** → **FRAGMENTS**
2. Click **Create fragment**
3. Name it `wine-demo-application`
4. Paste the JSON
5. Click **Save**

## Step 4: Use Fragments in a Machine

### 4.1 Add Fragments to Machine Configuration

In your machine's configuration, reference the fragments:

```json
{
  "fragments": [
    {
      "id": "YOUR_HARDWARE_FRAGMENT_ID",
      "variables": {
        "left-arm-ip-address": "192.168.1.200",
        "right-arm-ip-address": "192.168.1.208",
        "left-cam-serial": "022422071219",
        "right-cam-serial": "243222071262",
        "glass-cam-video-path": "video12"
      }
    },
    {
      "id": "YOUR_APPLICATION_FRAGMENT_ID",
      "variables": {
        "left-arm-name": "left-arm",
        "right-arm-name": "right-arm",
        "left-camera": "left-cam",
        "right-camera": "right-cam",
        "cup-height": 109,
        "cup-width": 100,
        "bottle-height": 298,
        "left-gripper-name": "left-gripper",
        "right-gripper-name": "right-gripper",
        "glass-cam-name": "cam-glass",
        "left-cam-name": "left-cam"
      }
    }
  ]
}
```

### 4.2 Via the UI

1. Go to **CONFIGURE** tab
2. Click **+** → **Insert fragment**
3. Select your hardware fragment
4. Fill in the variable values
5. Repeat for the application fragment
6. Click **Save**

## Step 5: Fragment Modifications (Mods)

Sometimes you need to override specific values in a fragment without changing the fragment itself. Fragment mods allow this.

### 5.1 Understanding Mods

The demo uses a mod to adjust the right arm's frame:

```json
{
  "fragment_mods": [
    {
      "fragment_id": "797ab2d0-91b7-48d7-b1ec-1510fe35216d",
      "mods": [
        {
          "$set": {
            "components.right-arm.frame": {
              "translation": {
                "x": -18,
                "y": 830,
                "z": 15
              },
              "orientation": {
                "value": {
                  "x": 0,
                  "y": 0,
                  "z": 1,
                  "th": 30
                },
                "type": "ov_degrees"
              },
              "parent": "world"
            }
          }
        }
      ]
    }
  ]
}
```

### 5.2 Mod Operations

| Operation | Description | Example |
|-----------|-------------|---------|
| `$set` | Set/replace a value | `{"$set": {"components.arm.attributes.speed": 50}}` |
| `$unset` | Remove a field | `{"$unset": {"components.arm.attributes.debug"}}` |

### 5.3 When to Use Mods vs Variables

**Use Variables when:**
- The value differs between ALL deployments (IP addresses, serials)
- The fragment author anticipated the customization

**Use Mods when:**
- Only SOME deployments need changes
- The fragment doesn't have a variable for what you need
- You're fixing a bug without updating the fragment

## Step 6: Fragment Versioning

### 6.1 Automatic Versioning

Every time you save a fragment, Viam creates a new version. View version history:

1. Go to **FLEET** → **FRAGMENTS**
2. Click on your fragment
3. Click **Versions** tab

### 6.2 Creating Tags

Tags mark specific versions for reference:

1. In the Versions tab, click **Add Tag**
2. Name it (e.g., `stable`, `v1.0`, `beta`)
3. Select the version to tag

### 6.3 Using Tagged Versions

Machines can pin to specific fragment versions:

```json
{
  "fragments": [
    {
      "id": "YOUR_FRAGMENT_ID",
      "version": "stable",  // Use tagged version
      "variables": { ... }
    }
  ]
}
```

Options:
- Omit `version`: Always use latest
- `"version": "stable"`: Use tagged version
- `"version": "2024-01-15T10:30:00Z"`: Use specific timestamp

## Step 7: Deploying to Multiple Machines

### 7.1 New Machine Setup

For each new wine pouring setup:

1. Create a new machine in Viam
2. Install viam-server
3. Add the fragments with site-specific variables:

```json
{
  "fragments": [
    {
      "id": "HARDWARE_FRAGMENT_ID",
      "variables": {
        "left-arm-ip-address": "192.168.2.100",  // Different IP
        "right-arm-ip-address": "192.168.2.101",
        "left-cam-serial": "111111111111",       // Different serial
        "right-cam-serial": "222222222222",
        "glass-cam-video-path": "video0"         // Different path
      }
    },
    {
      "id": "APPLICATION_FRAGMENT_ID",
      "variables": { ... }
    }
  ]
}
```

### 7.2 Updating All Machines

When you update a fragment:
1. Edit the fragment in **FLEET** → **FRAGMENTS**
2. Save changes
3. All machines using that fragment automatically update

For controlled rollouts:
1. Update fragment and tag as `beta`
2. Test on one machine with `"version": "beta"`
3. When verified, move `stable` tag to new version
4. Other machines update on next restart

## Step 8: Best Practices

### 8.1 Fragment Organization

```
Organization Fragments/
├── wine-demo-hardware-v1        # Hardware configuration
├── wine-demo-application-v1     # Application logic
├── wine-demo-ml-models-v1       # ML model packages
└── wine-demo-positions-v1       # Position definitions
```

### 8.2 Variable Naming

Use consistent, descriptive names:
- `left-arm-ip-address` ✓ (clear)
- `arm1_ip` ✗ (ambiguous)
- `camera-serial-left-realsense` ✓ (specific)
- `cam_sn` ✗ (unclear)

### 8.3 Documentation

Add fragment descriptions:
```json
{
  "description": "Hardware configuration for wine pouring demo. Variables: left-arm-ip-address (str): IP of left xArm..."
}
```

## Checkpoint: What You've Accomplished

✅ Created reusable fragments from machine configuration  
✅ Used variables for site-specific customization  
✅ Applied fragment modifications for overrides  
✅ Understood fragment versioning and tagging  
✅ Deployed configuration to multiple machines  
✅ Learned best practices for fragment organization  

## Troubleshooting

### Fragment not applying
- Check fragment ID is correct
- Verify all required variables are provided
- Look for JSON syntax errors
- Review viam-server logs

### Variables not substituting
- Ensure `$variable` syntax is correct
- Variable names are case-sensitive
- Check the variable is in the `variables` object

### Mod not working
- Path syntax must be exact: `components.name.field`
- Use dots for nested objects
- Check for typos in field names

### Version conflicts
- Clear browser cache if UI shows old version
- Restart viam-server to pick up changes
- Check machine isn't pinned to old version

## Next Steps

🎉 **Congratulations!** You've completed the core tutorial series!

Optional: Continue to [Part 9: StreamDeck Integration](./09-streamdeck.md) to add physical controls to your robot.

---

## Reference: Complete Fragment Setup

**Machine Configuration with Fragments:**
```json
{
  "fragments": [
    {
      "id": "797ab2d0-91b7-48d7-b1ec-1510fe35216d",
      "variables": {
        "right-cam-serial": "243222071262",
        "left-arm-ip-address": "192.168.1.200",
        "left-cam-serial": "022422071219",
        "glass-cam-video-path": "video12",
        "right-arm-ip-address": "192.168.1.208"
      }
    },
    {
      "id": "4d5dfec3-955a-49ea-8525-3d59bd4b2bee",
      "variables": {
        "right-arm-name": "right-arm",
        "cup-height": 109,
        "bottle-height": 298,
        "left-gripper-name": "left-gripper",
        "cup-width": 100,
        "right-gripper-name": "right-gripper",
        "right-camera": "right-cam",
        "left-arm-name": "left-arm",
        "glass-cam-name": "cam-glass",
        "left-camera": "left-cam",
        "left-cam-name": "left-cam"
      }
    }
  ],
  "fragment_mods": [
    {
      "fragment_id": "797ab2d0-91b7-48d7-b1ec-1510fe35216d",
      "mods": [
        {
          "$set": {
            "components.right-arm.frame": {
              "translation": { "x": -18, "y": 830, "z": 15 },
              "orientation": {
                "value": { "x": 0, "y": 0, "z": 1, "th": 30 },
                "type": "ov_degrees"
              },
              "parent": "world"
            }
          }
        }
      ]
    }
  ]
}
```
