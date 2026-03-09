---
linkTitle: "Avoid Obstacles"
title: "Avoid Obstacles During Navigation"
weight: 40
layout: "docs"
type: "docs"
description: "Configure vision-based obstacle detection for autonomous navigation."
---

## What Problem This Solves

Your rover is navigating autonomously but needs to detect and avoid obstacles
in its path -- objects not known at configuration time that appear during
operation.

## Prerequisites

- [Navigation service configured](/navigation/reference/navigation-service/)
- Camera configured
- Vision service configured for obstacle detection

## Steps

### 1. Configure obstacle detectors

Add obstacle detectors to your navigation service configuration. Each detector
pairs a vision service with a camera:

```json
{
  "name": "my-navigation",
  "api": "rdk:service:navigation",
  "model": "rdk:builtin:builtin",
  "attributes": {
    "base": "my-base",
    "movement_sensor": "my-gps",
    "store": { "type": "memory" },
    "obstacle_detectors": [
      {
        "vision_service": "my-obstacle-detector",
        "camera": "my-camera"
      }
    ],
    "obstacle_polling_frequency_hz": 2,
    "plan_deviation_m": 1.0
  }
}
```

### 2. Configure the vision service

The vision service must return detections or classifications that indicate
obstacles. A common approach is to use a 3D segmenter that returns point clouds
for detected objects.

```json
{
  "name": "my-obstacle-detector",
  "api": "rdk:service:vision",
  "model": "obstacles_depth",
  "attributes": {}
}
```

### 3. Tune detection parameters

| Parameter | Effect |
|-----------|--------|
| `obstacle_polling_frequency_hz` | How often to check for obstacles. Higher values detect obstacles sooner but use more CPU. Default: 1 Hz. |
| `plan_deviation_m` | How far the rover can deviate from its planned path. Smaller values keep the rover closer to the plan but may cause frequent replanning. Default: 2.6 m. |

### 4. Add static obstacles

For known obstacles at GPS coordinates, add them to the navigation service
configuration:

```json
{
  "obstacles": [
    {
      "geometries": [
        {
          "type": "box",
          "x": 1000,
          "y": 1000,
          "z": 500,
          "label": "building"
        }
      ],
      "location": {
        "latitude": 40.7128,
        "longitude": -74.0060
      }
    }
  ]
}
```

## What's Next

- [Navigate to a Waypoint](/navigation/how-to/navigate-to-waypoint/) -- basic
  autonomous navigation.
- [Detect While Moving](/navigation/how-to/detect-while-moving/) -- run
  vision concurrently with navigation.
