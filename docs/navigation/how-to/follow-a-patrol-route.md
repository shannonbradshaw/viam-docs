---
linkTitle: "Follow a Patrol Route"
title: "Follow a Patrol Route"
weight: 30
layout: "docs"
type: "docs"
description: "Define a sequence of waypoints and navigate them in a loop."
---

## What Problem This Solves

Your rover needs to patrol a defined route repeatedly -- for example, security
monitoring, facility inspection, or agricultural survey.

## Prerequisites

- [Navigation service configured](/navigation/reference/navigation-service/)
- GPS position and compass heading working

## Steps

### 1. Define the patrol route

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.services.navigation import NavigationClient
from viam.proto.common import GeoPoint
from viam.proto.service.navigation import Mode

nav = NavigationClient.from_robot(machine, "my-navigation")

# Define waypoints for the patrol route
patrol_points = [
    GeoPoint(latitude=40.7128, longitude=-74.0060),
    GeoPoint(latitude=40.7130, longitude=-74.0058),
    GeoPoint(latitude=40.7132, longitude=-74.0060),
    GeoPoint(latitude=40.7130, longitude=-74.0062),
]
```

{{% /tab %}}
{{< /tabs >}}

### 2. Run the patrol loop

{{< tabs >}}
{{% tab name="Python" %}}

```python
import asyncio

patrol_count = 0
while True:
    # Add all waypoints
    for point in patrol_points:
        await nav.add_waypoint(point=point)

    # Start navigation
    await nav.set_mode(mode=Mode.MODE_WAYPOINT)

    # Wait until all waypoints are reached
    while True:
        waypoints = await nav.get_waypoints()
        if len(waypoints) == 0:
            break
        await asyncio.sleep(2)

    patrol_count += 1
    print(f"Patrol {patrol_count} complete")

    # Brief pause before next patrol
    await asyncio.sleep(5)
```

{{% /tab %}}
{{< /tabs >}}

## What's Next

- [Avoid Obstacles](/navigation/how-to/avoid-obstacles/) -- add obstacle
  detection to your patrol route.
- [Detect While Moving](/navigation/how-to/detect-while-moving/) -- run
  vision alongside navigation.
