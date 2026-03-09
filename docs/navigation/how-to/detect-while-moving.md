---
linkTitle: "Detect While Moving"
title: "Detect Objects While Navigating"
weight: 60
layout: "docs"
type: "docs"
description: "Run vision services concurrently with navigation to detect and react to objects."
---

## What Problem This Solves

Your rover needs to detect objects (people, packages, anomalies) while
navigating its route and take action when something is found -- stopping,
approaching the object, or logging the detection.

## Prerequisites

- [Navigation service configured](/navigation/reference/navigation-service/)
- Vision service configured for detection
- Camera configured

## Steps

### 1. Set up concurrent detection

Run a detection loop alongside the navigation service.

{{< tabs >}}
{{% tab name="Python" %}}

```python
import asyncio
from viam.services.vision import VisionClient
from viam.services.navigation import NavigationClient
from viam.proto.service.navigation import Mode

nav = NavigationClient.from_robot(machine, "my-navigation")
vision = VisionClient.from_robot(machine, "my-detector")

async def detection_loop():
    """Continuously check for objects."""
    while True:
        detections = await vision.get_detections_from_camera("my-camera")
        for d in detections:
            if d.confidence > 0.7:
                print(f"Detected {d.class_name} "
                      f"(confidence: {d.confidence:.2f})")
                # Take action based on detection
                await handle_detection(d)
        await asyncio.sleep(0.5)

async def handle_detection(detection):
    """React to a detection."""
    if detection.class_name == "person":
        # Stop navigation when a person is detected
        await nav.set_mode(mode=Mode.MODE_MANUAL)
        print("Stopped: person detected")
        await asyncio.sleep(5)
        # Resume after pause
        await nav.set_mode(mode=Mode.MODE_WAYPOINT)
        print("Resuming navigation")

async def navigate_route():
    """Navigate through waypoints."""
    from viam.proto.common import GeoPoint

    waypoints = [
        GeoPoint(latitude=40.7128, longitude=-74.0060),
        GeoPoint(latitude=40.7130, longitude=-74.0058),
    ]
    for wp in waypoints:
        await nav.add_waypoint(point=wp)

    await nav.set_mode(mode=Mode.MODE_WAYPOINT)

    while True:
        remaining = await nav.get_waypoints()
        if len(remaining) == 0:
            break
        await asyncio.sleep(2)

    print("Route complete")

# Run both concurrently
async def main():
    await asyncio.gather(
        navigate_route(),
        detection_loop()
    )
```

{{% /tab %}}
{{< /tabs >}}

### 2. Log detections with location

Combine GPS position with detections to record where objects were found.

{{< tabs >}}
{{% tab name="Python" %}}

```python
async def log_detection_with_location(detection):
    location = await nav.get_location()
    print(f"Detected {detection.class_name} at "
          f"({location.latitude:.6f}, {location.longitude:.6f})")
```

{{% /tab %}}
{{< /tabs >}}

## What's Next

- [Navigate to a Waypoint](/navigation/how-to/navigate-to-waypoint/) -- basic
  navigation setup.
- [Avoid Obstacles](/navigation/how-to/avoid-obstacles/) -- automatic obstacle
  avoidance during navigation.
