---
linkTitle: "Create a Dataset"
title: "Create a Dataset"
weight: 10
layout: "docs"
type: "docs"
description: "Create a labeled dataset of images for training an ML model."
date: "2025-01-30"
aliases:
  - /build/train/create-a-dataset/
---

## What Problem This Solves

You have images in the cloud from your machine's camera. Raw images are not enough
to train an ML model -- you need a curated, labeled collection (a dataset) that
teaches the model what to look for. This how-to walks you through creating a
dataset, adding images, labeling them for classification or object detection, and
verifying quality before training.

## Concepts

### What a dataset is

A dataset in Viam is a named collection of data -- typically images -- that you
use to train an ML model. Datasets live at the organization level. You can add
images from any machine in your organization to a single dataset, which is useful
when you have multiple machines capturing data in different environments.

Datasets are versioned. Each time you train a model from a dataset, Viam
snapshots the dataset's contents at that point. You can continue adding images
and labels to a dataset after training without affecting previously trained
models.

### Dataset requirements

For training to succeed, your dataset must meet minimum quality thresholds.
These are not arbitrary -- they reflect what the training algorithms need to
generalize effectively.

| Requirement | Minimum | Why it matters |
|-------------|---------|----------------|
| Total images | 15 | Fewer than this and the model cannot learn meaningful patterns |
| Labeled images | 80% of total | Unlabeled images contribute nothing to training |
| Examples per label | 10 | Each class needs enough examples to learn variation |
| Label distribution | Roughly equal | A dataset with 100 "good" and 5 "defective" images will bias the model toward predicting "good" |
| Image source | Production environment | Images from your desk will not prepare the model for factory floor lighting |

More data is better. These are minimums. For production use, aim for hundreds of
images per label, captured under the full range of conditions your model will
encounter -- different lighting, angles, distances, and background clutter.

### Annotation types

Viam supports two types of annotations, each corresponding to a different ML
task:

**Tags** are for classification. A tag applies to the entire image. You tag an
image as "good-part" or "defective-part" -- the model learns to assign one of
those labels to any new image. Use tags when you care about what is in the
image, not where it is.

**Bounding boxes** are for object detection. A bounding box is a rectangle drawn
around a specific object in the image, with a label attached. The model learns
to find objects and draw boxes around them. Use bounding boxes when you need to
know both what objects are present and where they are in the frame.

You can mix both annotation types in the same dataset, but each training job
uses one or the other. A classification model trains on tags. An object
detection model trains on bounding boxes.

## Steps

### 1. Create a dataset

You can create a dataset from the web UI, the CLI, or programmatically.

**Web UI:**

1. Go to [app.viam.com](https://app.viam.com).
2. Click the **DATA** tab in the top navigation.
3. Click the **DATASETS** subtab.
4. Click **+ Create dataset**.
5. Enter a descriptive name for your dataset. Use a name that reflects the task,
   such as `inspection-parts-v1` or `package-detection`. Dataset names must be
   unique within your organization.
6. Click **Create**.

Your empty dataset now appears in the list.

**CLI:**

If you have the Viam CLI installed, create a dataset from the command line:

```bash
viam dataset create --org-id=YOUR-ORG-ID --name=my-inspection-dataset
```

The command returns the dataset ID, which you will need for subsequent CLI and
SDK operations.

{{< tabs >}}
{{% tab name="Python" %}}

```python
import asyncio
from viam.rpc.dial import DialOptions, Credentials
from viam.app.viam_client import ViamClient


API_KEY = "YOUR-API-KEY"
API_KEY_ID = "YOUR-API-KEY-ID"
ORG_ID = "YOUR-ORGANIZATION-ID"


async def connect() -> ViamClient:
    dial_options = DialOptions(
        credentials=Credentials(
            type="api-key",
            payload=API_KEY,
        ),
        auth_entity=API_KEY_ID,
    )
    return await ViamClient.create_from_dial_options(dial_options)


async def main():
    viam_client = await connect()
    data_client = viam_client.data_client

    dataset_id = await data_client.create_dataset(
        name="my-inspection-dataset",
        organization_id=ORG_ID,
    )
    print(f"Created dataset: {dataset_id}")

    viam_client.close()


if __name__ == "__main__":
    asyncio.run(main())
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
package main

import (
	"context"
	"fmt"

	"go.viam.com/rdk/app"
	"go.viam.com/rdk/logging"
)

func main() {
	apiKey := "YOUR-API-KEY"
	apiKeyID := "YOUR-API-KEY-ID"
	orgID := "YOUR-ORGANIZATION-ID"

	ctx := context.Background()
	logger := logging.NewDebugLogger("create-dataset")

	viamClient, err := app.CreateViamClientWithAPIKey(
		ctx, app.Options{}, apiKey, apiKeyID, logger)
	if err != nil {
		logger.Fatal(err)
	}
	defer viamClient.Close()

	dataClient := viamClient.DataClient()

	datasetID, err := dataClient.CreateDataset(
		ctx, "my-inspection-dataset", orgID)
	if err != nil {
		logger.Fatal(err)
	}
	fmt.Printf("Created dataset: %s\n", datasetID)
}
```

{{% /tab %}}
{{< /tabs >}}

Replace all placeholder values (`YOUR-API-KEY`, `YOUR-API-KEY-ID`,
`YOUR-ORGANIZATION-ID`) with your actual values. Find your organization ID in
the Viam app under **Settings** in the left navigation.

### 2. Add images to the dataset

With a dataset created, you need to populate it with images.

**Web UI:**

1. Click the **DATA** tab in the top navigation.
2. Use the filters to find the images you want. Filter by machine, component,
   time range, or tags.
3. Select individual images by clicking their checkboxes, or use **Select all**
   to select all visible images.
4. Click **Add to dataset** in the action bar that appears.
5. Select your dataset from the dropdown.
6. Click **Add**.

The selected images are now part of your dataset.

**CLI:**

Add images to a dataset using filter criteria:

```bash
viam dataset data add filter \
  --dataset-id=YOUR-DATASET-ID \
  --location-id=YOUR-LOCATION-ID \
  --tags=label1,label2
```

This adds all images matching the filter to the dataset. You can filter by
location, machine, component, tags, or time range.

{{< tabs >}}
{{% tab name="Python" %}}

```python
async def main():
    viam_client = await connect()
    data_client = viam_client.data_client

    await data_client.add_binary_data_to_dataset_by_ids(
        binary_ids=["binary-data-id-1", "binary-data-id-2"],
        dataset_id="YOUR-DATASET-ID",
    )
    print("Images added to dataset.")

    viam_client.close()
```

You can get binary data IDs by querying for images first using the data client's
`binary_data_by_filter` method, which returns objects that include the binary ID.

{{% /tab %}}
{{% tab name="Go" %}}

```go
err = dataClient.AddBinaryDataToDatasetByIDs(
	ctx,
	[]string{"binary-data-id-1", "binary-data-id-2"},
	"YOUR-DATASET-ID",
)
if err != nil {
	logger.Fatal(err)
}
fmt.Println("Images added to dataset.")
```

{{% /tab %}}
{{< /tabs >}}

### 3. Tag images for classification

Tags are labels that apply to an entire image. Use them when you are building
a classification model -- for example, labeling images as "good-part" or
"defective-part".

**Web UI:**

1. In the **DATA** tab, click an image to open it in the detail view.
2. On the right side panel, find the **Tags** section.
3. Click the **+** button next to **Tags**.
4. Type a tag name (e.g., `good-part`) and press Enter.
5. The tag is saved immediately. Repeat for each image.

To tag multiple images at once:

1. Select multiple images using checkboxes.
2. Click **Tag** in the action bar.
3. Enter the tag name and apply it.

{{< tabs >}}
{{% tab name="Python" %}}

```python
from viam.proto.app.data import BinaryID

async def main():
    viam_client = await connect()
    data_client = viam_client.data_client

    binary_ids = [
        BinaryID(
            file_id="file-id-1",
            organization_id=ORG_ID,
            location_id="YOUR-LOCATION-ID",
        ),
        BinaryID(
            file_id="file-id-2",
            organization_id=ORG_ID,
            location_id="YOUR-LOCATION-ID",
        ),
    ]

    await data_client.add_tags_to_binary_data_by_ids(
        tags=["good-part"],
        binary_ids=binary_ids,
    )
    print("Tags added.")

    viam_client.close()
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
binaryIDs := []string{"binary-data-id-1", "binary-data-id-2"}

err = dataClient.AddTagsToBinaryDataByIDs(
	ctx,
	[]string{"good-part"},
	binaryIDs,
)
if err != nil {
	logger.Fatal(err)
}
fmt.Println("Tags added.")
```

{{% /tab %}}
{{< /tabs >}}

Choose tag names that are clear, consistent, and descriptive. Use lowercase with
hyphens (e.g., `good-part`, `defective-part`, `no-part`). Avoid vague names like
`type1` or `other`.

### 4. Draw bounding boxes for object detection

Bounding boxes mark the location of specific objects within an image. Use them
when you are building an object detection model -- for example, detecting
packages on a conveyor belt.

**Web UI:**

1. In the **DATA** tab, click an image to open it in the detail view.
2. Click the **Annotate** button in the image toolbar.
3. In the annotation panel, select or create a label (e.g., `package`).
4. Hold **Cmd** (macOS) or **Ctrl** (Windows/Linux) and click-and-drag on the
   image to draw a rectangle around the object.
5. The bounding box appears with your selected label. Adjust the box edges by
   dragging them if needed.
6. Repeat for every object in the image that should be detected.
7. Move to the next image and repeat.

Draw tight bounding boxes that closely fit the object. Do not include excessive
background. If an image contains multiple objects, draw a separate bounding box
for each one.

{{< tabs >}}
{{% tab name="Python" %}}

```python
async def main():
    viam_client = await connect()
    data_client = viam_client.data_client

    bbox_id = await data_client.add_bounding_box_to_image_by_id(
        binary_id=BinaryID(
            file_id="file-id-1",
            organization_id=ORG_ID,
            location_id="YOUR-LOCATION-ID",
        ),
        label="package",
        x_min_normalized=0.15,
        y_min_normalized=0.20,
        x_max_normalized=0.85,
        y_max_normalized=0.90,
    )
    print(f"Added bounding box: {bbox_id}")

    viam_client.close()
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
bboxID, err := dataClient.AddBoundingBoxToImageByID(
	ctx,
	"binary-data-id-1",
	"package",
	0.15, // x_min_normalized
	0.20, // y_min_normalized
	0.85, // x_max_normalized
	0.90, // y_max_normalized
)
if err != nil {
	logger.Fatal(err)
}
fmt.Printf("Added bounding box: %s\n", bboxID)
```

{{% /tab %}}
{{< /tabs >}}

Bounding box coordinates are normalized to the image dimensions, meaning they
range from 0.0 to 1.0. The point (0.0, 0.0) is the top-left corner and
(1.0, 1.0) is the bottom-right corner. This means a box covering the center
half of the image would use `x_min=0.25, y_min=0.25, x_max=0.75, y_max=0.75`.

### 5. Verify dataset quality

Before you train a model, check that your dataset meets the requirements.

**In the web UI:**

1. Go to the **DATA** tab and click the **DATASETS** subtab.
2. Click your dataset to open it.
3. Review the dataset summary, which shows:
   - Total number of images
   - Number of labeled images
   - Labels used and their counts
4. Check each requirement:

| Check | What to look for |
|-------|-----------------|
| Enough images | At least 15 total. More is better. |
| Labeling coverage | At least 80% of images have tags or bounding boxes |
| Examples per label | At least 10 images per label |
| Label balance | No label should have more than 3x the images of any other label |
| Production conditions | Images should represent real operating conditions, not staged or ideal setups |

**Common issues to fix before training:**

- **Too few images in one class:** Capture more images of the underrepresented
  class, or remove the class and merge it with a related one.
- **Unlabeled images:** Either label them or remove them from the dataset.
  Unlabeled images do not help training and can confuse the summary statistics.
- **Non-representative images:** If your model will run on a factory floor but
  your training images were taken on a clean desk, the model will not generalize.
  Capture images under production conditions -- with the actual lighting,
  background, camera angle, and distance your machine uses.

{{< tabs >}}
{{% tab name="Python" %}}

```python
async def main():
    viam_client = await connect()
    data_client = viam_client.data_client

    datasets = await data_client.list_datasets_by_organization_id(
        organization_id=ORG_ID,
    )
    for ds in datasets:
        print(f"Dataset: {ds.name}, ID: {ds.id}")

    viam_client.close()
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
datasets, err := dataClient.ListDatasetsByOrganizationID(ctx, orgID)
if err != nil {
	logger.Fatal(err)
}
for _, ds := range datasets {
	fmt.Printf("Dataset: %s, ID: %s\n", ds.Name, ds.ID)
}
```

{{% /tab %}}
{{< /tabs >}}

## Try It

1. Create a dataset using any of the three methods (web UI, CLI, or SDK).
2. Add at least 15 images to the dataset.
3. Label every image with at least one tag or bounding box.
4. Open the dataset in the web UI and verify:
   - The image count is 15 or more.
   - Each label has at least 10 examples.
   - Labels are roughly balanced.
5. If you used tags, confirm each image has exactly one tag (for single-label
   classification) or one or more tags (for multi-label classification).
6. If you used bounding boxes, confirm each object in each image has a box
   drawn tightly around it.

## Troubleshooting

{{< expand "Dataset creation fails" >}}

- **Name already exists.** Dataset names must be unique within your organization.
  Choose a different name or delete the existing dataset if it is no longer
  needed.
- **Permission denied.** Verify that your API key has organization-level access.
  API keys scoped to a single machine or location cannot create datasets.

{{< /expand >}}

{{< expand "Images not appearing in the dataset" >}}

- **Sync delay.** Images must sync from the machine to the cloud before they
  are available to add to a dataset. Wait a minute and check the **DATA** tab
  to confirm images have arrived.
- **Wrong filter.** If using the CLI `add filter` command, double-check your
  filter criteria. A typo in a tag name or location ID will match zero images
  without an error.
- **Binary ID mismatch.** If adding images programmatically by ID, verify that
  the binary IDs are correct. You can retrieve valid IDs using the
  `binary_data_by_filter` method.

{{< /expand >}}

{{< expand "Bounding boxes not saving" >}}

- **Annotation mode not active.** You must click the **Annotate** button before
  drawing boxes. Without annotation mode, click-and-drag does nothing.
- **No label selected.** You must select or create a label before drawing a
  bounding box. If no label is selected, the box will not persist.
- **Box too small.** Very small bounding boxes (a few pixels) may not register.
  Draw boxes that clearly encompass the object.

{{< /expand >}}

{{< expand "Label imbalance warnings" >}}

- **Collect more data for underrepresented labels.** The most effective fix is
  to capture more images of the minority class under varied conditions.
- **Do not duplicate images.** Adding copies of the same image inflates the
  count without adding information. The model needs diverse examples.
- **Consider merging labels.** If two labels are very similar and one has few
  examples, merge them into a single label.

{{< /expand >}}

{{< expand "Tags applied to wrong images" >}}

- **Remove incorrect tags.** In the web UI, click the image, find the tag in
  the Tags section, and click the **x** next to it to remove it.
- **Programmatically remove tags.** Use `remove_tags_from_binary_data_by_ids`
  in the SDK to remove tags from specific images.

{{< /expand >}}

## What's Next

- [Train a Model](/train/train-a-model/) -- use your labeled dataset to
  train a classification or object detection model.
- [Add Computer Vision](/vision-detection/add-computer-vision/) -- deploy
  a trained model to your machine's camera feed.
- [Query Data](/data/query-data/) -- write queries to explore and analyze
  your captured data before building datasets.
