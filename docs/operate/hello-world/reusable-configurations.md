---
linkTitle: "Reusable Configurations"
title: "Reusable Configurations with Fragments"
weight: 18
layout: "docs"
type: "docs"
description: "Learn how fragments let you capture configuration once and reuse it across machines."
date: "2025-02-02"
---

Robot configuration can be complex.
A machine's setup includes hardware definitions, service configurations, network settings, calibration values, and more.

Fragments let you capture configuration once and reuse it.
A fragment is a reusable block of machine configuration.
You can use fragments others have created—from the Viam Registry or your organization—to configure common hardware without starting from scratch.
You can create your own fragments to share configuration across machines.
And when a fragment updates, every machine using it receives the change automatically.

This document explains what fragments enable and when to use them.
For step-by-step instructions on creating and managing fragments, see [Reuse machine configuration](/manage/fleet/reuse-configuration/).

---

## The Problems Fragments Solve

### For Any Machine: Leverage Existing Work

Configuring a machine from scratch means understanding every component, every parameter, every interaction.
Much of this work has been done before—by Viam, by hardware vendors, by the community.

**Avoid reinventing configuration.**
Without reusable configurations, you set up common hardware patterns from scratch every time.
Someone has already figured out the right settings for that camera, that sensor, that data pipeline.
Fragments let you use their work.

**Stay current.**
Hardware configurations evolve.
Better defaults emerge.
Bugs get fixed.
If you configure everything manually, you track and apply these improvements yourself.
With fragments, updates flow to you automatically.

### For Fleets: Maintain Consistency at Scale

As you move from one machine to many, new problems emerge:

**Configuration drift.**
When you configure machines individually, small differences creep in over time.
One machine gets a fix that others don't.
Settings diverge.
Debugging becomes harder because you can't assume machines are configured the same way.

**Update coordination.**
Pushing a configuration change to fifty machines manually is tedious and error-prone.
Miss one machine, and you have an inconsistent fleet.
Make a typo on one, and that machine behaves differently.

**Hardware variance.**
Real fleets have variation.
Some machines have different camera models.
Some are deployed in environments that require adjusted settings.
Managing these differences while maintaining a shared baseline is difficult without the right abstractions.

**Provisioning at scale.**
Setting up each new machine from scratch doesn't scale.
You need a way to stamp out machines with known-good configurations quickly and reliably.

Fragments address these problems—for single machines and fleets alike—by providing reusable, maintainable sources of configuration.

---

## Core Capabilities

### Use What Others Have Built

The Viam Registry contains fragments for common hardware and configurations.
Your organization may have fragments for internal standards.
Before configuring from scratch, check what's already available.

Using an existing fragment gives you:

- **Tested configuration.** Someone has already debugged the settings.
- **Automatic updates.** When the fragment improves, your machine benefits.
- **Faster setup.** Add the fragment, provide any required variables, and you're done.

This applies whether you have one machine or a hundred.
A single prototype benefits from a well-maintained camera fragment just as much as a production fleet does.

### Define Once, Deploy Everywhere

A fragment captures a configuration that you want to reuse.
This might be a complete machine setup (every component and service), or it might be a subset (just the camera and vision pipeline).
You create the fragment once, then add it to any machine that needs that configuration.

When you add a fragment to a machine, the machine's configuration merges with the fragment's contents.
The machine now has all the components, services, and settings defined in the fragment—without you copying and pasting JSON between machines.

Fragments can also include other fragments.
This lets you compose configurations from smaller, reusable pieces.
A "data capture" fragment might define capture rules; a "camera" fragment might define camera settings; a complete machine fragment might include both.

### Customize Without Forking

Fragments would be limited if every machine using them had to be identical.
In practice, machines vary.
One machine's camera is at a different IP address.
Another machine needs a higher capture rate.
A third machine is in a bright environment and needs adjusted exposure settings.

**Variables** let you parameterize a fragment.
Instead of hardcoding a camera's IP address, you define a variable.
When you add the fragment to a machine, you provide the value for that machine.
Same fragment, different values per machine.

**Overwrites** let you modify specific parts of a fragment for a particular machine.
If one machine in your fleet needs a different sensor threshold, you overwrite that value.
The rest of the fragment stays intact.
You're customizing, not forking.

This creates a spectrum of flexibility:

| Scenario                             | Approach                |
| ------------------------------------ | ----------------------- |
| Machines are identical               | Use fragment as-is      |
| Machines differ by known parameters  | Use fragment variables  |
| One machine needs a specific tweak   | Use fragment overwrites |
| Machines are fundamentally different | Use different fragments |

The goal is to keep shared configuration shared, while allowing necessary differences.

### Update Centrally, Propagate Automatically

When you update a fragment, machines using that fragment receive the update.
You don't push changes to each machine individually—you change the source, and the change propagates.

This works even when machines are offline.
Each machine caches its configuration locally.
When connectivity returns, the machine checks for updates and applies them.
If a machine reboots without network access, it runs from its cached configuration until it can sync.

For stability, you can pin a machine to a specific fragment version.
Pinned machines don't automatically receive updates—they stay on the version you specified until you explicitly update them.
This is useful for machines in production where you want to control exactly when changes take effect.

### Roll Out Safely

Updating configuration across a fleet carries risk.
A bad configuration change can break machines.
Fragments provide mechanisms to manage this risk:

**Version pinning.**
Every change to a fragment creates a new version.
You can pin machines to specific versions, ensuring they don't receive updates until you're ready.

**Tags.**
You can tag fragment versions with names like "stable" or "beta."
Some machines track the "stable" tag and only receive updates you've explicitly marked stable.
Other machines track "beta" and receive updates earlier, serving as a test group.

**Staged rollouts.**
Update a few machines first.
Verify the change works.
Then update the rest.
Tags and version pinning make this workflow possible.

**Rollback.**
If an update causes problems, pin affected machines back to the previous version.
The fragment's version history gives you a known-good state to return to.

---

## Fragment Patterns

Different situations call for different fragment structures.
Here are common patterns:

### The Golden Config

A complete machine template containing every component and service the machine needs.
Add this fragment to a new machine, and it's fully configured.

**Example:** The Viam Rover fragments configure the board, motors, base, camera, and all other components for a standard rover.
Add the fragment, and the rover is ready to drive.

**When to use:** Fleets of identical hardware where every machine should be configured the same way.

### The Component Library

Smaller fragments that each configure one piece of functionality.
You compose these into complete configurations by adding multiple fragments to a machine.

**Example:** A "camera-and-capture" fragment that configures a camera and data capture rules.
Add it to any machine that needs camera-based data collection, regardless of what else that machine does.

**When to use:** Mix-and-match scenarios where different machines need different combinations of capabilities.

### The Hardware Variant

A base fragment with overwrites for hardware differences.
The fragment defines the standard configuration; machines with different hardware override the relevant parts.

**Example:** A fleet of inspection robots where most have Camera Model A, but some have Camera Model B.
The fragment configures Model A.
Machines with Model B override the camera configuration while keeping everything else from the fragment.

**When to use:** Fleets with hardware variation that can be expressed as parameter differences.

### The Provisioning Template

A fragment designed for zero-touch device setup.
Combined with Viam's provisioning tools, new devices automatically configure themselves when they first connect.

**Example:** An air quality monitoring company creates a fragment with their sensor configuration and data pipeline.
They flash SD cards with the provisioning agent and fragment ID.
When a device boots and connects to WiFi, it automatically applies the fragment—no manual configuration needed.

**When to use:** Shipping pre-configured devices to customers or remote sites.

---

## When to Use Fragments

### Good Fit

Fragments make sense when:

- **A fragment already exists for your hardware or use case.** Check the Viam Registry and your organization's fragments before configuring from scratch—even for a single machine.

- **You have two or more machines with shared configuration.** Fragments prevent drift and simplify updates across machines.

- **You need to update multiple machines simultaneously.** Changing a fragment is faster and safer than updating machines individually.

- **Your hardware varies in ways that can be expressed as parameters.** Fragment variables and overwrites handle this cleanly.

- **You're shipping devices to customers or remote sites.** Provisioning templates make setup reliable and repeatable.

### Not a Fit

Creating your own fragment may not be worth it when:

- **You have a single machine with unique configuration.** If no existing fragment applies and you won't replicate this machine, direct configuration is simpler. You can always extract a fragment later.

- **Your machines have fundamentally different architectures.** If machines share little configuration, separate fragments (or direct configuration) may be clearer than heavily overwritten fragments.

- **Configuration changes more often than it's shared.** If every machine needs constant, unique adjustments, the overhead of fragment management may not pay off.

### When to Create Your Own Fragment

You might _use_ fragments from day one—applying existing fragments from the registry or your organization.
But when should you _create_ your own?

The practical answer: when you're about to configure a second machine that resembles the first.
Before you copy-paste configuration, extract it into a fragment.
This takes minutes and immediately prevents the drift that copy-paste introduces.

You don't need to plan for custom fragments from the start.
Use existing fragments where they fit.
Configure unique aspects directly.
When you need to replicate a configuration, that's when you create your own fragment.

---

## Fragments in Your Workflow

Fragments sit between your application code and individual machine configurations.
Your code defines behavior; fragments define the hardware that behavior runs on.

### Fragments vs. Modules

Both fragments and modules are reusable, but they serve different purposes:

| Aspect              | Fragments                                           | Modules                                                 |
| ------------------- | --------------------------------------------------- | ------------------------------------------------------- |
| **What it is**      | Reusable configuration (JSON)                       | Reusable code (Go/Python)                               |
| **Contains**        | Component definitions, service configs, parameters  | Custom logic, drivers, algorithms                       |
| **Answers**         | "What hardware do I have and how is it configured?" | "What can my hardware do beyond built-in capabilities?" |
| **Changes require** | Config update in Viam app (no rebuild)              | Code changes, rebuild, republish                        |

A chess-playing robot illustrates the distinction.
The **module** contains the chess logic: analyzing the board, calculating moves, coordinating the arm and gripper.
The **fragment** contains the hardware specifics: the arm's IP address, the camera's serial number, the gripper's offset values.

This separation means the same module can work with different fragments.
Deploy the chess module to a new robot arm by providing a new fragment with that arm's settings—no code changes needed.

**Decision framework:**

- Hardware settings that vary per deployment → fragment variables
- Custom behavior or new capabilities → module

### Team Patterns

As your team grows, consider:

- **Who owns the fragment?** Typically, the team responsible for that hardware or deployment pattern.
- **How do changes get reviewed?** Treat fragment changes like code changes when they affect production machines.
- **How do you test changes?** Use version pinning to test fragment updates on a subset of machines before fleet-wide rollout.

---

## Limitations and Boundaries

Fragments are powerful but not unlimited:

**Some resources can't be in fragments.**
Triggers (event-based automations) must be configured directly on machines, not in fragments.

**Overwrites are all-or-nothing.**
When you overwrite a fragment value on a machine, the overwrite either applies completely or fails entirely.
There's no partial application.
This prevents machines from ending up in inconsistent states.

**Complexity has limits.**
If a machine's overwrites become extensive—overriding most of the fragment's values—consider whether a separate fragment would be clearer.
Heavy overwrites can make configuration harder to understand and debug.

---

## Next Steps

To start working with fragments, see [Reuse machine configuration](/manage/fleet/reuse-configuration/) for step-by-step instructions on creating, customizing, and managing fragments.
