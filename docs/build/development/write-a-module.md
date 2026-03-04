---
linkTitle: "Write a Module"
title: "Write a Module"
weight: 20
layout: "docs"
type: "docs"
description: "Build a separate-process module that implements a typed resource API."
date: "2025-01-30"
---

## What Problem This Solves

Inline modules are great for prototyping, but production deployments need more.
A separate-process module runs as its own binary, isolated from `viam-server`.
This means it can have its own dependencies, crash without taking down the
server, and be distributed through the Viam registry.

More importantly, a full module implements a typed resource API -- sensor, camera,
motor, or any other Viam component type. This means data capture, ML pipelines,
and the CONTROL tab work with your component automatically, without requiring
custom DoCommand calls.

## Concepts

### Full modules vs. inline modules

| | Inline module | Full module |
|---|---|---|
| Process | Runs inside `viam-server` | Runs as a separate process |
| Isolation | Shares memory with server | Independent process, own memory |
| Dependencies | Must match server's environment | Can use any libraries and versions |
| Crash behavior | Takes down `viam-server` | Module restarts; server continues |
| Distribution | Local only | Can be uploaded to Viam registry |
| Best for | Prototyping, simple extensions | Production, shared modules |

### Choosing a resource API

Viam defines standard APIs for common hardware types. Pick the API that best
matches your hardware:

| API | Use when your hardware... | Key methods |
|-----|---------------------------|-------------|
| `sensor` | Produces readings (temperature, distance, humidity) | `get_readings()` |
| `camera` | Produces images or point clouds | `get_image()`, `get_point_cloud()` |
| `motor` | Drives rotational or linear motion | `set_power()`, `go_for()`, `stop()` |
| `servo` | Moves to angular positions | `move()`, `get_position()` |
| `board` | Exposes GPIO pins, analog readers, digital interrupts | `gpio_pin_by_name()`, `analog_by_name()` |
| `encoder` | Tracks position or rotation | `get_position()`, `reset_position()` |
| `movement_sensor` | Reports position, orientation, velocity | `get_position()`, `get_linear_velocity()` |
| `generic` | Does not fit any of the above | `do_command()` |

Using the right API means data capture, the CONTROL tab, and other platform
features work automatically.

### Module lifecycle

Every module goes through a defined lifecycle:

1. **Startup** -- `viam-server` launches the module as a separate process. The
   module registers its models and opens a gRPC connection back to the server.
2. **Validation** -- `viam-server` calls your `validate_config` method to check
   attributes and declare dependencies.
3. **Creation** -- If validation passes, `viam-server` calls your constructor
   with the resolved dependencies.
4. **Reconfiguration** -- If the user changes the configuration, `viam-server`
   calls `validate_config` again, then `reconfigure`.
5. **Shutdown** -- `viam-server` calls `close`. Clean up resources here.

### Dependencies

Dependencies let your component use other components on the same machine. You
declare dependencies in `validate_config` by returning the names of components
your module needs. `viam-server` resolves these, ensures the depended-on
components are ready, and passes them to your constructor.

## Steps

### 1. Generate the module

Run the Viam CLI generator, choosing the resource subtype that matches your
hardware. This example builds a sensor module:

```bash
viam module generate
```

| Prompt | What to enter | Why |
|--------|---------------|-----|
| Module name | `my-sensor-module` | A short, descriptive name |
| Language | `python` or `go` | Your implementation language |
| Visibility | `private` | Keep it private while developing |
| Namespace | Your organization namespace | Scopes the module to your org |
| Resource type | `component` | For hardware components |
| Resource subtype | `sensor` | Implements the sensor API |
| Model name | `my-sensor` | The model name for your sensor |
| Cloud build | `yes` | Enable for later deployment |
| Register | `yes` | Registers the module with Viam |

### 2. Implement the resource API

A sensor module must implement `get_readings()`, which returns a map of reading
names to values. This is the method that data capture calls, the CONTROL tab
displays, and your application code queries.

This example builds a sensor that reads from a custom HTTP API endpoint.
Replace the HTTP call with whatever data source your sensor uses.

{{< tabs >}}
{{% tab name="Python" %}}

```python
import requests
from typing import Any, ClassVar, Mapping, Optional, Sequence, Self, Tuple

from viam.components.sensor import Sensor
from viam.proto.app.robot import ComponentConfig
from viam.proto.common import ResourceName
from viam.resource.base import ResourceBase
from viam.resource.easy_resource import EasyResource
from viam.resource.types import Model, ModelFamily


class MySensor(Sensor, EasyResource):
    """A custom sensor that reads from an HTTP endpoint."""

    MODEL: ClassVar[Model] = Model(
        ModelFamily("my-org", "my-sensor-module"), "my-sensor"
    )

    source_url: str
    poll_interval: float

    @classmethod
    def new(cls, config: ComponentConfig,
            dependencies: Mapping[ResourceName, ResourceBase]) -> Self:
        sensor = cls(config.name)
        sensor.reconfigure(config, dependencies)
        return sensor

    @classmethod
    def validate_config(
        cls, config: ComponentConfig
    ) -> Tuple[Sequence[str], Sequence[str]]:
        fields = config.attributes.fields
        if "source_url" not in fields:
            raise Exception("source_url is required")
        if not fields["source_url"].string_value.startswith("http"):
            raise Exception("source_url must be an HTTP or HTTPS URL")
        return [], []  # No dependencies on other components

    def reconfigure(self, config: ComponentConfig,
                    dependencies: Mapping[ResourceName, ResourceBase]) -> None:
        fields = config.attributes.fields
        self.source_url = fields["source_url"].string_value
        self.poll_interval = (
            fields["poll_interval"].number_value
            if "poll_interval" in fields
            else 10.0
        )

    async def get_readings(
        self,
        *,
        extra: Optional[Mapping[str, Any]] = None,
        timeout: Optional[float] = None,
        **kwargs,
    ) -> Mapping[str, Any]:
        try:
            response = requests.get(self.source_url, timeout=5)
            response.raise_for_status()
            data = response.json()
            return {
                "temperature": data["temp"],
                "humidity": data["humidity"],
            }
        except requests.RequestException as e:
            self.logger.error(f"Failed to read from {self.source_url}: {e}")
            return {"error": str(e)}

    async def close(self):
        self.logger.info("Shutting down MySensor")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
package mysensor

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"

    "go.viam.com/rdk/components/sensor"
    "go.viam.com/rdk/logging"
    "go.viam.com/rdk/resource"
)

var Model = resource.NewModel("my-org", "my-sensor-module", "my-sensor")

type Config struct {
    SourceURL    string  `json:"source_url"`
    PollInterval float64 `json:"poll_interval"`
}

func (cfg *Config) Validate(path string) ([]string, error) {
    if cfg.SourceURL == "" {
        return nil, fmt.Errorf("source_url is required")
    }
    return nil, nil // No dependencies on other components
}

type MySensor struct {
    resource.Named
    resource.TriviallyCloseable
    logger    logging.Logger
    sourceURL string
    client    *http.Client
}

func init() {
    resource.RegisterComponent(sensor.API, Model,
        resource.Registration[sensor.Sensor, *Config]{
            Constructor: newMySensor,
        },
    )
}

func newMySensor(
    ctx context.Context,
    deps resource.Dependencies,
    conf resource.Config,
    logger logging.Logger,
) (sensor.Sensor, error) {
    cfg, err := resource.NativeConfig[*Config](conf)
    if err != nil {
        return nil, err
    }

    timeout := time.Duration(cfg.PollInterval) * time.Second
    if timeout == 0 {
        timeout = 10 * time.Second
    }

    return &MySensor{
        Named:     conf.ResourceName().AsNamed(),
        logger:    logger,
        sourceURL: cfg.SourceURL,
        client:    &http.Client{Timeout: timeout},
    }, nil
}

type sensorResponse struct {
    Temp     float64 `json:"temp"`
    Humidity float64 `json:"humidity"`
}

func (s *MySensor) Readings(
    ctx context.Context,
    extra map[string]interface{},
) (map[string]interface{}, error) {
    resp, err := s.client.Get(s.sourceURL)
    if err != nil {
        s.logger.CErrorw(ctx, "failed to read from source",
            "url", s.sourceURL, "error", err)
        return nil, fmt.Errorf("failed to read from %s: %w", s.sourceURL, err)
    }
    defer resp.Body.Close()

    var data sensorResponse
    if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
        return nil, fmt.Errorf("failed to decode response: %w", err)
    }

    return map[string]interface{}{
        "temperature": data.Temp,
        "humidity":    data.Humidity,
    }, nil
}
```

{{% /tab %}}
{{< /tabs >}}

The key differences from an inline module:

- You implement `get_readings()` / `Readings()` instead of `do_command()`.
- The return value follows the sensor API contract: a map of named readings.
- Data capture, the CONTROL tab, and the SDKs all work automatically.

### 3. Handle dependencies

Many modules need access to other components on the machine. You declare these
dependencies in `validate_config` and receive them in your constructor.

This example shows a sensor that depends on a camera to compute image brightness
as a "light level" reading:

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.components.camera import Camera

class LightSensor(Sensor, EasyResource):
    MODEL: ClassVar[Model] = Model(
        ModelFamily("my-org", "my-sensor-module"), "light-sensor"
    )

    camera_name: str
    camera: Camera

    @classmethod
    def validate_config(
        cls, config: ComponentConfig
    ) -> Tuple[Sequence[str], Sequence[str]]:
        fields = config.attributes.fields
        if "camera_name" not in fields:
            raise Exception("camera_name is required")
        camera_name = fields["camera_name"].string_value
        return [camera_name], []  # Declare camera as a required dependency

    def reconfigure(self, config: ComponentConfig,
                    dependencies: Mapping[ResourceName, ResourceBase]) -> None:
        fields = config.attributes.fields
        self.camera_name = fields["camera_name"].string_value

        for name, dep in dependencies.items():
            if name.name == self.camera_name:
                self.camera = dep
                break

    async def get_readings(self, *, extra=None, timeout=None,
                           **kwargs) -> Mapping[str, Any]:
        image = await self.camera.get_image()
        pixels = list(image.getdata())
        avg_brightness = sum(sum(p[:3]) / 3 for p in pixels) / len(pixels)
        return {"light_level": round(avg_brightness, 2)}
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
type LightConfig struct {
    CameraName string `json:"camera_name"`
}

func (cfg *LightConfig) Validate(path string) ([]string, error) {
    if cfg.CameraName == "" {
        return nil, fmt.Errorf("camera_name is required")
    }
    return []string{cfg.CameraName}, nil // Declare camera as a dependency
}

type LightSensor struct {
    resource.Named
    resource.TriviallyCloseable
    logger logging.Logger
    camera camera.Camera
}

func newLightSensor(
    ctx context.Context,
    deps resource.Dependencies,
    conf resource.Config,
    logger logging.Logger,
) (sensor.Sensor, error) {
    cfg, err := resource.NativeConfig[*LightConfig](conf)
    if err != nil {
        return nil, err
    }

    cam, err := camera.FromDependencies(deps, cfg.CameraName)
    if err != nil {
        return nil, fmt.Errorf("camera %q not found: %w", cfg.CameraName, err)
    }

    return &LightSensor{
        Named:  conf.ResourceName().AsNamed(),
        logger: logger,
        camera: cam,
    }, nil
}

func (s *LightSensor) Readings(
    ctx context.Context,
    extra map[string]interface{},
) (map[string]interface{}, error) {
    images, _, err := s.camera.Images(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed to get image: %w", err)
    }
    if len(images) == 0 {
        return nil, fmt.Errorf("no images returned from camera")
    }

    img := images[0].Image
    bounds := img.Bounds()
    var total float64
    pixels := 0
    for y := bounds.Min.Y; y < bounds.Max.Y; y++ {
        for x := bounds.Min.X; x < bounds.Max.X; x++ {
            r, g, b, _ := img.At(x, y).RGBA()
            total += float64(r+g+b) / 3.0 / 65535.0 * 255.0
            pixels++
        }
    }

    return map[string]interface{}{
        "light_level": total / float64(pixels),
    }, nil
}
```

{{% /tab %}}
{{< /tabs >}}

The pattern is the same in both languages:

1. Declare the dependency name in `validate_config` / `Validate`.
2. Resolve the dependency in `reconfigure` / the constructor.
3. Use the dependency in your API methods.

### 4. Test locally with hot reload

During development, run your module locally and configure it on your machine.

**Start the module:**

{{< tabs >}}
{{% tab name="Python" %}}

```bash
cd my-sensor-module
pip install -r requirements.txt
python -m src.main
```

{{% /tab %}}
{{% tab name="Go" %}}

```bash
cd my-sensor-module
go build -o bin/module cmd/module/main.go
./bin/module
```

{{% /tab %}}
{{< /tabs >}}

**Configure as a local module:**

1. In the [Viam app](https://app.viam.com), navigate to your machine's
   **CONFIGURE** tab.
2. Click **+** and select **Local module**, then **Local module**.
3. Set the **Executable path** to your module binary or script.
4. Click **Create**.
5. Click **+** again and select **Local module**, then **Local component**.
6. Select your module, set the type and model, and configure attributes:

```json
{
  "source_url": "https://api.example.com/sensor/data"
}
```

7. Click **Save**.

**Test from the CONTROL tab:**

1. Find your sensor component.
2. Click **Get readings**. You should see the temperature and humidity values.

**Hot reload:**

When you change your module code:

- **Python:** Save the file. `viam-server` detects the change and restarts the
  module automatically within a few seconds.
- **Go:** Rebuild the binary. `viam-server` detects the new binary and restarts
  the module.

You do not need to restart `viam-server` or reconfigure anything.

### 5. Add logging

Both the Python and Go SDKs provide a logger that writes to `viam-server`'s log
stream, visible in the **LOGS** tab.

{{< tabs >}}
{{% tab name="Python" %}}

```python
self.logger.info("Sensor initialized with source URL: %s", self.source_url)
self.logger.debug("Raw response from source: %s", data)
self.logger.warning("Source returned unexpected field: %s", field_name)
self.logger.error("Failed to connect to source: %s", error)
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
s.logger.CInfof(ctx, "Sensor initialized with source URL: %s", s.sourceURL)
s.logger.CDebugf(ctx, "Raw response from source: %v", data)
s.logger.CWarnw(ctx, "Source returned unexpected field", "field", fieldName)
s.logger.CErrorw(ctx, "Failed to connect to source", "error", err)
```

{{% /tab %}}
{{< /tabs >}}

Use `info` for significant events, `debug` for detailed data, `warning` for
recoverable problems, and `error` for failures.

## Try It

1. Generate a sensor module using `viam module generate`.
2. Implement `validate_config`, the constructor, and `get_readings`/`Readings`.
3. Configure the module on your machine as a local module.
4. Open the CONTROL tab and click **Get readings**. Verify the readings appear.
5. Enable data capture on the sensor. Wait one minute, then check the **DATA**
   tab to confirm readings are flowing to the cloud.
6. Change a value in your module code and verify the new field appears within
   a few seconds.

## Troubleshooting

{{< expand "Module crashes on startup" >}}

- Check the **LOGS** tab for the crash traceback. The most common cause is a
  missing dependency -- a Python import not in `requirements.txt` or a Go
  package not in `go.mod`.
- For Python, verify the module runs standalone: `python -m src.main`.
- For Go, verify the binary runs: `./bin/module`.

{{< /expand >}}

{{< expand "Dependency not found" >}}

- Confirm the dependency name returned by `validate_config` matches the
  component name on the machine exactly. Names are case-sensitive.
- Verify the depended-on component exists and is configured correctly.
- Check for circular dependencies -- both components will fail to start.

{{< /expand >}}

{{< expand "Readings returning None or nil" >}}

- Add logging inside `get_readings` / `Readings` to see what data your source
  returns.
- Check network connectivity from the machine. Verify the URL is reachable:
  `curl https://api.example.com/sensor/data`.
- Confirm the source is returning valid JSON.

{{< /expand >}}

{{< expand "Hot reload not working" >}}

- **Python:** Verify `viam-server` is watching the module path. Check the LOGS
  tab for module restart messages.
- **Go:** Rebuild the binary after every change. `viam-server` watches the
  binary, not the source files.
- Check the module's executable path. If it points to a symlink, the file
  watcher may not detect changes.

{{< /expand >}}

{{< expand "Data capture not recording readings" >}}

- Verify data capture is enabled on your sensor component, not just on the data
  management service. Both are required.
- Check that `get_readings` returns a valid map (not `None` or an error).
- Look at the capture frequency. If it is very low, you may need to wait longer
  to see data.

{{< /expand >}}

## What's Next

- [Deploy a Module](/build/development/deploy-a-module/) -- package your module
  and upload it to the Viam registry for distribution.
- [Write an Inline Module](/build/development/write-an-inline-module/) -- if you
  need a quick prototype using the Generic component.
- [Add Computer Vision](/build/vision-detection/add-computer-vision/) -- build a
  vision service module that processes camera feeds.
