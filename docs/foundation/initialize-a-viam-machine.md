---
linkTitle: "Initialize a Viam Machine"
title: "Initialize a Viam Machine"
weight: 10
layout: "docs"
type: "docs"
description: "Install viam-agent and viam-server on your machine and connect it to the Viam platform."
date: "2025-01-30"
aliases:
  - /build/foundation/initialize-a-viam-machine/
  - /operate/install/
  - /operate/reference/prepare/
  - /operate/install/setup/
  - /installation/viam-server-setup/
  - /how-tos/configure/
  - /installation/prepare/
  - /installation/macos-install/
  - /installation/linux-install/
  - /installation/install/
  - /installation/install/linux-install/
  - /installation/install/macos-install
  - /getting-started/installation/
  - /getting-started/macos-install/
  - /getting-started/linux-install/
  - /installation/
  - /get-started/installation/
  - /operate/get-started/setup/
---

## What Problem This Solves

Before you can configure hardware, capture data, or run any logic on a machine,
the machine needs to connect to the Viam platform. This how-to walks you
through setting up your machine to use Viam.

## Concepts

### viam-server

`viam-server` is the core software that runs on your machine. It:

- **Syncs configuration** from the Viam app, so you can configure your machine
  remotely.
- **Manages components** — controls cameras, motors, sensors, and other
  hardware through a uniform interface.
- **Installs and runs required software** — automatically downloads and manages
  hardware drivers and other software your machine needs based on its
  configuration.
- **Handles networking** — manages gRPC and WebRTC connections between your
  machine, the Viam app, and your code.
- **Serves an API** that lets your code control the machine from anywhere.

`viam-agent` is a service manager that installs and manages `viam-server` —
downloading, starting, monitoring, and automatically updating it.

## Steps

### 1. Create a machine in the Viam app

1. Go to [app.viam.com](https://app.viam.com) and log in (or create an
   account).
2. Create or select an organization, then create or select a location.
3. Click **New machine**.
4. Give your machine a name (e.g., `my-first-machine`). Click **Save**.

The app creates a machine entry and shows a setup page. Don't close this page —
you'll need the install command in the next step.

### 2. Install viam-agent

On the machine's setup page in the Viam app, you'll find credentials for your
machine. You'll need the **API key ID**, **API key**, and **Part ID** from this
page.

{{< tabs >}}
{{% tab name="Linux" %}}

Run the following command, replacing the placeholder values with your machine's
credentials from the setup page:

```bash
sudo /bin/sh -c "VIAM_API_KEY_ID=<YOUR_API_KEY_ID> VIAM_API_KEY=<YOUR_API_KEY> VIAM_PART_ID=<YOUR_PART_ID>; $(curl -fsSL https://storage.googleapis.com/packages.viam.com/apps/viam-agent/install.sh)"
```

This installs `viam-agent` as a systemd service, which automatically installs
and starts `viam-server`.

{{% /tab %}}
{{% tab name="macOS" %}}

Run the following command, replacing the placeholder values with your machine's
credentials from the setup page:

```bash
sudo /bin/sh -c "VIAM_API_KEY_ID=<YOUR_API_KEY_ID> VIAM_API_KEY=<YOUR_API_KEY> VIAM_PART_ID=<YOUR_PART_ID>; $(curl -fsSL https://storage.googleapis.com/packages.viam.com/apps/viam-agent/install.sh)"
```

This installs `viam-agent`, which automatically installs and starts
`viam-server`.

{{% /tab %}}
{{% tab name="Windows" %}}

1. Download the [viam-agent Windows installer](https://storage.googleapis.com/packages.viam.com/apps/viam-agent/viam-agent-windows-installer.exe).
2. Copy your machine cloud credentials from the setup page and save them to
   `\etc\viam.json`.
3. Run the installer.

This installs `viam-agent` as a Windows service, which automatically installs
and starts `viam-server`.

{{% /tab %}}
{{< /tabs >}}

{{< alert title="Tip" color="tip" >}}

Always copy your credentials directly from the Viam app's setup page rather
than manually editing a command.

{{< /alert >}}

### 3. Verify your machine is online

Go back to the Viam app and navigate to your machine's page.

- **Green dot** next to the machine name means it's online and connected.
- **Red or gray dot** means the machine is offline.

If the indicator shows online, `viam-server` is running and connected to the
cloud. You're ready to configure components and connect programmatically.

### 4. Connect programmatically

Every subsequent block builds on programmatic access to your machine. Set this
up now.

#### Get your API credentials

1. In the Viam app, go to your machine's page.
2. Click the **CONNECT** tab.
3. Select the **API keys** section.
4. Copy the **API key** and **API key ID**. You'll also need your **machine
   address** (shown on the same page).

{{< alert title="Important" color="caution" >}}

Keep credentials secure. Don't commit API keys to version control. Use
environment variables or a `.env` file.

{{< /alert >}}

{{< tabs >}}
{{% tab name="Python" %}}

Install the Viam Python SDK:

```bash
pip install viam-sdk
```

Create a file called `connect.py`:

```python
import asyncio
from viam.robot.client import RobotClient


async def connect():
    opts = RobotClient.Options.with_api_key(
        api_key="YOUR_API_KEY",
        api_key_id="YOUR_API_KEY_ID",
    )
    robot = await RobotClient.at_address(
        "YOUR_MACHINE_ADDRESS",
        opts,
    )

    print("Connected!")
    print("Resources:")
    print(robot.resource_names)

    await robot.close()


if __name__ == "__main__":
    asyncio.run(connect())
```

Replace `YOUR_API_KEY`, `YOUR_API_KEY_ID`, and `YOUR_MACHINE_ADDRESS` with the
values from the **CONNECT** tab. Then run:

```bash
python connect.py
```

You should see `Connected!` followed by a list of available resources (empty for
now, since you haven't configured any components yet).

{{% /tab %}}
{{% tab name="Go" %}}

Initialize a Go module and install the Viam Go SDK:

```bash
mkdir viam-connect && cd viam-connect
go mod init viam-connect
go get go.viam.com/rdk/robot/client
```

Create a file called `main.go`:

```go
package main

import (
	"context"
	"fmt"

	"go.viam.com/rdk/logging"
	"go.viam.com/rdk/robot/client"
	"go.viam.com/rdk/utils"
)

func main() {
	logger := logging.NewLogger("client")

	robot, err := client.New(
		context.Background(),
		"YOUR_MACHINE_ADDRESS",
		logger,
		client.WithCredentials(
			utils.Credentials{
				Type:    utils.CredentialsTypeAPIKey,
				Payload: "YOUR_API_KEY",
			},
		),
		client.WithAPIKeyID("YOUR_API_KEY_ID"),
	)
	if err != nil {
		logger.Fatal(err)
	}
	defer robot.Close(context.Background())

	fmt.Println("Connected!")
	fmt.Println("Resources:")
	fmt.Println(robot.ResourceNames())
}
```

Replace `YOUR_API_KEY`, `YOUR_API_KEY_ID`, and `YOUR_MACHINE_ADDRESS` with the
values from the **CONNECT** tab. Then run:

```bash
go run main.go
```

You should see `Connected!` followed by a list of available resources.

{{% /tab %}}
{{< /tabs >}}

## Try It

Run your connection script (Python or Go). Confirm that:

1. The script prints `Connected!` without errors.
2. The resource list appears (it will be empty or minimal at this stage).
3. In the Viam app, your machine's status indicator stays green while the script
   runs.

If all three checks pass, your machine is connected and ready for the next
block.

## Troubleshooting

{{< expand "Machine shows offline in the Viam app" >}}

- **Is `viam-agent` running?** On Linux, check with
  `sudo systemctl status viam-agent`. If it's not running, start it with
  `sudo systemctl start viam-agent`. This will also start `viam-server`.
- **Does the machine have network access?** Verify with `ping google.com`.
- **Is the config correct?** Inspect `/etc/viam.json` and confirm it contains
  valid credentials. If in doubt, re-run the install command from the Viam app.

{{< /expand >}}

{{< expand "Connection script fails with \"context deadline exceeded\"" >}}

- Confirm your machine is online (green dot in the Viam app).
- Verify the machine address is correct — it should look like
  `my-first-machine-main.1234abcd.viam.cloud`.
- Check that your laptop/desktop has internet access.

{{< /expand >}}

{{< expand "Connection script fails with \"authentication failed\"" >}}

- Double-check that you copied the API key, API key ID, and machine address
  correctly from the **CONNECT** tab.
- Make sure there are no trailing spaces or newlines in the credentials.
- Verify the API key hasn't been revoked — you can check this in the Viam app
  under your organization's settings.

{{< /expand >}}

{{< expand "\"Permission denied\" during install" >}}

- The install script requires `sudo`. Run the `curl` command exactly as shown on
  the setup page.
- On some systems, your user may not be in the `sudo` group. Consult your
  device's documentation for how to grant sudo access.

{{< /expand >}}

{{< expand "viam-server starts but immediately exits" >}}

- Check the logs: `sudo journalctl -u viam-server -n 50`.
- Common causes: invalid JSON in `/etc/viam.json`, port conflicts (another
  service on port 8080), or missing system libraries.

{{< /expand >}}

## What's Next

- [Add a Camera](/hardware/common-components/add-a-camera/) — Configure your first hardware
  component and see a live feed.
- [Capture and Sync Data](/data/capture-and-sync-data/) — Start recording data from
  your machine to the cloud.
