---
linkTitle: "Drive the Base"
title: "Drive a Mobile Base"
weight: 10
layout: "docs"
type: "docs"
description: "Control a mobile robot base with direct movement commands."
---

## What Problem This Solves

You need to move your mobile robot using direct commands -- driving forward,
spinning, or setting velocities manually. This is useful for teleoperation,
testing, and simple movement tasks that don't require autonomous navigation.

## Steps

### 1. Get the base client

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.components.base import Base

base = Base.from_robot(machine, "my-base")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
import "go.viam.com/rdk/components/base"

myBase, err := base.FromRobot(machine, "my-base")
if err != nil {
    logger.Fatal(err)
}
```

{{% /tab %}}
{{< /tabs >}}

### 2. Drive straight

{{< tabs >}}
{{% tab name="Python" %}}

```python
# Move forward 500mm at 500mm/s
await base.move_straight(velocity=500, distance=500)
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
// Move forward 500mm at 500mm/s
err = myBase.MoveStraight(ctx, 500, 500.0, nil)
```

{{% /tab %}}
{{< /tabs >}}

### 3. Spin

{{< tabs >}}
{{% tab name="Python" %}}

```python
# Spin 90 degrees at 100 degrees/sec
await base.spin(velocity=100, angle=90)
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
// Spin 90 degrees at 100 degrees/sec
err = myBase.Spin(ctx, 90, 100.0, nil)
```

{{% /tab %}}
{{< /tabs >}}

### 4. Set velocity (continuous control)

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.common import Vector3

# Drive forward and turn right
await base.set_velocity(
    linear=Vector3(x=0, y=200, z=0),    # 200mm/s forward
    angular=Vector3(x=0, y=0, z=-30),   # 30 deg/s clockwise
)

import asyncio
await asyncio.sleep(3)

# Stop
await base.stop()
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
// Drive forward and turn right
err = myBase.SetVelocity(ctx,
    r3.Vector{X: 0, Y: 200, Z: 0},    // 200mm/s forward
    r3.Vector{X: 0, Y: 0, Z: -30},    // 30 deg/s clockwise
    nil)

time.Sleep(3 * time.Second)

err = myBase.Stop(ctx, nil)
```

{{% /tab %}}
{{< /tabs >}}

### 5. Drive in a square

{{< tabs >}}
{{% tab name="Python" %}}

```python
for _ in range(4):
    await base.move_straight(velocity=500, distance=500)
    await base.spin(velocity=100, angle=90)
print("Square complete")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
for i := 0; i < 4; i++ {
    err = myBase.MoveStraight(ctx, 500, 500.0, nil)
    if err != nil {
        logger.Fatal(err)
    }
    err = myBase.Spin(ctx, 90, 100.0, nil)
    if err != nil {
        logger.Fatal(err)
    }
}
fmt.Println("Square complete")
```

{{% /tab %}}
{{< /tabs >}}

## What's Next

- [Navigate to a Waypoint](/navigation/how-to/navigate-to-waypoint/) --
  autonomous GPS-based navigation.
- [Navigation Service Configuration](/navigation/reference/navigation-service/)
  -- configure the navigation service.
