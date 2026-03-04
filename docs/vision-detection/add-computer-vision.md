---
linkTitle: "Add Computer Vision"
title: "Add Computer Vision"
weight: 10
layout: "docs"
type: "docs"
description: "Configure an ML model service and a vision service to give your machine the ability to detect or classify objects."
date: "2025-01-30"
aliases:
  - /build/vision-detection/add-computer-vision/
---

## What Problem This Solves

You have a trained machine learning model and a camera, and you want your machine to understand what it sees. This how-to configures both an ML model service and a vision service so that downstream how-to guides -- Detect Objects, Classify Objects, Track Objects -- have a working vision pipeline to build on.

## Concepts

### The two-service architecture

Most robotics platforms handle ML inference as a monolithic block: one configuration entry that loads a model and runs it against a camera. Viam splits this into two services for a reason.

The ML model service handles the mechanics of model loading: reading the file, allocating memory, preparing the inference runtime. The vision service handles the semantics: what does "run a detection" mean, how do you map model outputs to bounding boxes, how do you associate results with camera frames.

This separation means:

- You can update your model (retrain, swap architectures) without changing your vision service configuration.
- You can run the same model against multiple cameras by creating multiple vision services that reference the same ML model service.
- Different model formats (TFLite, ONNX, TensorFlow, PyTorch) are handled by different ML model service implementations, but the vision service API stays the same.

### ML model service

The ML model service loads a model file and exposes it for inference. The most common implementation is `tflite_cpu`, which runs TensorFlow Lite models on the CPU. Other implementations support ONNX, TensorFlow SavedModel, and PyTorch.

When you deploy a model from the Viam registry, the model files are downloaded to your machine and referenced via the `${packages.model-name}` variable. You can also point directly to a local file path if you have a model file on disk.

The `tflite_cpu` implementation runs entirely on the CPU, so it works on any hardware -- no GPU required. For Raspberry Pi, Jetson Nano, and other edge machines, this is the standard approach. Performance depends on the model size: a MobileNet-based detector typically runs at 5-10 frames per second on a Raspberry Pi 4.

### Vision service

The vision service connects an ML model to a camera and provides high-level APIs for detection and classification. The `mlmodel` implementation takes detections or classifications from the ML model and returns them as structured data: bounding boxes with labels and confidence scores for detections, or label-confidence pairs for classifications.

The vision service is what your code interacts with. You never call the ML model service directly from application code. This means switching from a detection model to a classification model requires only a configuration change to the ML model service, not a rewrite of your application.

The vision service also handles the image capture pipeline. When you call `GetDetectionsFromCamera`, the vision service captures a frame from the specified camera, converts it to the format expected by the model, runs inference, and returns structured results. You do not need to manage image capture and model input formatting yourself.

### Supported model formats

| Format | ML model implementation | Notes |
|--------|------------------------|-------|
| TensorFlow Lite (.tflite) | `tflite_cpu` | Most common for edge deployment |
| ONNX (.onnx) | `onnx_cpu` | Cross-framework format |
| TensorFlow SavedModel | `tensorflow_cpu` | Full TensorFlow models |
| PyTorch (.pt) | `torch_cpu` | PyTorch models |

TFLite is the most widely used for on-device inference because of its small size and low resource requirements. If you trained a model through Viam's training pipeline, it is already in TFLite format.

### Labels and label files

ML models output numeric class IDs, not human-readable names. A detection model might output class ID `3`, which means nothing to you. The **label file** maps these IDs to names: class 3 might be "dog", class 7 might be "car".

The label file is a plain text file with one label per line. The line number (zero-indexed) corresponds to the class ID. For example:

```
background
person
bicycle
car
motorcycle
```

In this file, class ID 0 is "background", class ID 1 is "person", and so on. When you configure the ML model service with a `label_path`, detections and classifications will use these human-readable names instead of numeric IDs.

If you do not provide a label file, detections will report numeric class IDs as their labels.

### Model deployment via the Viam registry

The Viam registry is a centralized repository for ML models, modules, and other packages. When you deploy a model from the registry to your machine, Viam downloads the model files and stores them in a known location. You reference them in your configuration using the `${packages.package-name}` syntax.

This approach has several advantages over manually copying model files:

- **Version management.** You can update to a new model version from the Viam app without manually copying files to each machine.
- **Multi-machine deployment.** Deploy the same model to hundreds of machines from one place.
- **Rollback.** If a new model performs worse, revert to the previous version from the Viam app.
- **Consistency.** Every machine in your fleet runs the same model version.

To deploy a model from the registry, go to the **Packages** section of your machine configuration, search for your model, and add it. The package name you assign is what you use in `${packages.package-name}`.

You can also upload your own models to the registry using the Viam CLI or the Viam app. This is the recommended approach for production deployments where you need to manage model versions across a fleet of machines.

## Steps

### 1. Add an ML model service

1. Go to [app.viam.com](https://app.viam.com) and navigate to your machine.
2. Confirm it shows as **Live**.
3. Click the **+** button in the left sidebar.
4. Select **Service**.
5. For the type, select **ML model**.
6. Choose **tflite_cpu** as the model (or the appropriate implementation for your model format).
7. Name it `my-ml-model`.
8. Click **Create**.

### 2. Configure the ML model service

After creating the service, configure it to point to your model file.

**If you deployed a model from the Viam registry:**

```json
{
  "name": "my-ml-model",
  "api": "rdk:service:mlmodel",
  "model": "tflite_cpu",
  "attributes": {
    "model_path": "${packages.my-model}/model.tflite",
    "label_path": "${packages.my-model}/labels.txt"
  }
}
```

The `${packages.my-model}` variable resolves to the directory where the registry package was downloaded. Replace `my-model` with the name of your deployed model package.

**If you have a local model file:**

```json
{
  "name": "my-ml-model",
  "api": "rdk:service:mlmodel",
  "model": "tflite_cpu",
  "attributes": {
    "model_path": "/path/to/your/model.tflite",
    "label_path": "/path/to/your/labels.txt"
  }
}
```

The `model_path` is the path to the model file on the machine where `viam-server` is running. The `label_path` is optional but recommended -- it maps numeric class IDs to human-readable names.

### 3. Add a vision service

1. Click the **+** button in the left sidebar again.
2. Select **Service**.
3. For the type, select **vision**.
4. Choose **mlmodel** as the model.
5. Name it `my-detector`.
6. Click **Create**.

### 4. Configure the vision service

Link the vision service to your ML model service:

```json
{
  "name": "my-detector",
  "api": "rdk:service:vision",
  "model": "mlmodel",
  "attributes": {
    "mlmodel_name": "my-ml-model"
  }
}
```

The `mlmodel_name` must match the name you gave your ML model service in step 1. This is how the vision service knows which model to use for inference.

### 5. Save the configuration

Click **Save** in the upper right. `viam-server` reloads automatically and initializes both services. You do not need to restart anything.

### 6. Test from the CONTROL tab

1. Go to the **CONTROL** tab in the Viam app.
2. Find your camera in the component list and open it.
3. Enable the camera stream.
4. In the vision service overlay dropdown, select your vision service (`my-detector`).
5. You should see bounding boxes drawn on the camera feed if your model is detecting objects in the current frame.

If the camera feed appears but no detections are shown, point the camera at an object your model was trained to recognize. If you are using a general-purpose COCO model, try pointing the camera at a person, a cup, or a keyboard.

### 7. Verify the full configuration

Your machine configuration should now include both services. The relevant sections look like this:

```json
{
  "services": [
    {
      "name": "my-ml-model",
      "api": "rdk:service:mlmodel",
      "model": "tflite_cpu",
      "attributes": {
        "model_path": "${packages.my-model}/model.tflite",
        "label_path": "${packages.my-model}/labels.txt"
      }
    },
    {
      "name": "my-detector",
      "api": "rdk:service:vision",
      "model": "mlmodel",
      "attributes": {
        "mlmodel_name": "my-ml-model"
      }
    }
  ]
}
```

The ML model service must be listed before or alongside the vision service. The vision service depends on the ML model service, and `viam-server` resolves dependencies automatically regardless of order in the configuration file.

## Try It

Verify the full pipeline is working by running a quick detection from code.

Install the SDK if you haven't already:

```bash
pip install viam-sdk
```

{{< tabs >}}
{{% tab name="Python" %}}

Save this as `vision_test.py`:

```python
import asyncio
from viam.robot.client import RobotClient
from viam.services.vision import VisionClient


async def main():
    opts = RobotClient.Options.with_api_key(
        api_key="YOUR-API-KEY",
        api_key_id="YOUR-API-KEY-ID"
    )
    robot = await RobotClient.at_address("YOUR-MACHINE-ADDRESS", opts)

    detector = VisionClient.from_robot(robot, "my-detector")
    detections = await detector.get_detections_from_camera("my-camera")

    print(f"Found {len(detections)} detections:")
    for d in detections:
        print(f"  {d.class_name}: {d.confidence:.2f}")

    await robot.close()

if __name__ == "__main__":
    asyncio.run(main())
```

Run it:

```bash
python vision_test.py
```

{{% /tab %}}
{{% tab name="Go" %}}

```bash
mkdir vision-test && cd vision-test
go mod init vision-test
go get go.viam.com/rdk
```

Save this as `main.go`:

```go
package main

import (
	"context"
	"fmt"

	"go.viam.com/rdk/logging"
	"go.viam.com/rdk/robot/client"
	"go.viam.com/rdk/services/vision"
	"go.viam.com/rdk/utils"
)

func main() {
	ctx := context.Background()
	logger := logging.NewLogger("vision-test")

	robot, err := client.New(ctx, "YOUR-MACHINE-ADDRESS", logger,
		client.WithCredentials(utils.Credentials{
			Type:    utils.CredentialsTypeAPIKey,
			Payload: "YOUR-API-KEY",
		}),
		client.WithAPIKeyID("YOUR-API-KEY-ID"),
	)
	if err != nil {
		logger.Fatal(err)
	}
	defer robot.Close(ctx)

	detector, err := vision.FromRobot(robot, "my-detector")
	if err != nil {
		logger.Fatal(err)
	}

	detections, err := detector.DetectionsFromCamera(ctx, "my-camera", nil)
	if err != nil {
		logger.Fatal(err)
	}

	fmt.Printf("Found %d detections:\n", len(detections))
	for _, d := range detections {
		fmt.Printf("  %s: %.2f\n", d.Label(), d.Score())
	}
}
```

Run it:

```bash
go run main.go
```

{{% /tab %}}
{{< /tabs >}}

You should see a list of detected objects with their confidence scores. If the list is empty, point the camera at objects your model recognizes.

Replace the placeholder values in both examples:

1. In the Viam app, go to your machine's **CONNECT** tab.
2. Select **API keys** and copy your API key and API key ID.
3. Copy the machine address from the same tab.

To confirm the full pipeline is working end-to-end:

1. Run the Python or Go script.
2. Point the camera at an object your model recognizes.
3. Verify the script prints at least one detection with a confidence score above 0.5.
4. Move the object out of frame and run the script again. Verify that no detections (or only low-confidence detections) are returned.

If both tests pass, your vision pipeline is configured correctly and ready for use by downstream how-to guides.

## Troubleshooting

{{< expand "Vision service fails to start" >}}

- Check the `viam-server` logs for error messages. The most common cause is a mismatch between the `mlmodel_name` in the vision service config and the name of the ML model service.
- Verify both services are saved in the configuration.

{{< /expand >}}

{{< expand "ML model service fails to load the model" >}}

- Verify the `model_path` points to a file that exists on the machine. If using `${packages.my-model}`, confirm the model package is deployed to the machine in the **Packages** section of the configuration.
- Check that the model file format matches the ML model implementation. A `.tflite` file requires `tflite_cpu`, not `onnx_cpu`.
- On resource-constrained machines, large models may fail to load due to insufficient memory. Check system memory usage.

{{< /expand >}}

{{< expand "Detections are empty even though the camera works" >}}

- Verify the model is trained to detect the objects in the camera's field of view. A model trained on dogs will not detect cats.
- Check the confidence threshold. Some models produce low-confidence detections that may be filtered out by default. Try lowering the threshold if your vision service configuration supports it.
- Ensure the camera resolution and image format are compatible with the model's expected input. Most TFLite models expect RGB images.

{{< /expand >}}

{{< expand "Detections are inaccurate or noisy" >}}

- The model may need more training data. See [Train a Model](/train/train-a-model/) for improving model quality.
- Lighting conditions affect detection quality significantly. Test under the same lighting conditions the model was trained for.
- If using a general-purpose model, expect lower accuracy than a model trained specifically for your use case.

{{< /expand >}}

{{< expand "\"Model not found\" error in code" >}}

- The service name in your code must match the name in the Viam app configuration exactly. Names are case-sensitive.
- Verify the machine is online and `viam-server` is running.
- Check that you are connecting to the correct machine address.

{{< /expand >}}

{{< expand "Multiple vision services on the same model" >}}

If you configured two vision services that reference the same ML model service, and one works while the other does not, check the `mlmodel_name` attribute in both. A common mistake is pointing both vision services at each other instead of at the ML model service.

{{< /expand >}}

{{< expand "Model loads but inference is very slow" >}}

- On a Raspberry Pi 4, a typical TFLite model runs inference in 100-500 ms per frame. This is normal for CPU-based inference.
- Larger models take longer. If you need faster inference, use a smaller model architecture (MobileNet is faster than EfficientDet) or reduce the input image resolution.
- Check CPU usage on the machine. If other processes are consuming CPU, inference will be slower. Use `top` or `htop` to monitor.

{{< /expand >}}

{{< expand "Label file format errors" >}}

- The label file must be plain text with one label per line. No headers, no commas, no quotes.
- The number of labels must match the number of output classes in the model. If your model outputs 80 classes but your label file has 79 lines, the last class will have no name.
- Encoding must be UTF-8. Non-ASCII characters in labels may cause issues on some systems.

{{< /expand >}}

## What's Next

- [Detect Objects (2D)](/vision-detection/detect-objects-2d/) -- use the vision service to get bounding boxes, filter by confidence, and process detection results in code.
- [Classify Objects](/vision-detection/classify-objects/) -- use the vision service for whole-image classification instead of per-object detection.
- [Train a Model](/train/train-a-model/) -- train a custom model on your own data for better accuracy on your specific use case.
