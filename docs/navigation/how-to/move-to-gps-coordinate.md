---
linkTitle: "Move to GPS Coordinate"
title: "Move to a GPS Coordinate"
weight: 50
layout: "docs"
type: "docs"
description: "Move a base to a specific GPS coordinate using MoveOnGlobe."
---

## What Problem This Solves

You want to move your rover to a specific GPS coordinate without setting up the
full navigation service. The motion service's `MoveOnGlobe` method provides a
single-call way to do this.

## Important limitation

`MoveOnGlobe` is **not supported** by the builtin motion service. It requires
a motion service module that implements globe-based movement, or you can use the
[navigation service](/navigation/how-to/navigate-to-waypoint/) which handles
GPS navigation natively.

For most GPS navigation use cases, the navigation service is the recommended
approach.

## Using the navigation service instead

If `MoveOnGlobe` is not available, use the navigation service to achieve the
same result:

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.services.navigation import NavigationClient
from viam.proto.common import GeoPoint
from viam.proto.service.navigation import Mode

nav = NavigationClient.from_robot(machine, "my-navigation")

# Add a single waypoint at the target coordinate
target = GeoPoint(latitude=40.7128, longitude=-74.0060)
await nav.add_waypoint(point=target)

# Start navigation
await nav.set_mode(mode=Mode.MODE_WAYPOINT)

# Wait for arrival
import asyncio
while True:
    waypoints = await nav.get_waypoints()
    if len(waypoints) == 0:
        print("Arrived at target coordinate")
        break
    await asyncio.sleep(2)

# Return to manual mode
await nav.set_mode(mode=Mode.MODE_MANUAL)
```

{{% /tab %}}
{{< /tabs >}}

## What's Next

- [Navigate to a Waypoint](/navigation/how-to/navigate-to-waypoint/) -- full
  waypoint navigation setup.
- [Navigation Service Configuration](/navigation/reference/navigation-service/)
  -- configuration reference.
