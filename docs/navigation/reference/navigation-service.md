---
linkTitle: "Navigation Service"
title: "Navigation Service Configuration"
weight: 10
layout: "docs"
type: "docs"
description: "Configure the navigation service for autonomous GPS-based waypoint navigation."
aliases:
  - /reference/services/navigation/
  - /services/navigation/
  - /mobility/navigation/
---

The navigation service uses GPS to autonomously navigate a rover base to
user-defined waypoints. It is a stateful service that manages the full
navigation loop: reading GPS position, planning paths between waypoints,
sending movement commands to the base, and optionally detecting and avoiding
obstacles.

## Requirements

You must configure:

- A [base](/reference/components/base/) that implements `SetVelocity()`,
  `GetGeometries()`, and `GetProperties()`
- A [movement sensor](/reference/components/movement-sensor/) that implements
  `GetPosition()`, `GetCompassHeading()`, and `GetProperties()`

## Configuration

```json
{
  "name": "my-navigation",
  "api": "rdk:service:navigation",
  "model": "rdk:builtin:builtin",
  "attributes": {
    "store": {
      "type": "memory"
    },
    "base": "my-base",
    "movement_sensor": "my-gps",
    "obstacle_detectors": [
      {
        "vision_service": "my-vision",
        "camera": "my-camera"
      }
    ]
  }
}
```

### Attributes

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `store` | object | **Yes** | `"memory"` | Waypoint storage. `"type": "memory"` for in-memory (lost on restart) or `"type": "mongodb"` with a `"config": {"uri": "..."}` for persistent storage. |
| `base` | string | **Yes** | -- | Name of the base component. |
| `movement_sensor` | string | **Yes** | -- | Name of the movement sensor (must provide GPS position and compass heading). |
| `motion_service` | string | No | `"builtin"` | Name of the motion service to use. |
| `obstacle_detectors` | array | No | (none) | Vision service + camera pairs for dynamic obstacle detection. |
| `position_polling_frequency_hz` | float | No | 1 | How often to poll GPS position, in Hz. |
| `obstacle_polling_frequency_hz` | float | No | 1 | How often to poll vision services for obstacles, in Hz. |
| `plan_deviation_m` | float | No | 2.6 | Maximum allowed deviation from the plan, in meters. |
| `degs_per_sec` | float | No | 20 | Angular velocity for the base, in degrees per second. |
| `meters_per_sec` | float | No | 0.3 | Linear velocity for the base, in meters per second. |
| `obstacles` | array | No | (none) | Static obstacles with GPS coordinates. |
| `bounding_regions` | array | No | (none) | Geographic bounds the robot must stay within. |

## Navigation modes

| Mode | Behavior |
|------|----------|
| **Manual** (0) | Service is configured but does not drive. Use for setup and testing. |
| **Waypoint** (1) | Drives to each waypoint in sequence. |
| **Explore** (2) | Drives in the direction of waypoints while exploring. |

Set the mode using the `SetMode` API method.

## Frame system setup

For accurate GPS navigation, configure frames for your base and movement sensor:

1. Add a frame to your base with parent `"world"` and a geometry that matches
   its physical dimensions.
2. Add a frame to your movement sensor with parent set to the base name. Set the
   `translation` to reflect where the sensor is mounted on the base.
3. Calibrate: in the **CONTROL** tab, verify the compass heading reads `0` when
   the base faces north. Adjust the movement sensor frame orientation if needed.

## API

The [navigation service API](/reference/apis/services/navigation/) supports the
following methods:

{{< readfile "/static/include/services/apis/generated/navigation-table.md" >}}

## Control tab

After configuring the navigation service, use the **CONTROL** tab to:

- Add waypoints on a map
- Add obstacles at GPS coordinates
- View the rover's current position
- Start and stop navigation

## What's Next

- [Navigate to a Waypoint](/navigation/how-to/navigate-to-waypoint/) -- set up
  waypoints and start autonomous navigation.
- [Drive the Base](/navigation/how-to/drive-the-base/) -- direct base control
  for manual movement.
- [Avoid Obstacles](/navigation/how-to/avoid-obstacles/) -- configure
  vision-based obstacle avoidance.
