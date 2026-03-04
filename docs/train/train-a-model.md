---
linkTitle: "Train a Model"
title: "Train a Model"
weight: 20
layout: "docs"
type: "docs"
description: "Train a classification or object detection model from a labeled dataset."
date: "2025-01-30"
aliases:
  - /build/train/train-a-model/
---

## What Problem This Solves

You have a labeled dataset with images tagged or annotated with bounding boxes.
Now you need to turn that data into a model your machine can run. Training takes
your labeled examples and produces a model file that can classify new images or
detect objects in real time.

Viam handles the compute. You do not need to provision GPUs, install TensorFlow,
or write training scripts. You select a dataset, pick a model type and task,
and start the job. When training completes, the model is available to deploy to
any machine in your organization.

## Concepts

### ML model types

Viam supports two model frameworks:

**TensorFlow Lite (TFLite)** produces compact models optimized for edge devices.
TFLite models run directly on single-board computers like the Raspberry Pi
without requiring a GPU. Use TFLite when your model will run on the machine
itself, which is the most common case.

**TensorFlow (TF)** produces larger models that require more compute resources.
Use TF when you are running on hardware with a capable CPU or GPU and need the
additional model capacity.

For most Viam use cases -- quality inspection, object detection on a camera
feed, simple classification -- TFLite is the right choice.

### Task types

The task type determines what the model learns to do. It must match how you
labeled your dataset.

**Single Label Classification** assigns exactly one label to each image. Your
dataset should use tags with exactly one tag per image.

**Multi Label Classification** assigns one or more labels to each image. Your
dataset should use tags, and images may have multiple tags.

**Object Detection** finds objects in an image and draws bounding boxes around
them with labels. Your dataset should use bounding box annotations.

### Training in the cloud

When you start a training job, Viam runs the training process on cloud
infrastructure. Your machine does not need to be online during training. The
job runs asynchronously -- you can close the browser and come back later.
Training times vary based on dataset size and model type, but typically range
from a few minutes to an hour.

Training logs are available for 7 days after the job completes.

### Automatic deployment

When training completes, the model is stored in your organization's registry.
If a machine is already configured to use that model, Viam automatically deploys
the new version. The machine pulls the latest version the next time it checks
for updates.

## Steps

### 1. Start a training job from the web UI

1. Go to [app.viam.com](https://app.viam.com).
2. Click the **DATA** tab in the top navigation.
3. Click the **DATASETS** subtab.
4. Click the dataset you want to train on.
5. Click **Train model**.
6. Select the model framework:
   - **TFLite** for edge devices (recommended for most use cases)
   - **TF** for general-purpose models requiring more compute
7. Enter a name for your model. Use a descriptive name like
   `part-inspector-v1` or `package-detector-v1`. This name identifies the model
   in your organization's registry.
8. Select the task type:
   - **Single Label Classification** if each image has one tag
   - **Multi Label Classification** if images have multiple tags
   - **Object Detection** if you used bounding box annotations
9. Select which labels to include in training. You can exclude labels that have
   too few examples or that you do not want the model to learn.
10. Click **Train model**.

The training job starts. You will see a confirmation message with the job ID.

### 2. Start a training job from the CLI

If you prefer the command line, use the Viam CLI:

```bash
viam train submit managed \
  --dataset-id=YOUR-DATASET-ID \
  --model-name=part-inspector-v1 \
  --model-type=tflite_classifier \
  --org-id=YOUR-ORG-ID
```

Available model types:

| Flag value | Framework | Task |
|------------|-----------|------|
| `tflite_classifier` | TFLite | Single or Multi Label Classification |
| `tflite_detector` | TFLite | Object Detection |
| `tf_classifier` | TensorFlow | Single or Multi Label Classification |
| `tf_detector` | TensorFlow | Object Detection |

The command returns a training job ID that you can use to check status.

### 3. Monitor training progress

**Web UI:**

1. In the Viam app, click the **DATA** tab.
2. Click the **TRAINING** subtab.
3. You will see a list of training jobs with their status:
   - **Pending** -- the job is queued
   - **Running** -- training is in progress
   - **Completed** -- the model is ready
   - **Failed** -- something went wrong
4. Click a job ID to view detailed logs.

**CLI:**

Check the status of a training job:

```bash
viam train get --job-id=YOUR-JOB-ID
```

View training logs:

```bash
viam train logs --job-id=YOUR-JOB-ID
```

Training logs expire after 7 days. If you need to retain logs for longer,
copy them before they expire.

{{< tabs >}}
{{% tab name="Python" %}}

```python
async def main():
    viam_client = await connect()
    ml_training_client = viam_client.ml_training_client

    job = await ml_training_client.get_training_job(
        id="YOUR-TRAINING-JOB-ID",
    )
    print(f"Status: {job.status}")
    print(f"Model name: {job.model_name}")
    print(f"Created: {job.created_on}")

    viam_client.close()
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
job, err := mlTrainingClient.GetTrainingJob(ctx, "YOUR-TRAINING-JOB-ID")
if err != nil {
	logger.Fatal(err)
}
fmt.Printf("Status: %s\n", job.Status)
fmt.Printf("Model name: %s\n", job.ModelName)
fmt.Printf("Created: %s\n", job.CreatedOn)
```

{{% /tab %}}
{{< /tabs >}}

### 4. Test your model

After training completes, test the model against images before deploying it.

1. In the Viam app, click the **DATA** tab.
2. Click an image to open the detail view. Choose images the model has not seen
   during training for a more realistic test.
3. Click **Actions** in the image toolbar.
4. Click **Run model**.
5. Select your trained model from the dropdown.
6. Set a **confidence threshold** (0.0 to 1.0). Start with 0.5 and adjust:
   - Lower the threshold to see more predictions (including less certain ones)
   - Raise the threshold to see only high-confidence predictions
7. Click **Run model**.

Test with a variety of images:

- Images that clearly belong to each class (should get high confidence)
- Ambiguous images (helps you understand the model's decision boundary)
- Images from conditions not in the training set (reveals generalization gaps)

### 5. Deploy the model to a machine

Deploying a model requires two service configurations: an ML model service that
loads the model file, and a vision service that uses the ML model to process
camera frames.

**Configure the ML model service:**

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

- `model`: Use `tflite_cpu` for TFLite models. This runs inference on the CPU,
  which works on all hardware including Raspberry Pi.
- `model_path`: The `${packages.my-model}` syntax references a model package
  from your organization's registry. Replace `my-model` with your model's
  package name.

**Configure the vision service:**

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

- `mlmodel_name`: Must match the `name` of the ML model service you configured
  above.

**Verify in the web UI:**

1. Save your configuration.
2. Navigate to the **CONTROL** tab.
3. Find your vision service and click **Test**.
4. Select a camera to run the model against.
5. You should see live classifications or detections overlaid on the camera
   feed.

**Use the model in code:**

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.services.vision import VisionClient

# Get the vision service (assumes you have a robot connection)
vision = VisionClient.from_robot(robot, "my-detector")

# For classification
classifications = await vision.get_classifications(
    image=my_image,
    count=5,
)
for c in classifications:
    print(f"  {c.class_name}: {c.confidence:.2f}")

# For object detection
detections = await vision.get_detections(image=my_image)
for d in detections:
    print(f"  {d.class_name}: {d.confidence:.2f} "
          f"at ({d.x_min}, {d.y_min}) to ({d.x_max}, {d.y_max})")
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
import "go.viam.com/rdk/services/vision"

// Get the vision service (assumes you have a robot connection)
visionSvc, err := vision.FromRobot(robot, "my-detector")
if err != nil {
	logger.Fatal(err)
}

// For classification
classifications, err := visionSvc.Classifications(ctx, myImage, 5, nil)
if err != nil {
	logger.Fatal(err)
}
for _, c := range classifications {
	fmt.Printf("  %s: %.2f\n", c.Label(), c.Score())
}

// For object detection
detections, err := visionSvc.Detections(ctx, myImage, nil)
if err != nil {
	logger.Fatal(err)
}
for _, d := range detections {
	box := d.BoundingBox()
	fmt.Printf("  %s: %.2f at (%d, %d) to (%d, %d)\n",
		d.Label(), d.Score(),
		box.Min.X, box.Min.Y, box.Max.X, box.Max.Y)
}
```

{{% /tab %}}
{{< /tabs >}}

### 6. Iterate on the model

Your first model is a baseline. Improving it is an ongoing process.

**Add more data.** The single most effective way to improve a model is to give
it more diverse training examples. Focus on:

- **Edge cases:** Images where the model got it wrong or gave low confidence.
- **Counterexamples:** If the model falsely detects objects in empty scenes, add
  images of empty scenes labeled appropriately.
- **Varied conditions:** Capture images under different lighting, angles,
  distances, and backgrounds.

**Retrain.** After updating your dataset, start a new training job. If your
machine is configured to use the model, the new version deploys automatically.

{{< tabs >}}
{{% tab name="Python" %}}

```python
async def main():
    viam_client = await connect()
    ml_training_client = viam_client.ml_training_client

    jobs = await ml_training_client.list_training_jobs(
        organization_id=ORG_ID,
    )
    for job in jobs:
        print(f"Job: {job.id}, Status: {job.status}, "
              f"Model: {job.model_name}, Created: {job.created_on}")

    viam_client.close()
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
jobs, err := mlTrainingClient.ListTrainingJobs(ctx, orgID)
if err != nil {
	logger.Fatal(err)
}
for _, job := range jobs {
	fmt.Printf("Job: %s, Status: %s, Model: %s, Created: %s\n",
		job.ID, job.Status, job.ModelName, job.CreatedOn)
}
```

{{% /tab %}}
{{< /tabs >}}

## Try It

1. Start a training job from the web UI using your labeled dataset.
2. Wait for the training job to complete. Monitor it in the **TRAINING** tab.
3. When training finishes, test the model on several images:
   - Test with clear, obvious examples. Expect high confidence scores (above
     0.8).
   - Test with ambiguous or edge-case images. Note where confidence drops.
4. Configure the ML model service and vision service on your machine.
5. Open the **CONTROL** tab and test the vision service with a live camera
   feed.
6. If results are not satisfactory, add more images to your dataset and retrain.

## Troubleshooting

{{< expand "Training job fails" >}}

- **Check the training logs.** In the **TRAINING** tab, click the failed job ID
  to view logs. The error message usually indicates the problem.
- **Dataset too small.** Training requires at least 15 images with at least 80%
  labeled. Check your dataset in the **DATASETS** tab.
- **No labels selected.** You must select at least two labels. A model cannot
  learn to classify if there is only one category.
- **Bounding box format issue.** For object detection, verify that bounding box
  coordinates are normalized between 0.0 and 1.0, and `x_min` is less than
  `x_max`.

{{< /expand >}}

{{< expand "Low confidence scores" >}}

- **Add more training data.** Low confidence usually means the model has not seen
  enough examples. More diverse images of each class will help.
- **Check label balance.** If one label dominates the dataset, the model may
  assign low confidence to minority labels. Balance the dataset and retrain.
- **Verify image quality.** Blurry, dark, or low-resolution images make it
  harder for the model to learn distinctive features.
- **Lower the confidence threshold.** If the model is correct but with scores
  around 0.4-0.6, your threshold may be set too high.

{{< /expand >}}

{{< expand "Model not appearing on the machine" >}}

- **Check the ML model service configuration.** Verify that `model_path` uses
  the correct `${packages.<name>}` syntax and that the package name matches your
  model's name in the registry.
- **Restart viam-server.** In some cases, the machine may need to restart to
  pick up a new model version.
- **Check machine connectivity.** The machine must be online and connected to
  the cloud to download model updates.

{{< /expand >}}

{{< expand "Vision service returns no results" >}}

- **Verify the mlmodel_name.** The `mlmodel_name` in the vision service must
  exactly match the `name` of the ML model service.
- **Check the camera.** Verify that the camera is working in the **CONTROL** tab
  before testing the vision service.
- **Check the confidence threshold.** Try lowering the threshold. The model may
  be producing results below your current threshold.

{{< /expand >}}

{{< expand "Training takes too long" >}}

- **Large datasets take longer.** A dataset with thousands of images may take an
  hour or more. This is normal.
- **TF models take longer than TFLite.** If training time is a concern, switch
  to TFLite.
- **Training is queued.** If the status stays at "Pending", the training
  infrastructure may be busy. Jobs are processed in order.

{{< /expand >}}

## What's Next

- [Add Computer Vision](/vision-detection/add-computer-vision/) -- build
  applications that use your trained model to process live camera feeds.
- [Detect Objects (2D)](/vision-detection/detect-objects-2d/) -- use your
  object detection model to find and locate objects in camera images.
- [Classify Objects](/build/stationary-vision/classify-objects/) -- use your
  classification model to categorize images from your machine's camera.
