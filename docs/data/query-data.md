---
linkTitle: "Query Data"
title: "Query Data"
weight: 10
layout: "docs"
type: "docs"
description: "Write SQL and MQL queries against captured data in the Viam app or programmatically."
date: "2025-01-30"
aliases:
  - /build/data/query-data/
---

## What Problem This Solves

Query captured tabular data — sensor readings, motor positions, encoder ticks,
and other structured key-value data — using SQL or MQL in the Viam app's query
editor or programmatically through the SDK.

Binary data such as images and point clouds is stored separately and is not
queryable via SQL/MQL. To access binary data programmatically, use the
[data client API](/reference/apis/services/data/).

## Concepts

### The readings table schema

All tabular data captured by Viam is stored in a single table called `readings`
in the `sensorData` database. Each row represents one capture event from one
component. The columns are:

| Column | Type | Description |
|--------|------|-------------|
| `organization_id` | String | Organization UUID |
| `location_id` | String | Location UUID |
| `robot_id` | String | Machine UUID |
| `part_id` | String | Machine part UUID |
| `component_type` | String | Resource type (e.g., `rdk:component:sensor`) |
| `component_name` | String | Resource name (e.g., `my-sensor`) |
| `method_name` | String | Capture method (e.g., `Readings`, `EndPosition`, `GetImages`) |
| `time_requested` | Timestamp | When the capture was requested on the machine |
| `time_received` | Timestamp | When the cloud received the data |
| `tags` | Array | User-applied tags |
| `additional_parameters` | JSON | Method-specific parameters (e.g., `pin_name`, `reader_name`) |
| `data` | JSON | The captured reading -- nested structure varies by component type |

The `data` column is where the actual reading lives. Its structure depends on
what was captured. A sensor reading might look like:

```json
{
  "readings": {
    "temperature": 23.5,
    "humidity": 61.2
  }
}
```

A vision service detection result might look like:

```json
{
  "detections": [
    {
      "class_name": "person",
      "confidence": 0.94,
      "x_min": 120, "y_min": 50,
      "x_max": 340, "y_max": 480
    }
  ]
}
```

Understanding this nested structure is key to writing effective queries.

### SQL vs. MQL

Viam's query editor supports two languages:

- **SQL** is good for straightforward filtering, sorting, and limiting. If you
  want "show me the last N readings" or "find entries where component name is X",
  SQL is direct and familiar.
- **MQL** (MongoDB Query Language) uses aggregation pipelines -- a sequence of
  stages that transform documents. MQL is more powerful when you need to group,
  count, compute averages, or restructure nested data. Each stage in the
  pipeline feeds its output to the next stage.

Both are available in the Viam app query editor and through the programmatic
API.

## Open the query editor

1. Go to [app.viam.com](https://app.viam.com).
2. Click the **DATA** tab in the top navigation.
3. Click **Query** to open the query editor.
4. Select **SQL** or **MQL** mode depending on which language you want to use.

## Explore your data with basic SQL

Start with a broad query to see what data you have:

```sql
SELECT time_received, component_name, component_type, data
FROM readings
ORDER BY time_received DESC
LIMIT 10
```

This shows the 10 most recent readings across all components. Look at the
`data` column to understand the JSON structure of your specific readings.

To narrow to a specific component:

```sql
SELECT time_received, data
FROM readings
WHERE component_name = 'my-sensor'
ORDER BY time_received DESC
LIMIT 10
```

To filter by time range:

```sql
SELECT time_received, component_name, data
FROM readings
WHERE time_received > '2025-01-15T00:00:00Z'
  AND time_received < '2025-01-16T00:00:00Z'
ORDER BY time_received ASC
```

## Extract fields from nested JSON

The `data` column contains JSON, so you need JSON functions to extract specific
values. Use dot notation to reach into the nested structure:

```sql
SELECT
  time_received,
  data.readings.temperature AS temperature,
  data.readings.humidity AS humidity
FROM readings
WHERE component_name = 'my-sensor'
ORDER BY time_received DESC
LIMIT 20
```

For detection results from a vision service:

```sql
SELECT
  time_received,
  data.detections
FROM readings
WHERE component_name = 'my-detector'
  AND component_type = 'rdk:service:vision'
ORDER BY time_received DESC
LIMIT 10
```

To filter by a specific machine:

```sql
SELECT time_received, component_name, data
FROM readings
WHERE robot_id = 'YOUR-MACHINE-ID'
ORDER BY time_received DESC
LIMIT 10
```

## Write MQL aggregation pipelines

Switch to **MQL** mode in the query editor. MQL queries are JSON arrays where
each element is a pipeline stage.

Get the last 10 readings from a component:

```json
[
  {"$match": {"component_name": "my-sensor"}},
  {"$sort": {"time_received": -1}},
  {"$limit": 10},
  {"$project": {
    "time_received": 1,
    "data": 1,
    "_id": 0
  }}
]
```

Count readings per component:

```json
[
  {"$group": {
    "_id": "$component_name",
    "count": {"$sum": 1}
  }},
  {"$sort": {"count": -1}}
]
```

Count readings per component over a specific time window:

```json
[
  {"$match": {
    "time_received": {
      "$gte": {"$date": "2025-01-15T00:00:00Z"},
      "$lt": {"$date": "2025-01-16T00:00:00Z"}
    }
  }},
  {"$group": {
    "_id": "$component_name",
    "count": {"$sum": 1}
  }},
  {"$sort": {"count": -1}}
]
```

Compute average, min, and max of a sensor value:

```json
[
  {"$match": {"component_name": "my-sensor"}},
  {"$group": {
    "_id": null,
    "avg_temperature": {"$avg": "$data.readings.temperature"},
    "min_temperature": {"$min": "$data.readings.temperature"},
    "max_temperature": {"$max": "$data.readings.temperature"},
    "total_readings": {"$sum": 1}
  }}
]
```

Group readings by hour to see trends over time:

```json
[
  {"$match": {"component_name": "my-sensor"}},
  {"$group": {
    "_id": {
      "$dateToString": {
        "format": "%Y-%m-%d %H:00",
        "date": "$time_received"
      }
    },
    "avg_temperature": {"$avg": "$data.readings.temperature"},
    "count": {"$sum": 1}
  }},
  {"$sort": {"_id": 1}}
]
```

Find all detections above a confidence threshold:

```json
[
  {"$match": {"component_name": "my-detector"}},
  {"$unwind": "$data.detections"},
  {"$match": {"data.detections.confidence": {"$gte": 0.9}}},
  {"$project": {
    "time_received": 1,
    "class_name": "$data.detections.class_name",
    "confidence": "$data.detections.confidence",
    "_id": 0
  }},
  {"$sort": {"time_received": -1}},
  {"$limit": 20}
]
```

The `$unwind` stage is important when your data contains arrays. It flattens
the array so each element becomes its own document, which you can then filter
and project individually.

## Query data programmatically

You can run the same SQL and MQL queries from your own code using the Viam
data client API. This is useful for building dashboards, generating reports,
or feeding data into analysis pipelines.

{{< tabs >}}
{{% tab name="Python" %}}

Install the SDK if you haven't already:

```bash
pip install viam-sdk
```

Save this as `query_data.py`:

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

    # SQL: recent readings from a specific component
    print("=== Recent sensor readings (SQL) ===")
    sql_results = await data_client.tabular_data_by_sql(
        organization_id=ORG_ID,
        sql_query=(
            "SELECT time_received, "
            "  data.readings.temperature AS temperature, "
            "  data.readings.humidity AS humidity "
            "FROM readings "
            "WHERE component_name = 'my-sensor' "
            "ORDER BY time_received DESC "
            "LIMIT 5"
        ),
    )
    for row in sql_results:
        print(f"  {row}")

    # MQL: count readings per component
    print("\n=== Readings per component (MQL) ===")
    mql_results = await data_client.tabular_data_by_mql(
        organization_id=ORG_ID,
        mql_binary=[
            {"$group": {
                "_id": "$component_name",
                "count": {"$sum": 1},
            }},
            {"$sort": {"count": -1}},
        ],
    )
    for entry in mql_results:
        print(f"  {entry}")

    viam_client.close()


if __name__ == "__main__":
    asyncio.run(main())
```

Replace `YOUR-API-KEY`, `YOUR-API-KEY-ID`, and `YOUR-ORGANIZATION-ID` with your
values. Find your organization ID in the Viam app under **Settings** in the left
navigation.

Run it:

```bash
python query_data.py
```

{{% /tab %}}
{{% tab name="Go" %}}

Initialize a Go module and install the SDK:

```bash
mkdir query-data && cd query-data
go mod init query-data
go get go.viam.com/rdk
```

Save this as `main.go`:

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
	logger := logging.NewDebugLogger("query-data")

	viamClient, err := app.CreateViamClientWithAPIKey(
		ctx, app.Options{}, apiKey, apiKeyID, logger)
	if err != nil {
		logger.Fatal(err)
	}
	defer viamClient.Close()

	dataClient := viamClient.DataClient()

	// SQL: recent readings from a specific component
	fmt.Println("=== Recent sensor readings (SQL) ===")
	sqlResults, err := dataClient.TabularDataBySQL(ctx, orgID,
		"SELECT time_received, "+
			"data.readings.temperature AS temperature, "+
			"data.readings.humidity AS humidity "+
			"FROM readings "+
			"WHERE component_name = 'my-sensor' "+
			"ORDER BY time_received DESC LIMIT 5")
	if err != nil {
		logger.Fatal(err)
	}
	for _, row := range sqlResults {
		fmt.Printf("  %v\n", row)
	}

	// MQL: count readings per component
	fmt.Println("\n=== Readings per component (MQL) ===")
	countStages := []map[string]interface{}{
		{"$group": map[string]interface{}{
			"_id":   "$component_name",
			"count": map[string]interface{}{"$sum": 1},
		}},
		{"$sort": map[string]interface{}{"count": -1}},
	}
	countResults, err := dataClient.TabularDataByMQL(ctx, orgID, countStages, nil)
	if err != nil {
		logger.Fatal(err)
	}
	for _, entry := range countResults {
		fmt.Printf("  %v\n", entry)
	}
}
```

Replace the placeholder values with your API key, API key ID, and organization
ID. Then run:

```bash
go run main.go
```

{{% /tab %}}
{{< /tabs >}}

If you need to export data for use outside of Viam -- in a Jupyter notebook, your
own database, or a BI tool -- see
[Sync Data to Your Database](/data/sync-data-to-your-database/).

## Try It

Run these queries in the Viam app query editor to verify you can work with your
data:

1. **List your components.** Run this MQL query to see what components have
   captured data:
   ```json
   [
     {"$group": {"_id": "$component_name", "count": {"$sum": 1}}},
     {"$sort": {"count": -1}}
   ]
   ```
   You should see each component name and how many readings it has.

2. **Inspect the data structure.** Pick a component from the results above and
   run:
   ```sql
   SELECT data FROM readings
   WHERE component_name = 'YOUR-COMPONENT-NAME'
   LIMIT 1
   ```
   Examine the JSON structure in the `data` column. Note the field names -- you
   will use them to extract specific values.

3. **Extract a specific field.** Using the field names you found, write a query
   that pulls out a specific value. For example, if your sensor has a
   `temperature` field under `readings`:
   ```sql
   SELECT time_received, data.readings.temperature AS temperature
   FROM readings
   WHERE component_name = 'YOUR-COMPONENT-NAME'
   ORDER BY time_received DESC
   LIMIT 10
   ```

4. **Run the Python or Go script.** Copy one of the programmatic query scripts,
   fill in your credentials, and run it. Verify that the output matches what you
   see in the Viam app query editor.

## Troubleshooting

{{< expand "Query returns empty results" >}}

- **Check the component name.** Component names are case-sensitive and must match
  exactly. Run `SELECT DISTINCT component_name FROM readings` to see all
  available component names.
- **Check the time range.** If you're filtering by time, make sure your range
  covers a period when data was actually captured. Remove the time filter first
  to confirm data exists, then add it back.
- **Verify data has synced.** Data must sync from the machine to the cloud before
  it is queryable. Check the **DATA** tab to confirm entries are visible.

{{< /expand >}}

{{< expand "Query returns data but fields are null" >}}

- **Check the JSON path.** The nested path must match the actual structure of
  your data. Run `SELECT data FROM readings LIMIT 1` to see the raw JSON, then
  build your dot-notation path to match. A common mistake is using
  `data.temperature` when the actual path is `data.readings.temperature`.

{{< /expand >}}

{{< expand "MQL pipeline returns unexpected results" >}}

- **Build incrementally.** Start with just the `$match` stage and verify it
  returns the documents you expect. Then add one stage at a time. This makes it
  easy to identify which stage is producing unexpected output.
- **Check field references.** In MQL, field references use `$` prefix
  (e.g., `$component_name`, `$data.readings.temperature`). Missing the `$` is
  a common source of errors.

{{< /expand >}}

{{< expand "Query is slow" >}}

- **Add filters early.** Always include a `$match` stage (MQL) or `WHERE`
  clause (SQL) to narrow the data before doing expensive operations like
  grouping or sorting. Filtering by `component_name` and `time_received` is
  particularly effective.
- **Use LIMIT.** While developing queries, always include a `LIMIT` clause (SQL)
  or `$limit` stage (MQL) to avoid scanning your entire dataset.

{{< /expand >}}

{{< expand "Programmatic query fails with authentication error" >}}

- **Verify your API key and API key ID.** Both values are required and they
  serve different purposes. The API key is the secret; the API key ID identifies
  which key is being used.
- **Verify your organization ID.** Find it in the Viam app under **Settings** in
  the left navigation. It is a UUID, not your organization name.
- **Check that the API key has data access.** API keys can be scoped to specific
  resources. Ensure yours has access to the data service.

{{< /expand >}}

## What's Next

- [Create a Dataset](/train/create-a-dataset/) -- organize queried data
  into datasets for ML training.
- [Add Computer Vision](/vision-detection/add-computer-vision/) -- capture
  detection results that you can query with the patterns above.
- [Trigger on Detection](/build/stationary-vision/trigger-on-detection/) -- use
  data patterns to trigger automated actions.
