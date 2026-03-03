---
linkTitle: "Visualize Data"
title: "Visualize Data"
weight: 50
layout: "docs"
type: "docs"
description: "Build dashboards with Teleop, Grafana, or programmatic charts to visualize captured data."
date: "2025-01-30"
---

## What Problem This Solves

Captured data sitting in the cloud is useful for queries, but humans understand
trends, anomalies, and system health much faster when they can see them. A
temperature reading of 47.3 degrees C is just a number. A line chart showing
temperature climbing steadily over the last six hours tells you something is
wrong before a threshold is crossed.

This how-to covers four approaches to visualization, from simplest to most
flexible.

## Concepts

### Viam's visualization options

Viam stores captured data in a MongoDB Atlas Data Federation instance. This
means any tool that can connect to MongoDB can access your data. You have four
options:

| Approach | Best for | Setup time |
|----------|----------|------------|
| Teleop dashboard | Quick monitoring, camera feeds, live sensor data | 2 minutes |
| Grafana | Rich dashboards, alerting, team-shared views | 10 minutes |
| Other third-party tools (Tableau, Looker Studio) | BI reporting, executive dashboards | 10-15 minutes |
| Programmatic (Python/Go + matplotlib or similar) | Custom reports, embedded charts, notebooks | 5 minutes |

### Teleop dashboards vs. external tools

Teleop dashboards live inside the Viam app. They are scoped to a single machine
and location, and they update in real time. You do not need to configure any
database credentials or install any software. This makes them the right choice
when you want a quick operational view of one machine.

External tools like Grafana connect to the underlying MongoDB data store. They
can query data across machines and organizations, apply complex transformations,
set up alerts, and share dashboards with team members who do not have Viam app
access.

Programmatic visualization gives you complete control. You write code that
queries data through the Viam SDK and renders it however you want -- matplotlib
charts, Plotly dashboards, HTML reports, or any other format.

## Steps

### 1. Build a Teleop dashboard

The Teleop dashboard is the fastest way to visualize data from a single machine.

1. Go to [app.viam.com](https://app.viam.com).
2. Click **FLEET** in the top navigation, then click the **TELEOP** tab. You
   can also navigate directly to [app.viam.com/teleop](https://app.viam.com/teleop).
3. Click **+ Create workspace**.
4. In the top left of the page, replace the placeholder `untitled-workspace`
   text with a descriptive name for your workspace (e.g., "Sensor Monitor" or
   "Factory Floor Cameras").
5. Use the **Select location** dropdown to choose the location that contains
   your machine.
6. Use the **Select machine** dropdown to choose the machine you want to
   visualize data from.

#### Add widgets

1. Click **Add widget** and select a widget type. Available widget types include:
   - **Time-series chart** -- plots sensor values over time.
   - **Camera feed** -- shows a live camera stream.
   - **Value display** -- shows the current reading from a sensor.
   - **Button** -- triggers an action on the machine.
2. To configure a widget, click the **pencil icon** in the top right corner of
   the widget. This opens the widget settings where you select which component
   and data source the widget displays.
3. Repeat for each widget you want on the dashboard. You can mix camera feeds,
   time-series charts, and value displays on the same workspace.

#### Arrange the layout

- Click and drag the **grid icon** in the top left corner of any widget to move
  it to a new position on the workspace.
- Resize widgets by dragging their edges.
- The layout saves automatically.

You now have a live dashboard showing real-time data from your machine. The
dashboard updates as new data arrives -- no refresh needed.

### 2. Configure database access for third-party tools

To connect external visualization tools (Grafana, Tableau, Looker Studio, or
any MongoDB-compatible client), you first need to set up database credentials
using the Viam CLI.

1. Install the Viam CLI if you have not already:

   ```bash
   brew tap viamrobotics/brews
   brew install viam
   ```

   On Linux, download the binary from [the CLI documentation](https://docs.viam.com/dev/tools/cli/).

2. Log in:

   ```bash
   viam login
   ```

3. Find your organization ID:

   ```bash
   viam organizations list
   ```

4. Configure database credentials:

   ```bash
   viam data database configure --org-id=<YOUR-ORG-ID> --password=<NEW-PASSWORD>
   ```

5. Get the database hostname:

   ```bash
   viam data database hostname --org-id=<YOUR-ORG-ID>
   ```

You now have three pieces of information needed by any external tool:

- **Hostname** (from the `hostname` command)
- **Username**: `db-user-<YOUR-ORG-ID>`
- **Password** (from the `configure` command)

### 3. Connect Grafana to your data

Grafana is a popular open-source visualization tool that works well with Viam's
MongoDB-backed data store.

#### Install Grafana

If you do not already have a Grafana instance:

- **Grafana Cloud**: Sign up at [grafana.com](https://grafana.com). The free
  tier is sufficient for testing.
- **Local Grafana Enterprise**: Install following the [official Grafana
  installation guide](https://grafana.com/docs/grafana/latest/setup-grafana/installation/).

#### Install the MongoDB data source plugin

1. Open your Grafana web UI.
2. Navigate to **Connections > Add new connection**.
3. Search for **MongoDB**.
4. Install the **Grafana MongoDB data source** plugin.

{{< alert title="Important" color="caution" >}}

Install the MongoDB **data source**, not the MongoDB **integration**. They are
different plugins. The data source is what allows you to query MongoDB from
Grafana dashboards.

{{< /alert >}}

#### Configure the data connection

1. After installing the plugin, go to its configuration page and click
   **Add new data source**.
2. Fill in the following fields:

   - **Connection string**:

     ```
     mongodb://<HOSTNAME>/sensorData?directConnection=true&authSource=admin&tls=true
     ```

     Replace `<HOSTNAME>` with the hostname from step 2.

   - **User**: `db-user-<YOUR-ORG-ID>`

   - **Password**: The password you set with the
     `viam data database configure` command.

3. Click **Save & test**. Grafana should confirm the connection is working.

#### Build a Grafana dashboard

1. Click **Dashboards** in the left sidebar, then **New > New dashboard**.
2. Click **Add visualization**.
3. Select the MongoDB data source you just configured.
4. In the query editor, enter an MQL query. For example, to plot sensor readings
   over time:

   ```mql
   sensorData.readings.aggregate([
     {
       $match: {
         component_name: "sensor-1",
         time_received: { $gte: ISODate(${__from}) }
       }
     },
     { $sort: { time_received: 1 } },
     { $limit: 1000 }
   ])
   ```

   Replace `sensor-1` with the name of your component.

5. Click **Run query** to see results.
6. Choose a visualization type (Time series, Gauge, Stat, Table, etc.) from the
   panel options on the right.
7. Click **Apply** to add the panel to your dashboard.

#### Useful query patterns for Grafana

Average sensor value per hour:

```mql
sensorData.readings.aggregate([
  {
    $match: {
      component_name: "my-sensor",
      time_received: { $gte: ISODate(${__from}), $lte: ISODate(${__to}) }
    }
  },
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-%m-%d %H:00", date: "$time_received" }
      },
      avg_value: { $avg: "$data.readings.temperature" },
      count: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
])
```

Count of readings per component:

```mql
sensorData.readings.aggregate([
  {
    $match: {
      time_received: { $gte: ISODate(${__from}) }
    }
  },
  {
    $group: {
      _id: "$component_name",
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } }
])
```

### 4. Connect other third-party tools

The database credentials from step 2 work with any tool that supports MongoDB
Atlas Data Federation as a data store. The general pattern is the same:

1. Install the tool's MongoDB connector or driver. For example:
   - **Tableau** requires the [Atlas SQL JDBC Driver](https://www.mongodb.com/try/download/jdbc-driver) and the [Tableau Connector](https://www.mongodb.com/try/download/tableau-connector).
   - **Google Looker Studio** supports MongoDB connections natively through community connectors.

2. Configure the connection using one of these formats:

   **Connection URI format** (most tools):

   ```
   mongodb://db-user-<YOUR-ORG-ID>:<YOUR-PASSWORD>@<HOSTNAME>/sensorData?ssl=true&authSource=admin
   ```

   **Hostname + credentials format** (tools that ask for fields separately):

   - Hostname: the value from `viam data database hostname`
   - Database name: `sensorData`
   - Username: `db-user-<YOUR-ORG-ID>`
   - Password: from the `configure` command

3. Build dashboards using the tool's native interface.

{{< alert title="Tip" color="tip" >}}

Before building dashboards, use [MongoDB Compass](https://www.mongodb.com/products/tools/compass)
to browse your data and explore its structure. Compass's Schema Analyzer shows
you the fields, types, and value distributions in your data, which helps you
write accurate queries in your visualization tool.

{{< /alert >}}

### 5. Build a programmatic dashboard

When you need full control over visualization -- embedding charts in your own
application, generating reports, or running in a Jupyter notebook -- use the
Viam SDK to query data and a plotting library to render it.

{{< tabs >}}
{{% tab name="Python" %}}

Install the dependencies:

```bash
pip install viam-sdk matplotlib
```

Save this as `visualize_data.py`:

```python
import asyncio
from datetime import datetime, timedelta, timezone

import matplotlib.pyplot as plt
import matplotlib.dates as mdates
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

    # Query the last 24 hours of sensor data.
    data = await data_client.tabular_data_by_mql(
        organization_id=ORG_ID,
        mql_binary=[
            {"$match": {
                "component_name": "my-sensor",
                "time_received": {
                    "$gte": {"$date": (
                        datetime.now(timezone.utc) - timedelta(hours=24)
                    ).isoformat()}
                },
            }},
            {"$sort": {"time_received": 1}},
            {"$limit": 1000},
            {"$project": {
                "time_received": 1,
                "temperature": "$data.readings.temperature",
                "_id": 0,
            }},
        ],
    )

    if not data:
        print("No data returned. Check your component name and time range.")
        viam_client.close()
        return

    # Extract timestamps and values.
    timestamps = [entry["time_received"] for entry in data]
    temperatures = [entry["temperature"] for entry in data]

    # Plot the data.
    fig, ax = plt.subplots(figsize=(12, 5))
    ax.plot(timestamps, temperatures, linewidth=1.5, color="#3b82f6")
    ax.set_xlabel("Time")
    ax.set_ylabel("Temperature")
    ax.set_title("Sensor Temperature - Last 24 Hours")
    ax.xaxis.set_major_formatter(mdates.DateFormatter("%H:%M"))
    ax.grid(True, alpha=0.3)
    fig.autofmt_xdate()
    plt.tight_layout()

    # Save to file and show.
    plt.savefig("temperature_chart.png", dpi=150)
    print("Chart saved to temperature_chart.png")
    plt.show()

    viam_client.close()


if __name__ == "__main__":
    asyncio.run(main())
```

Replace `YOUR-API-KEY`, `YOUR-API-KEY-ID`, `YOUR-ORGANIZATION-ID`, and
`my-sensor` with your values. Then run:

```bash
python visualize_data.py
```

This produces a time-series chart of temperature readings and saves it as a PNG
file.

{{% /tab %}}
{{% tab name="Go" %}}

Initialize a Go module and install dependencies:

```bash
mkdir visualize-data && cd visualize-data
go mod init visualize-data
go get go.viam.com/rdk
go get gonum.org/v1/plot
```

Save this as `main.go`:

```go
package main

import (
	"context"
	"fmt"
	"time"

	"go.viam.com/rdk/app"
	"go.viam.com/rdk/logging"
	"gonum.org/v1/plot"
	"gonum.org/v1/plot/plotter"
	"gonum.org/v1/plot/vg"
)

func main() {
	apiKey := "YOUR-API-KEY"
	apiKeyID := "YOUR-API-KEY-ID"
	orgID := "YOUR-ORGANIZATION-ID"

	ctx := context.Background()
	logger := logging.NewDebugLogger("visualize-data")

	viamClient, err := app.CreateViamClientWithAPIKey(
		ctx, app.Options{}, apiKey, apiKeyID, logger)
	if err != nil {
		logger.Fatal(err)
	}
	defer viamClient.Close()

	dataClient := viamClient.DataClient()

	// Query the last 24 hours of sensor data.
	cutoff := time.Now().UTC().Add(-24 * time.Hour)
	mqlStages := []map[string]interface{}{
		{"$match": map[string]interface{}{
			"component_name": "my-sensor",
			"time_received": map[string]interface{}{
				"$gte": map[string]interface{}{
					"$date": cutoff.Format(time.RFC3339),
				},
			},
		}},
		{"$sort": map[string]interface{}{"time_received": 1}},
		{"$limit": 1000},
		{"$project": map[string]interface{}{
			"time_received": 1,
			"temperature":   "$data.readings.temperature",
			"_id":           0,
		}},
	}

	results, err := dataClient.TabularDataByMQL(ctx, orgID, mqlStages, nil)
	if err != nil {
		logger.Fatal(err)
	}

	if len(results) == 0 {
		fmt.Println("No data returned. Check your component name and time range.")
		return
	}

	// Build plot data points.
	pts := make(plotter.XYs, len(results))
	for i, entry := range results {
		// Extract timestamp as float (seconds since epoch).
		if t, ok := entry["time_received"].(time.Time); ok {
			pts[i].X = float64(t.Unix())
		}
		// Extract temperature value.
		if temp, ok := entry["temperature"].(float64); ok {
			pts[i].Y = temp
		}
	}

	// Create the plot.
	p := plot.New()
	p.Title.Text = "Sensor Temperature - Last 24 Hours"
	p.X.Label.Text = "Time"
	p.Y.Label.Text = "Temperature"

	line, err := plotter.NewLine(pts)
	if err != nil {
		logger.Fatal(err)
	}
	p.Add(line)

	// Save to file.
	if err := p.Save(10*vg.Inch, 4*vg.Inch, "temperature_chart.png"); err != nil {
		logger.Fatal(err)
	}
	fmt.Println("Chart saved to temperature_chart.png")
}
```

Replace the placeholder values and component name, then run:

```bash
go run main.go
```

{{% /tab %}}
{{< /tabs >}}

## Try It

1. **Build a Teleop dashboard.** Create a workspace, add a time-series widget
   for a sensor and a camera feed widget, and verify that data appears in real
   time.

2. **Connect Grafana.** Run the CLI commands to configure database access, set
   up the Grafana MongoDB data source, and run the example MQL query. Create a
   time-series panel and verify the chart updates when you change the time range
   selector.

3. **Run the Python or Go script.** Copy one of the programmatic examples, fill
   in your credentials, and run it. You should see a `temperature_chart.png`
   file in your working directory.

## Troubleshooting

{{< expand "Teleop dashboard shows no data" >}}

- **Check that the machine is online.** The Teleop dashboard shows live data
  from the machine. If the machine is offline, widgets will be empty.
- **Verify the component is configured.** Make sure the component you selected
  in the widget configuration exists on the machine and is producing data.
- **Check the widget configuration.** Click the pencil icon on the widget and
  verify that the correct component and data source are selected.

{{< /expand >}}

{{< expand "CLI commands fail with authentication error" >}}

- **Run `viam login` first.** The `database configure` and `database hostname`
  commands require an active login session.
- **Verify your organization ID.** Copy it directly from `viam organizations list`
  output. It is a UUID, not your organization name.

{{< /expand >}}

{{< expand "Grafana connection test fails" >}}

- **Check the connection string format.** The string must include
  `directConnection=true&authSource=admin&tls=true`. Missing any of these
  parameters can cause connection failures.
- **Verify the hostname.** Use the exact hostname returned by
  `viam data database hostname`, not the connection URI.
- **Verify the username format.** The username must be `db-user-<YOUR-ORG-ID>`,
  including the `db-user-` prefix.
- **Verify the password.** This is the password you set with
  `viam data database configure`, not your Viam app password.
- **Check that you installed the data source, not the integration.** The Grafana
  MongoDB integration and the Grafana MongoDB data source are different plugins.
  You need the data source.

{{< /expand >}}

{{< expand "Grafana query returns empty results" >}}

- **Check the component name.** Component names in MQL queries are
  case-sensitive and must match exactly.
- **Check the time range.** The `${__from}` variable is controlled by the time
  range selector at the top of the Grafana dashboard. Make sure the range covers
  a period when data was captured.
- **Verify the database name.** Sensor data is stored in the `sensorData`
  database. If you used a different database name in the connection string, the
  query will find no data.

{{< /expand >}}

{{< expand "Programmatic script fails with authentication error" >}}

- **Verify your API key and API key ID.** Both are required. The API key is the
  secret; the API key ID identifies which key is being used.
- **Verify your organization ID.** Find it in the Viam app under **Settings**
  in the left navigation.
- **Check that the API key has data access.** API keys can be scoped to specific
  resources. Ensure yours has access to the data service.

{{< /expand >}}

{{< expand "Chart is empty or shows no data points" >}}

- **Check the component name in the query.** Replace `my-sensor` with the actual
  name of your component.
- **Check the time range.** The example queries the last 24 hours. If your data
  is older than that, adjust the `timedelta` (Python) or `time.Duration` (Go).
- **Check the field path.** The example uses `data.readings.temperature`. Your
  sensor may use a different field name. Run a query in the Viam app query
  editor first to see the actual structure of your data.

{{< /expand >}}

## What's Next

- [Query Data](/build/data/query-data/) -- learn the full range of SQL and MQL
  queries you can use to extract insights from your data.
- [Configure Data Pipelines](/build/data/configure-data-pipelines/) -- set up
  aggregated views and derived metrics to power your dashboards more efficiently.
- [Filter at the Edge](/build/data/filter-at-the-edge/) -- reduce noise in your
  visualizations by filtering data before it leaves the machine.
