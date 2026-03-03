---
linkTitle: "Filter at the Edge"
title: "Filter Data Before Sync"
weight: 20
layout: "docs"
type: "docs"
description: "Reduce bandwidth and storage costs by filtering data on the machine before syncing to the cloud."
date: "2025-01-30"
---

## What Problem This Solves

A camera capturing one frame per second generates roughly 2.5 GB of images per
day. Most of those frames are redundant -- nothing changed between them. Sending
all of that data to the cloud wastes bandwidth, fills storage, and creates noise
that makes useful data harder to find.

Edge filtering means the machine decides what is worth sending. Instead of
capturing everything and sorting it out later, you configure rules that discard
uninteresting data before it ever leaves the machine. This is especially
important for machines on cellular connections, metered networks, or with
limited local storage.

This how-to covers four techniques, from simplest to most powerful:

1. **Reduce capture frequency** -- capture less often.
2. **Conditional sync** -- capture locally but only sync when conditions are met.
3. **Filtered camera module** -- write code that decides frame-by-frame what to capture.
4. **Sensor threshold filtering** -- only record readings that cross a boundary.

## Concepts

### The capture-then-sync lifecycle

When data capture is enabled, `viam-server` writes captured data to the local
filesystem (`~/.viam/capture` by default) at the frequency you configure. A
separate sync process then uploads that data to Viam's cloud. After a successful
upload, local files are cleaned up.

This two-stage design means you have two places to filter:

- **At capture time** -- reduce what gets written to disk in the first place.
- **At sync time** -- capture everything locally but only upload what matters.

### Why filter at the edge

Filtering at the edge (on the machine itself) rather than in the cloud has
several advantages:

- **Lower bandwidth** -- less data transmitted, important for cellular or
  satellite connections.
- **Lower storage costs** -- less data stored in the cloud.
- **Less noise** -- downstream consumers (ML training, dashboards, alerts) work
  with higher-signal data.
- **Faster iteration** -- less data to sift through when reviewing captures.

## Steps

### 1. Reduce capture frequency (time-based sampling)

The simplest filter is capturing less often. If you configured your camera at
1 Hz (one frame per second), consider whether you actually need that rate.

1. In the [Viam app](https://app.viam.com), navigate to your machine's
   **CONFIGURE** tab.
2. Find the component you are capturing from (e.g., `my-camera`).
3. In the **Data capture** section, find the capture method you configured.
4. Change the **frequency** to a lower value:
   - `1` Hz = 1 capture per second = ~2.5 GB/day for camera images
   - `0.1` Hz = 1 capture every 10 seconds = ~250 MB/day
   - `0.0167` Hz = 1 capture per minute = ~25 MB/day
   - `0.00028` Hz = 1 capture per hour = ~0.4 MB/day
5. Click **Save**.

The change takes effect immediately. No restart required.

{{< alert title="Tip" color="tip" >}}

Start with the lowest frequency that meets your needs. You can always
increase it later once you understand your data volume and bandwidth budget.

{{< /alert >}}

This approach works well when you need periodic snapshots but not continuous
monitoring. It does not help when you need to capture specific events -- for
that, continue to the next techniques.

### 2. Configure conditional sync

Conditional sync lets you capture data locally at full frequency but only upload
it to the cloud when certain conditions are met. This is useful when you want
a local buffer of recent data but only care about syncing data that meets
specific criteria.

You configure conditional sync through the data management service in your
machine's configuration. The sync configuration supports conditions based on
sensor readings, component states, or other data sources on your machine.

1. In the Viam app, go to your machine's **CONFIGURE** tab.
2. Find the **data management** service in your configuration. If you do not see
   it, add it by clicking **+**, selecting **Service**, and choosing **data
   management**.
3. In the data management service configuration, set the **Sync interval** to
   control how frequently synced data is uploaded. A longer interval means data
   accumulates locally before being sent in batches.
4. To add sync conditions, you can use the `selective_syncer_name` attribute to
   specify a custom module that controls when sync occurs. This module
   programmatically decides whether accumulated data should be synced based on
   any logic you define -- sensor thresholds, time of day, connectivity status,
   or external triggers.

For example, you might write a selective sync module that checks a temperature
sensor and only triggers sync when the reading exceeds a threshold. The data
management service calls into your module to determine whether to proceed with
each sync cycle.

{{< alert title="Note" color="note" >}}

Conditional sync still captures data locally at your configured
frequency. It only controls when that data gets uploaded. Make sure your
machine has enough local storage for the capture buffer (see
[Storage management](#6-manage-local-storage) below).

{{< /alert >}}

### 3. Build a filtered camera module

The most powerful filtering technique is writing a module that wraps an existing
camera and only outputs frames that meet your criteria. The data capture service
captures from your filtered camera instead of the raw camera, so only
interesting frames are ever written to disk.

This example builds a filtered camera that compares consecutive frames and only
returns a new image when significant visual change is detected. This is useful
for security cameras, monitoring stations, or any scenario where you want to
ignore static scenes.

{{< tabs >}}
{{% tab name="Python" %}}

Create a directory for your module:

```bash
mkdir -p filtered-camera
cd filtered-camera
```

Save this as `main.py`:

```python
import asyncio
from typing import (
    Any,
    ClassVar,
    Dict,
    Final,
    List,
    Mapping,
    Optional,
    Sequence,
    Tuple,
)

import numpy as np
from PIL import Image
from viam.components.camera import Camera, DistortionParameters, IntrinsicParameters
from viam.media.video import CameraMimeType, NamedImage, ViamImage
from viam.module.module import Module
from viam.proto.app.robot import ComponentConfig
from viam.proto.common import ResourceName, ResponseMetadata
from viam.resource.base import ResourceBase
from viam.resource.easy_resource import EasyResource
from viam.resource.types import Model, ModelFamily


class FilteredCamera(Camera, EasyResource):
    """A camera that only returns new frames when significant change is detected."""

    MODEL: ClassVar[Model] = Model(
        ModelFamily("custom", "camera"), "filtered"
    )

    source_camera_name: str
    threshold: float
    last_frame: Optional[np.ndarray]
    last_image: Optional[ViamImage]

    @classmethod
    def new(
        cls,
        config: ComponentConfig,
        dependencies: Mapping[ResourceName, ResourceBase],
    ) -> "FilteredCamera":
        camera = cls(config.name)
        camera.reconfigure(config, dependencies)
        return camera

    def reconfigure(
        self,
        config: ComponentConfig,
        dependencies: Mapping[ResourceName, ResourceBase],
    ) -> None:
        attrs = config.attributes.fields

        # The name of the source camera to wrap.
        self.source_camera_name = attrs["source_camera"].string_value

        # Percentage of pixels that must differ to count as a change.
        # Default is 5% (0.05). Lower values are more sensitive.
        self.threshold = attrs["threshold"].number_value if "threshold" in attrs else 0.05

        self.last_frame = None
        self.last_image = None

    def _get_source_camera(
        self, dependencies: Mapping[ResourceName, ResourceBase]
    ) -> Camera:
        for name, dep in dependencies.items():
            if name.name == self.source_camera_name:
                return dep
        raise ValueError(
            f"Source camera '{self.source_camera_name}' not found in dependencies"
        )

    def _frame_changed(self, current: np.ndarray) -> bool:
        """Compare current frame to previous frame.

        Returns True if the frames differ by more than the configured threshold.
        Uses mean absolute difference across all channels.
        """
        if self.last_frame is None:
            return True

        if current.shape != self.last_frame.shape:
            return True

        # Compute per-pixel absolute difference, normalized to [0, 1].
        diff = np.abs(
            current.astype(np.float32) - self.last_frame.astype(np.float32)
        ) / 255.0

        # Fraction of pixels where any channel changed by more than 10%.
        changed_pixels = np.mean(np.any(diff > 0.1, axis=-1))

        return changed_pixels > self.threshold

    async def get_image(
        self,
        mime_type: str = "",
        *,
        extra: Optional[Dict[str, Any]] = None,
        timeout: Optional[float] = None,
        **kwargs,
    ) -> ViamImage:
        # Get the source camera from the robot.
        source = self.robot.get_component(
            Camera.get_resource_name(self.source_camera_name)
        )
        current_image = await source.get_image(mime_type=mime_type)

        # Convert to numpy array for comparison.
        pil_image = current_image.pil_image if hasattr(current_image, 'pil_image') else Image.open(current_image)
        current_array = np.array(pil_image)

        if self._frame_changed(current_array):
            self.last_frame = current_array
            self.last_image = current_image

        # Return the last "interesting" frame.
        # If nothing has changed, this returns the previous interesting frame,
        # and the data capture service will store a duplicate -- but since
        # consecutive duplicates are the same image, storage impact is minimal.
        # For stricter deduplication, raise an exception or return None
        # to signal "no new data."
        return self.last_image

    async def get_images(
        self,
        *,
        timeout: Optional[float] = None,
        **kwargs,
    ) -> Tuple[List[NamedImage], ResponseMetadata]:
        source = self.robot.get_component(
            Camera.get_resource_name(self.source_camera_name)
        )
        return await source.get_images(timeout=timeout)

    async def get_point_cloud(
        self,
        *,
        extra: Optional[Dict[str, Any]] = None,
        timeout: Optional[float] = None,
        **kwargs,
    ) -> Tuple[bytes, str]:
        source = self.robot.get_component(
            Camera.get_resource_name(self.source_camera_name)
        )
        return await source.get_point_cloud(extra=extra, timeout=timeout)

    async def get_properties(
        self,
        *,
        timeout: Optional[float] = None,
        **kwargs,
    ) -> Camera.Properties:
        source = self.robot.get_component(
            Camera.get_resource_name(self.source_camera_name)
        )
        return await source.get_properties(timeout=timeout)


if __name__ == "__main__":
    asyncio.run(Module.run_from_registry())
```

Save this as `meta.json` in the same directory:

```json
{
  "module_id": "custom:filtered-camera",
  "visibility": "private",
  "description": "A camera module that only captures when visual change is detected.",
  "models": [
    {
      "api": "rdk:component:camera",
      "model": "custom:camera:filtered"
    }
  ],
  "entrypoint": "main.py"
}
```

Save this as `requirements.txt`:

```
viam-sdk
numpy
Pillow
```

{{% /tab %}}
{{% tab name="Go" %}}

Create a directory for your module:

```bash
mkdir -p filtered-camera-go
cd filtered-camera-go
go mod init filtered-camera-go
go get go.viam.com/rdk
```

Save this as `main.go`:

```go
package main

import (
	"context"
	"image"
	"image/color"
	"math"

	"go.viam.com/rdk/components/camera"
	"go.viam.com/rdk/gostream"
	"go.viam.com/rdk/logging"
	"go.viam.com/rdk/module"
	"go.viam.com/rdk/pointcloud"
	"go.viam.com/rdk/resource"
	"go.viam.com/utils"
)

var Model = resource.NewModel("custom", "camera", "filtered")

func init() {
	resource.RegisterComponent(camera.API, Model,
		resource.Registration[camera.Camera, *Config]{
			Constructor: newFilteredCamera,
		},
	)
}

// Config holds the configuration for the filtered camera.
type Config struct {
	SourceCamera string  `json:"source_camera"`
	Threshold    float64 `json:"threshold"`
}

// Validate checks that the config is valid.
func (c *Config) Validate(path string) ([]string, error) {
	if c.SourceCamera == "" {
		return nil, utils.NewConfigValidationFieldRequiredError(path, "source_camera")
	}
	if c.Threshold == 0 {
		c.Threshold = 0.05
	}
	return []string{c.SourceCamera}, nil
}

type filteredCamera struct {
	resource.Named
	resource.AlwaysRebuild

	source    camera.Camera
	lastFrame image.Image
	threshold float64
	logger    logging.Logger
}

func newFilteredCamera(
	ctx context.Context,
	deps resource.Dependencies,
	conf resource.Config,
	logger logging.Logger,
) (camera.Camera, error) {
	cfg, err := resource.NativeConfig[*Config](conf)
	if err != nil {
		return nil, err
	}

	src, err := camera.FromDependencies(deps, cfg.SourceCamera)
	if err != nil {
		return nil, err
	}

	threshold := cfg.Threshold
	if threshold == 0 {
		threshold = 0.05
	}

	return &filteredCamera{
		Named:     conf.ResourceName().AsNamed(),
		source:    src,
		threshold: threshold,
		logger:    logger,
	}, nil
}

// frameChanged compares two images and returns true if they differ
// by more than the configured threshold.
func (fc *filteredCamera) frameChanged(current image.Image) bool {
	if fc.lastFrame == nil {
		return true
	}

	curBounds := current.Bounds()
	lastBounds := fc.lastFrame.Bounds()
	if curBounds != lastBounds {
		return true
	}

	totalPixels := curBounds.Dx() * curBounds.Dy()
	changedPixels := 0

	for y := curBounds.Min.Y; y < curBounds.Max.Y; y++ {
		for x := curBounds.Min.X; x < curBounds.Max.X; x++ {
			r1, g1, b1, _ := fc.lastFrame.At(x, y).RGBA()
			r2, g2, b2, _ := current.At(x, y).RGBA()

			// RGBA() returns values in [0, 65535]. Normalize to [0, 1].
			dr := math.Abs(float64(r1)-float64(r2)) / 65535.0
			dg := math.Abs(float64(g1)-float64(g2)) / 65535.0
			db := math.Abs(float64(b1)-float64(b2)) / 65535.0

			// If any channel changed by more than 10%, count this pixel.
			if dr > 0.1 || dg > 0.1 || db > 0.1 {
				changedPixels++
			}
		}
	}

	fraction := float64(changedPixels) / float64(totalPixels)
	fc.logger.Debugf("frame change: %.2f%% of pixels changed (threshold: %.2f%%)",
		fraction*100, fc.threshold*100)

	return fraction > fc.threshold
}

func (fc *filteredCamera) Images(ctx context.Context) ([]camera.NamedImage, resource.ResponseMetadata, error) {
	images, meta, err := fc.source.Images(ctx)
	if err != nil {
		return nil, meta, err
	}

	if len(images) > 0 && fc.frameChanged(images[0].Image) {
		fc.lastFrame = images[0].Image
	}

	// Return the last interesting frame set.
	if fc.lastFrame != nil && !fc.frameChanged(images[0].Image) {
		// No significant change -- return the stored frame.
		return images, meta, nil
	}

	return images, meta, nil
}

func (fc *filteredCamera) Stream(ctx context.Context, errHandlers ...gostream.ErrorHandler) (gostream.VideoStream, error) {
	return fc.source.Stream(ctx, errHandlers...)
}

func (fc *filteredCamera) PointCloud(ctx context.Context, extra map[string]interface{}) (pointcloud.PointCloud, error) {
	return fc.source.PointCloud(ctx, extra)
}

func (fc *filteredCamera) Properties(ctx context.Context) (camera.Properties, error) {
	return fc.source.Properties(ctx)
}

func (fc *filteredCamera) Close(ctx context.Context) error {
	return nil
}

func (fc *filteredCamera) DoCommand(ctx context.Context, cmd map[string]interface{}) (map[string]interface{}, error) {
	return map[string]interface{}{}, nil
}

func main() {
	utils.ContextualMain(func(ctx context.Context, logger logging.Logger) error {
		mod, err := module.NewModuleFromArgs(ctx, logger)
		if err != nil {
			return err
		}
		if err := mod.AddModelFromRegistry(ctx, camera.API, Model); err != nil {
			return err
		}
		err = mod.Start(ctx)
		defer mod.Close(ctx)
		if err != nil {
			return err
		}
		<-ctx.Done()
		return nil
	}, logger)
}
```

{{% /tab %}}
{{< /tabs >}}

#### Configure the filtered camera

After deploying the module to your machine (see
[Deploy a Module](/build/development/deploy-a-module/) for full deployment
instructions), add it to your machine configuration:

1. In the Viam app, go to your machine's **CONFIGURE** tab.
2. Click **+** and select **Component**.
3. Select **camera** as the type and choose your custom `custom:camera:filtered`
   model.
4. Name it `filtered-camera`.
5. Set the attributes:

```json
{
  "source_camera": "my-camera",
  "threshold": 0.05
}
```

6. Add data capture on `filtered-camera` (not the raw `my-camera`). Configure
   `GetImages` at your desired frequency.
7. Optionally, remove data capture from `my-camera` to avoid capturing duplicate
   raw frames.
8. Click **Save**.

Now only frames where meaningful visual change is detected will be captured and
synced.

{{< alert title="Tip" color="tip" >}}

**Tuning the threshold:** A threshold of `0.05` means 5% of pixels must change
significantly. Increase it (e.g., `0.15`) to require more change before
capturing. Decrease it (e.g., `0.01`) to capture on subtle changes. Test with
your actual scene to find the right value.

{{< /alert >}}

### 4. Filter sensor data by threshold

For sensor data, you often only care about readings that cross a specific
boundary -- a temperature above 50 degrees C, a distance below 30 cm, or
humidity above 80%. You can implement this with a filtering wrapper module
similar to the camera approach, but for sensors.

Here is a Python sensor filter that only reports readings when a value exceeds a
threshold:

```python
import asyncio
from typing import Any, ClassVar, Dict, Mapping, Optional

from viam.components.sensor import Sensor
from viam.module.module import Module
from viam.proto.app.robot import ComponentConfig
from viam.proto.common import ResourceName
from viam.resource.base import ResourceBase
from viam.resource.easy_resource import EasyResource
from viam.resource.types import Model, ModelFamily


class ThresholdSensor(Sensor, EasyResource):
    """A sensor wrapper that only returns readings above a configured threshold."""

    MODEL: ClassVar[Model] = Model(
        ModelFamily("custom", "sensor"), "threshold"
    )

    source_sensor_name: str
    field: str
    min_value: float
    last_interesting_reading: Optional[Dict[str, Any]]

    @classmethod
    def new(
        cls,
        config: ComponentConfig,
        dependencies: Mapping[ResourceName, ResourceBase],
    ) -> "ThresholdSensor":
        sensor = cls(config.name)
        sensor.reconfigure(config, dependencies)
        return sensor

    def reconfigure(
        self,
        config: ComponentConfig,
        dependencies: Mapping[ResourceName, ResourceBase],
    ) -> None:
        attrs = config.attributes.fields
        self.source_sensor_name = attrs["source_sensor"].string_value
        self.field = attrs["field"].string_value
        self.min_value = attrs["min_value"].number_value
        self.last_interesting_reading = None

    async def get_readings(
        self,
        *,
        extra: Optional[Dict[str, Any]] = None,
        timeout: Optional[float] = None,
        **kwargs,
    ) -> Mapping[str, Any]:
        source = self.robot.get_component(
            Sensor.get_resource_name(self.source_sensor_name)
        )
        readings = await source.get_readings(extra=extra, timeout=timeout)

        value = readings.get(self.field)
        if value is not None and float(value) >= self.min_value:
            self.last_interesting_reading = dict(readings)

        # Return the last reading that exceeded the threshold.
        # If no reading has ever exceeded it, return the current
        # reading with a flag indicating it's below threshold.
        if self.last_interesting_reading is not None:
            return self.last_interesting_reading

        return {**readings, "_below_threshold": True}


if __name__ == "__main__":
    asyncio.run(Module.run_from_registry())
```

Configure it with:

```json
{
  "source_sensor": "my-temp-sensor",
  "field": "temperature_celsius",
  "min_value": 50.0
}
```

Then configure data capture on the `threshold-sensor` component instead of the
raw sensor. Only readings where the temperature reaches or exceeds 50 degrees C
will produce new captured data.

### 5. Manage local storage

On constrained machines (Raspberry Pi, Jetson Nano, single-board computers),
local storage is limited. Understanding how Viam manages the capture directory
helps you avoid filling the disk.

#### How local storage works

- Captured data is written to `~/.viam/capture` by default.
- Each capture creates a file (JPEG for images, JSON for tabular data).
- The sync process uploads files and deletes them after successful upload.
- If sync is disabled or the network is down, files accumulate locally.

#### Configure storage limits

You can configure the maximum storage that data capture will use on disk. In
your data management service configuration, set the `maximum_capture_file_size`
attribute to limit the size of individual capture files, and monitor the overall
capture directory size.

To check current disk usage of the capture directory:

```bash
du -sh ~/.viam/capture
```

To monitor it over time:

```bash
watch -n 60 du -sh ~/.viam/capture
```

#### Best practices for constrained machines

- **Use the lowest capture frequency that meets your needs.** Every reduction in
  frequency directly reduces storage pressure.
- **Enable sync.** Without sync, local storage fills indefinitely.
- **Use filtered capture.** The filtered camera and threshold sensor modules
  above reduce what gets written to disk, not just what gets synced.
- **Monitor disk space.** Set up a simple cron job or health check that alerts
  you when the capture directory exceeds a size threshold.
- **Adjust sync interval.** A shorter sync interval (e.g., every 30 seconds)
  keeps the local buffer small. A longer interval (e.g., every 5 minutes)
  reduces network requests but requires more local storage.

## Try It

### Verify your filtering is working

1. Configure one of the filtering techniques above on your machine.
2. Open the **DATA** tab in the Viam app.
3. Compare data volume before and after filtering:
   - Check how many new entries appear per minute with your filter active.
   - Compare to the rate you saw in
     [Capture and Sync Data](/build/foundation/capture-and-sync-data/) before
     filtering.
4. For the filtered camera module, verify that captured images show meaningful
   variation between consecutive frames. If consecutive frames look identical,
   your threshold may be too low.

### Test threshold sensitivity

If you are using the filtered camera module:

1. Start with `threshold: 0.05` and confirm that normal scene activity
   (people walking, objects moving) triggers new captures.
2. Increase to `threshold: 0.15` and verify that only large changes
   (someone entering the frame, lighting changes) trigger captures.
3. Decrease to `threshold: 0.01` and observe that even subtle changes
   (shadows shifting, camera noise) trigger captures.

Pick the value that best matches your use case.

## Troubleshooting

{{< expand "Filtered camera module not loading" >}}

- Check the module logs in the Viam app (**LOGS** tab) for import errors or
  configuration issues.
- Verify that the `source_camera` name in the filtered camera attributes matches
  the name of your actual camera component exactly. Names are case-sensitive.
- Ensure all Python dependencies (`numpy`, `Pillow`, `viam-sdk`) are installed
  in the module's environment.

{{< /expand >}}

{{< expand "No data captured after enabling filter" >}}

- Your threshold may be too high. Try lowering it (e.g., from `0.05` to
  `0.01`) to see if frames start coming through.
- Confirm the source camera is still working. Check the raw camera's test panel
  in the Viam app.
- Verify that data capture is configured on the filtered camera component, not
  only on the raw camera.

{{< /expand >}}

{{< expand "Too much data still being captured" >}}

- Increase the threshold value to require more change before capturing.
- Reduce the capture frequency on the filtered camera. Even with filtering, a
  high capture frequency means more comparisons and more opportunities for a
  frame to be flagged as changed.
- For sensor data, verify that `min_value` is set correctly and that the
  `field` name matches the actual key in the sensor readings.

{{< /expand >}}

{{< expand "Local disk filling up despite sync being enabled" >}}

- Check your network connection. If the machine cannot reach Viam's cloud, data
  accumulates locally.
- Verify that both **Capturing** and **Syncing** are enabled in the data
  management service configuration.
- Check the sync interval. A very long sync interval combined with high capture
  frequency can cause the local buffer to grow between sync cycles.
- Run `du -sh ~/.viam/capture` to check current usage.

{{< /expand >}}

{{< expand "Frame comparison is too slow" >}}

- The numpy-based comparison in the Python module processes full-resolution
  frames. If your camera outputs high-resolution images (1080p or above), the
  comparison may add latency.
- Resize frames before comparison: add
  `current_array = np.array(pil_image.resize((320, 240)))` to compare
  downscaled versions while still capturing the full-resolution image.
- The Go implementation is generally faster for pixel-level operations.

{{< /expand >}}

## What's Next

- [Query Data](/build/data/query-data/) -- write SQL and MQL queries against your
  filtered, high-signal dataset.
- [Create a Dataset](/build/train/create-a-dataset/) -- use filtered captures to
  build cleaner training datasets for ML models.
- [Write a Module](/build/development/write-a-module/) -- learn more about
  building and deploying custom modules on Viam.
