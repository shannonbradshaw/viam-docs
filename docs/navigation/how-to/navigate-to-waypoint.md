---
linkTitle: "Navigate to Waypoint"
title: "Navigate to a Waypoint"
weight: 20
layout: "docs"
type: "docs"
description: "Configure the navigation service and autonomously navigate to GPS waypoints."
aliases:
  - /tutorials/services/navigate-with-rover-base/
---

## What Problem This Solves

You want your rover to drive itself to a GPS coordinate without manual control.
The navigation service handles the full loop: reading GPS, planning a path,
sending movement commands, and monitoring progress.

## Prerequisites

- Base with motors configured
- Movement sensor providing GPS position and compass heading
- [Navigation service configured](/navigation/reference/navigation-service/)

## Steps

### 1. Add waypoints

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.services.navigation import NavigationClient
from viam.proto.common import GeoPoint

nav = NavigationClient.from_robot(machine, "my-navigation")

# Add a waypoint
waypoint = GeoPoint(latitude=40.7128, longitude=-74.0060)
await nav.add_waypoint(point=waypoint)
print("Waypoint added")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
import (
    "go.viam.com/rdk/services/navigation"
    "github.com/kellydunn/golang-geo"
)

nav, err := navigation.FromRobot(machine, "my-navigation")
if err != nil {
    logger.Fatal(err)
}

point := geo.NewPoint(40.7128, -74.0060)
err = nav.AddWaypoint(ctx, point, nil)
if err != nil {
    logger.Fatal(err)
}
```

{{% /tab %}}
{{< /tabs >}}

### 2. Start navigation

Set the mode to Waypoint to begin autonomous navigation.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.service.navigation import Mode

await nav.set_mode(mode=Mode.MODE_WAYPOINT)
print("Navigation started")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
err = nav.SetMode(ctx, navigation.ModeWaypoint, nil)
if err != nil {
    logger.Fatal(err)
}
fmt.Println("Navigation started")
```

{{% /tab %}}
{{< /tabs >}}

### 3. Monitor progress

{{< tabs >}}
{{% tab name="Python" %}}

```python
import asyncio

while True:
    location = await nav.get_location()
    waypoints = await nav.get_waypoints()
    print(f"Current: ({location.latitude:.6f}, {location.longitude:.6f})")
    print(f"Remaining waypoints: {len(waypoints)}")
    if len(waypoints) == 0:
        print("All waypoints reached")
        break
    await asyncio.sleep(2)
```

{{% /tab %}}
{{< /tabs >}}

### 4. Stop navigation

{{< tabs >}}
{{% tab name="Python" %}}

```python
await nav.set_mode(mode=Mode.MODE_MANUAL)
print("Navigation stopped")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
err = nav.SetMode(ctx, navigation.ModeManual, nil)
```

{{% /tab %}}
{{< /tabs >}}

## Using the Control tab

You can also add waypoints and control navigation from the Viam app:

1. Navigate to your machine's **CONTROL** tab.
2. Expand the navigation service card.
3. Click on the map to add waypoints.
4. Use the mode buttons to start/stop navigation.

## What's Next

- [Follow a Patrol Route](/navigation/how-to/follow-a-patrol-route/) --
  navigate through a sequence of waypoints in a loop.
- [Avoid Obstacles](/navigation/how-to/avoid-obstacles/) -- configure
  obstacle detection during navigation.
