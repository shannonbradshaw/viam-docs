---
linkTitle: "What is a module?"
title: "What is a module?"
weight: 1
layout: "docs"
type: "docs"
description: "Understand the two kinds of modules and how configured hardware connects to application logic."
date: "2025-03-07"
---

Modules extend what your machine can do. They come in two varieties: driver
modules that add support for new hardware, and logic modules that tie
components together with decision-making code.

If you've already [configured components](/hardware/configure-hardware/) on
your machine, each one works individually: you can test it from the Viam app,
capture its data, and call its API from a script. The next step is making
them work **together**. A camera detects an object, and a motor responds. A
temperature sensor crosses a threshold, and a notification fires. A movement
sensor reports position, and an arm adjusts.

## Two kinds of modules

Viam has two kinds of modules, and they serve different purposes:

### Driver modules: add hardware support

A [driver module](/build-modules/write-a-driver-module/) teaches Viam how to
talk to a specific piece of hardware. It implements a standard component API
(sensor, camera, motor, etc.) for a device that `viam-server` doesn't support
out of the box.

You need a driver module when you have hardware with **no existing model**.
Once the driver module exists, the hardware behaves like any built-in component.
Data capture, test panels, and the SDKs work automatically.

### Logic modules: sense and act

A [logic module](/build-modules/write-a-logic-module/) reads from components
and takes action based on what it finds. It runs as a service alongside
`viam-server`, declares dependencies on the resources it needs, and implements
your application's decision-making.

Use a logic module when you need your machine to:

- **React to sensor data**: trigger an alert or an actuator when a reading
  crosses a threshold.
- **Coordinate multiple components**: read from a camera and command an arm
  based on what the camera sees.
- **Run continuous processes**: monitor, aggregate, or transform data from
  multiple sources.
- **Schedule actions**: perform operations at specific intervals or times.

A logic module declares its dependencies in configuration. `viam-server`
ensures those resources are initialized and ready before the module starts,
and passes them to your code as typed API clients. If a dependency
reconfigures, your module gets notified and can adapt.

## How it fits together

Here's the typical progression:

1. **Configure components** ([How components work](/hardware/configure-hardware/)).
   Your machine can sense and act through individual hardware.
2. **Test interactively**. Use test panels in the Viam app and SDK scripts
   to verify each component works.
3. **Capture data** ([Capture and Sync Data](/data/capture-and-sync-data/)).
   Start recording what your sensors observe.
4. **Write a logic module** ([Write a Logic Module](/build-modules/write-a-logic-module/)).
   Tie components together with decision-making code.
5. **Deploy** ([Deploy a Module](/build-modules/deploy-a-module/)).
   Package your module and deploy it to one machine or a fleet.

Steps 1-3 happen entirely through configuration with no code required.
Step 4 is where you write code, and the module pattern means your code runs
on the machine as a managed process with proper lifecycle, dependency
resolution, and hot reloading.

## Example: sense and respond

Imagine a machine with a temperature sensor and a motor-driven fan.
Without a logic module, both work independently. You can read the temperature
and you can spin the fan, but nothing connects them.

A logic module bridges the gap:

- It declares dependencies on the sensor and the motor.
- On startup, it gets typed API clients for both.
- It reads the sensor at an interval.
- When the temperature exceeds a threshold, it commands the motor to run.
- When it drops below, it stops the motor.

This is a few dozen lines of code in Python or Go. The module pattern handles
everything else: startup, shutdown, reconfiguration, crash recovery, and
deployment.

See [Write a Logic Module](/build-modules/write-a-logic-module/) for the full
walkthrough with working code.

## What's next

- [Write a Logic Module](/build-modules/write-a-logic-module/): build a module
  that monitors sensors and coordinates components.
- [Write a Driver Module](/build-modules/write-a-driver-module/): add support
  for hardware that Viam doesn't cover out of the box.
- [Deploy a Module](/build-modules/deploy-a-module/): package and distribute
  your module to machines.
