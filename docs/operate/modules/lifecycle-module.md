---
title: "Lifecycle of a module"
linkTitle: "Lifecycle of a module"
weight: 20
layout: "docs"
type: "docs"
description: "Learn about the lifecycle of a module: How and when it starts, shuts down, and reconfigures."
date: "2025-10-17"
aliases:
  - /operate/modules/other-hardware/lifecycle-module/
---

Modules run on your machine, alongside `viam-server` as separate processes, communicating with `viam-server` over UNIX sockets.

[`viam-server` manages](/operate/reference/viam-server/) the dependencies, start-up, reconfiguration, [data management](/data-ai/capture-data/capture-sync/), and shutdown behavior of your {{< glossary_tooltip term_id="resource" text="modular resources" >}}.

The lifecycle of a module and the resources it provides are as follows:

1. `viam-server` starts, and if it is connected to the internet, it checks for configuration updates.

1. `viam-server` starts any configured modules.

1. When a module initializes, it registers its model or models and associated [APIs](/dev/reference/apis/) with `viam-server`, making the models available for use.

1. For each modular resource configured on the machine, `viam-server` uses the return values of the resource's `validate_config` function to determine the required and optional [dependencies](/operate/modules/advanced/dependencies/) of the resource.

1. If a required dependency is not already running, `viam-server` starts it before starting the resource.
   If a required dependency is not found or fails to start, `viam-server` does not start the resource that depends on it.

1. `viam-server` calls the resource's constructor to build the resource based on its configuration.
   Typically, the constructor calls the `reconfigure` function.

1. Modular resources can fail to start for various reasons:

   - a validation error or exception thrown in constructor
   - a validation error or exception thrown during reconfiguration
   - exceeding the [configured timeout limits](/operate/modules/advanced/module-configuration/#environment-variables) (default 5 minutes to start up and 1 minute to reconfigure)

   Check the `viam-server` logs for these [errors](/dev/tools/common-errors/#timed-out-waiting-for-module) on the machine's **LOGS** tab.

1. Once the modular resource has started up and finished configuring, it is available for use.

1. If at any point the user changes the configuration of the machine, `viam-server` reconfigures the affected resources within 15 seconds.

1. If `viam-server` attempts to shut down an individual module (for example due to a user disabling a module) and the module does not shut down within 30 seconds, `viam-server` kills the module process.

1. When `viam-server` shuts down, it first attempts to shut down each module process sequentially in no particular order.
   If a module process does not shut down within 30 seconds, it is killed with a `SIGKILL` signal.
   If any modules are still running after 90 seconds, `viam-server` kills them as well.
   This means that if four modules are running and the first three each fail to shut down within 30 seconds each, the fourth is killed immediately at the 90 second mark.

## Crash recovery

If a module process crashes unexpectedly, `viam-server` automatically attempts to restart it:

1. `viam-server` detects the unexpected exit and marks the module as failed.
2. The machine's status changes to **initializing** while any modules are in the failed state.
3. `viam-server` retries starting the module every 5 seconds.
4. On successful restart, `viam-server` re-adds all resources that the module previously provided, in the correct dependency order.
5. The machine's status returns to **running** once all modules are healthy.

If the module continues to crash, `viam-server` keeps retrying indefinitely.
Check the **LOGS** tab for crash tracebacks and startup errors.

## Communication

By default, modules communicate with `viam-server` over a Unix domain socket.
The socket file is created in `/tmp/` with a randomized suffix.

TCP mode is also available and is used automatically on Windows, or when the Unix socket path would exceed the operating system limit (103 characters on macOS).
You can force TCP mode by setting `"tcp_mode": true` in the module configuration or by setting the `VIAM_TCP_SOCKETS=true` environment variable.
In TCP mode, the module binds to `127.0.0.1` on an automatically assigned port.

## Data directory

Every module receives a persistent data directory at `~/.viam/module-data/<robot-id>/<module-name>/`.
The path is available inside the module via the `VIAM_MODULE_DATA` environment variable.
This directory persists across module restarts and reconfigurations.

## Timeouts

| Event | Timeout | Behavior on timeout |
|-------|---------|---------------------|
| Module startup (ready check) | ~15 seconds | Module marked as failed; retry begins |
| Resource removal during shutdown | 20 seconds (all resources combined) | Resources orphaned |
| Module process shutdown (SIGTERM) | 30 seconds | Process killed with SIGKILL |
| Total shutdown (all modules) | 90 seconds | All remaining module processes killed |
| First-run setup script | 1 hour (configurable via `first_run_timeout`) | Module startup fails |
| Crash restart retry interval | 5 seconds | Next attempt after delay |

Microcontroller modules function differently and are embedded in the firmware you flash onto your device.
For more information see [Modules for ESP32](/operate/modules/advanced/micro-module/).
