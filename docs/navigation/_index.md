---
linkTitle: "Navigation"
title: "Navigation"
weight: 170
layout: "docs"
type: "docs"
no_list: true
noedit: true
open_on_desktop: true
overview: true
description: "Autonomous GPS-based navigation for mobile robot bases."
notoc: true
aliases:
  - /operate/mobility/move-base/
  - /how-tos/navigate/
  - /use-cases/navigate/
---

## What Problem This Solves

Your mobile robot needs to drive itself to a location, follow a route, or avoid
obstacles while moving. Navigation combines GPS positioning, base motor control,
and optional vision-based obstacle detection to move your robot autonomously.

## How It Works

Viam provides two approaches to moving a mobile base:

**Direct base commands** -- call `Spin`, `MoveStraight`, or `SetVelocity` on the
base component for manual or teleoperated control. Use this for simple movements
or when you are controlling the robot in real time.

**Navigation service** -- configure the navigation service for autonomous
waypoint-based navigation. The service manages the full navigation loop: it reads
GPS position, plans paths between waypoints, sends movement commands to the base,
and optionally detects and avoids obstacles using a vision service.

The navigation service supports three modes:

| Mode | Behavior |
|------|----------|
| **Manual** | Service is configured but does not drive. Use this for setup and testing. |
| **Waypoint** | Drives to each waypoint in sequence. |
| **Explore** | Drives in the direction of waypoints while actively exploring the environment. |

## Get Started

{{< cards >}}
{{% card link="/navigation/how-to/drive-the-base/" noimage="true" %}}
{{% card link="/navigation/how-to/navigate-to-waypoint/" noimage="true" %}}
{{% card link="/navigation/how-to/move-to-gps-coordinate/" noimage="true" %}}
{{% card link="/navigation/how-to/avoid-obstacles/" noimage="true" %}}
{{% card link="/navigation/how-to/follow-a-patrol-route/" noimage="true" %}}
{{% card link="/navigation/how-to/detect-while-moving/" noimage="true" %}}
{{< /cards >}}

### Reference

{{< cards >}}
{{% card link="/navigation/reference/" noimage="true" %}}
{{< /cards >}}
