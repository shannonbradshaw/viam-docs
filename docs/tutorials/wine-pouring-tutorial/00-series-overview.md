# Build a Dual-Arm Wine Pouring Robot with Viam

This tutorial series walks you through building and programming a dual-arm robotic system that can autonomously detect glasses and pour wine. Along the way, you'll learn key Viam platform concepts that apply to any robotics project.

## What You'll Build

A robot sommelier with:
- **Two xArm 6 robotic arms** - one to handle the bottle, one to position glasses
- **Intel RealSense depth cameras** - for 3D point cloud-based object detection
- **ML-powered vision** - trained models to detect glasses during the pour
- **Collision-free motion planning** - safe movements around obstacles
- **Choreographed pour sequence** - touch → prep → pour → return

![Wine Pouring Demo Architecture](./images/architecture-diagram.png)

## Series Structure

Each part teaches standalone Viam concepts while building toward the complete system:

| Part | Title | Viam Concepts | Standalone Value |
|------|-------|---------------|------------------|
| 1 | [Hardware Setup & Arm Configuration](./01-hardware-setup.md) | Machine creation, modules, component configuration, testing | Configure any robotic arm with Viam |
| 2 | [Depth Cameras & Point Clouds](./02-cameras-point-clouds.md) | Camera configuration, RealSense setup, point cloud basics | Work with depth cameras in any project |
| 3 | [The Frame System & Obstacle Definition](./03-frame-system-obstacles.md) | Frames, transforms, spatial relationships, world modeling | Set up collision avoidance for any arm |
| 4 | [Training ML Models for Glass Detection](./04-ml-training.md) | Data capture, labeling, model training, deployment | Train custom object detectors |
| 5 | [Vision Service Pipelines](./05-vision-pipelines.md) | Vision services, ML model integration, camera chaining | Build vision processing pipelines |
| 6 | [Motion Planning & Choreography](./06-motion-planning.md) | Motion service, position sequences, arm coordination | Program complex arm movements |
| 7 | [The Pouring Demo Module Deep Dive](./07-pouring-module.md) | Module architecture, DoCommand, state machines | Understand production module patterns |
| 8 | [Fragments for Reusable Configuration](./08-fragments.md) | Fragments, variables, fragment modifications | Deploy configurations across multiple machines |
| 9 | [Optional: StreamDeck Integration](./09-streamdeck.md) | Custom UI integration, DoCommand triggers | Add physical controls to any robot |

## Prerequisites

**Hardware Required:**
- 2× UFactory xArm 6 robotic arms with controllers
- 2× Intel RealSense D435 depth cameras  
- 1× USB webcam (1080p or higher recommended)
- 1× Linux computer (Ubuntu 22.04 recommended) with:
  - USB 3.0 ports for cameras
  - Ethernet ports for arm controllers
  - 16GB+ RAM recommended for motion planning
- Network switch for arm connections
- Stable mounting surface/table
- Optional: Elgato StreamDeck for physical controls

**Recommended Supplies:**
- Wine bottle (or water bottle of similar dimensions: ~75mm diameter, ~300mm height)
- Stemless wine glasses (~100mm diameter, ~110mm height)
- Cable management supplies

**Software Prerequisites:**
- Viam account (free at [app.viam.com](https://app.viam.com))
- Basic familiarity with:
  - Linux command line
  - JSON configuration files
  - Python or Go (for SDK examples)

**Safety Considerations:**
- Robotic arms can cause injury - establish a safety perimeter
- Test movements at reduced speed first
- Use emergency stop functionality
- Never reach into the robot's workspace while operating

## Hardware Layout

```
                   ┌─────────────────────────────────────────┐
                   │              BACK WALL                  │
                   └─────────────────────────────────────────┘
    ┌───────────────────────────────────────────────────────────────────┐
                                TABLE SURFACE               

        ┌─────────┐                ┌──────┐               ┌─────────┐
        │ LEFT    │                │bottle│               │ RIGHT   │
        │ ARM     │                └──────┘               │ ARM     │
        │ (glass) │                                       │ (bottle)│
        └────┬────┘                                       └────┬────┘
             │                                                 │
        ┌────┴────┐                                       ┌────┴────┐
        │ LEFT    │                                       │ RIGHT   │
        │REALSENSE│                                       │REALSENSE│
        │ (depth) │                                       │ (depth) │
        └─────────┘                                       └─────────┘
      
                                   ┌─────┐                 ┌─────────┐ 
                                   │glass│                 │ GLASS   │ 
                                   └─────┘                 │ WEBCAM  │  
                                                           └─────────┘   
    └──────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────────────────────────────┐
                    │              FRONT (operator)           │
                    └─────────────────────────────────────────┘
```

## Quick Reference: IP Addresses and Naming

| Component | IP Address | Config Name |
|-----------|------------|-------------|
| Left Arm | 192.168.1.200 | `left-arm` |
| Right Arm | 192.168.1.208 | `right-arm` |
| Left RealSense | USB (serial: varies) | `left-cam` |
| Right RealSense | USB (serial: varies) | `right-cam` |
| Glass Webcam | USB (video path: varies) | `cam-glass` |

## Getting Help

- **Viam Documentation**: [docs.viam.com](https://docs.viam.com)
- **Viam Discord**: Community support and discussion
- **GitHub Issues**: [viam-modules/viam-pouring-demo](https://github.com/viam-modules/viam-pouring-demo/issues)

## Let's Begin!

Ready to start? Head to [Part 1: Hardware Setup & Arm Configuration](./01-hardware-setup.md).

---

*This tutorial series was created by Viam's Developer Education team. The complete configuration files and code are available in the [companion repository](#).*
