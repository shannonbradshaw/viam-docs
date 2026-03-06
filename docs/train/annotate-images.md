---
linkTitle: "Annotate Images"
title: "Annotate Images"
weight: 15
layout: "docs"
type: "docs"
description: "Label images with tags or bounding boxes for training an ML model."
date: "2025-01-30"
aliases:
  - /data-ai/train/annotate-images/
  - /data-ai/train/capture-annotate-images/
---

Before you can train an ML model, you need to label the images in your dataset.
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

## Prerequisites

You need a [dataset with images](/train/create-a-dataset/) before you can
annotate.

## Tag images for classification

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

## Draw bounding boxes for object detection

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

## Troubleshooting

{{< expand "Bounding boxes not saving" >}}

- **Annotation mode not active.** You must click the **Annotate** button before
  drawing boxes. Without annotation mode, click-and-drag does nothing.
- **No label selected.** You must select or create a label before drawing a
  bounding box. If no label is selected, the box will not persist.
- **Box too small.** Very small bounding boxes (a few pixels) may not register.
  Draw boxes that clearly encompass the object.

{{< /expand >}}

{{< expand "Tags applied to wrong images" >}}

- **Remove incorrect tags.** In the web UI, click the image, find the tag in
  the Tags section, and click the **x** next to it to remove it.
- **Programmatically remove tags.** Use `remove_tags_from_binary_data_by_ids`
  in the SDK to remove tags from specific images.

{{< /expand >}}

## What's Next

- [Automate Annotation](/train/automate-annotation/) -- speed up labeling by
  using an existing ML model to generate predictions automatically.
- [Train a Model](/train/train-a-model/) -- use your labeled dataset to
  train a classification or object detection model.
