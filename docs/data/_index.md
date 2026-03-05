---
linkTitle: "Data"
title: "Data"
weight: 40
layout: "docs"
type: "docs"
no_list: true
description: "Capture, query, filter, pipeline, export, and visualize data."
aliases:
  - /build/data/
  - /services/data/capture/
  - /data/capture/
  - /build/micro-rdk/data_management/
  - /services/data/cloud-sync/
  - /data/cloud-sync/
  - /services/data/capture-sync/
  - /how-tos/sensor-data/
  - /services/data/
  - fleet/data-management/
  - /manage/data-management/
  - /services/data-management/
  - /manage/data/
  - /data-management/
  - /manage/data/export/
  - /data/export/
  - /services/data/export/
  - /manage/data/view/
  - /data/view/
  - /services/data/view/
  - /how-tos/collect-data/
  - /how-tos/collect-sensor-data/
  - /get-started/quickstarts/collect-data/
  - /use-cases/collect-sensor-data/
  - /use-cases/image-data/
  - /get-started/collect-data/
  - /fleet/data-management/
  - /data-ai/capture-data/capture-sync/
  - /data/data-capture-reference/
---

Viam's data management service captures data from your machines and syncs it to the cloud.
From there you can query, filter, build pipelines, sync to your own database, or visualize your data.

The data management service captures data from configured resources and syncs it to the cloud:

- Captured data is written to local storage (<file>~/.viam/capture</file> by default).
- Data syncs to the Viam cloud at a configured interval using encrypted gRPC calls, then is deleted from the device.
- Capture and sync run independently — one can operate without the other.

For details on sync internals (retry behavior, platform-specific paths, deletion criteria), see [How sync works](/data-ai/capture-data/advanced/how-sync-works/).

<img src="/data-manager.svg" alt="Diagram showing data being captured, synced, and removed." class="aligncenter overview imgzoom" style="width:800px;height:auto" >

## Supported resources

The following components and services support data capture and cloud sync:

{{< readfile "/static/include/data/capture-supported.md" >}}

{{< alert title="Support notice" color="note" >}}
Some models do not support all methods. For example, webcams do not capture point clouds.
{{< /alert >}}

## Micro-RDK

The micro-RDK (for ESP32 and similar microcontrollers) supports data capture with a limited set of resources. See the **Micro-RDK** tab in the supported resources table above for details.

Note that on micro-RDK devices, captured data is stored in flash memory and may be lost if the device restarts before syncing. See [How sync works](/data-ai/capture-data/advanced/how-sync-works/) for more information.

## Advanced configuration

For JSON-level configuration options including retention policies, sync optimization, direct MongoDB capture, and remote parts capture, see [Advanced data capture and sync configurations](/data-ai/capture-data/advanced/advanced-data-capture-sync/).

## Related pages

- [Upload data from other sources](/data-ai/capture-data/upload-other-data/) — sync from arbitrary directories or upload via SDK
- [Conditional sync](/data-ai/capture-data/conditional-sync/) — sync only during specific time windows
- [Filter before sync](/data-ai/capture-data/filter-before-sync/) — use ML to filter captured images before syncing
- [Filter at the Edge](/data/filter-at-the-edge/) — reduce data volume with frequency, threshold, and custom filters
