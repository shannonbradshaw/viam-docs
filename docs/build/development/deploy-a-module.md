---
linkTitle: "Deploy a Module"
title: "Deploy a Module"
weight: 30
layout: "docs"
type: "docs"
description: "Package, upload, and distribute a module through the Viam registry."
date: "2025-01-30"
---

## What Problem This Solves

A module that only runs locally on your development machine is useful for testing
but does not scale. If you have ten machines that need your custom sensor driver,
you do not want to SSH into each one and copy files. The Viam module registry
lets you upload your module once and install it on any machine in your
organization (or publicly) through the Viam app. When you release a new version,
machines update automatically.

## Concepts

### The module registry

The Viam module registry is a package manager for modules. It stores versioned
module packages and serves them to machines on demand. When you configure a
module on a machine, `viam-server` downloads the correct version for the
machine's platform (OS and architecture).

Modules can be:

- **Private** -- visible only to your organization.
- **Public** -- visible to all Viam users.

### meta.json

Every module has a `meta.json` file that describes it to the registry:

- **module_id** -- unique identifier in the format `namespace:module-name`.
- **visibility** -- `private` or `public`.
- **description** -- a summary shown in the registry.
- **models** -- the list of resource models this module provides.
- **entrypoint** -- the command that starts the module.
- **build** -- configuration for cloud builds.

### Cloud build

Cloud build uses GitHub Actions to compile your module for multiple platforms
automatically. When you push a version tag (e.g., `v0.1.0`), a workflow builds
your module for each target architecture, packages it, and uploads it to the
registry.

### Versioning

The registry uses semantic versioning. Machines can be configured to:

- **Track the latest version** -- automatically update when a new version is
  uploaded (default).
- **Pin to a specific version** -- stay on a fixed version.

## Steps

### 1. Prepare meta.json

Open the `meta.json` file in your module directory. Review and update each
field:

```json
{
  "module_id": "my-org:my-sensor-module",
  "visibility": "private",
  "url": "https://github.com/my-org/my-sensor-module",
  "description": "A custom sensor module that reads temperature and humidity from an HTTP endpoint.",
  "models": [
    {
      "api": "rdk:component:sensor",
      "model": "my-org:my-sensor-module:my-sensor"
    }
  ],
  "entrypoint": "run.sh",
  "build": {
    "setup": "./setup.sh",
    "build": "./build.sh",
    "path": "dist/archive.tar.gz",
    "arch": ["linux/amd64", "linux/arm64"]
  }
}
```

| Field | Purpose |
|-------|---------|
| `module_id` | Unique ID in the registry. Format: `namespace:name`. |
| `visibility` | Who can see and install the module. |
| `url` | Link to the source code repository. |
| `description` | Shown in the registry UI and search results. |
| `models` | List of resource models the module provides. Each has an `api` and `model` triplet. |
| `entrypoint` | The command that starts the module. |
| `build.setup` | Script that installs build dependencies. |
| `build.build` | Script that compiles and packages the module. |
| `build.path` | Path to the packaged output archive. |
| `build.arch` | Target platforms to build for. |

Common `api` values:

- `rdk:component:sensor` for sensors
- `rdk:component:camera` for cameras
- `rdk:component:motor` for motors
- `rdk:component:generic` for generic components
- `rdk:service:vision` for vision services

### 2. Create build and setup scripts

{{< tabs >}}
{{% tab name="Python" %}}

**`setup.sh`:**

```bash
#!/bin/bash
set -e
pip install -r requirements.txt
```

**`build.sh`:**

```bash
#!/bin/bash
set -e

mkdir -p dist

tar -czf dist/archive.tar.gz \
    src/ \
    requirements.txt \
    meta.json \
    run.sh
```

**`run.sh`** (the entrypoint):

```bash
#!/bin/bash
cd "$(dirname "$0")"
exec python3 -m src.main "$@"
```

{{% /tab %}}
{{% tab name="Go" %}}

**`setup.sh`:**

```bash
#!/bin/bash
set -e
# Go modules are self-contained; no setup needed.
```

**`build.sh`:**

```bash
#!/bin/bash
set -e

mkdir -p dist

GOOS=linux GOARCH=$TARGET_ARCH go build -o dist/module cmd/module/main.go

tar -czf dist/archive.tar.gz -C dist module
```

{{% /tab %}}
{{< /tabs >}}

Make all scripts executable:

```bash
chmod +x setup.sh build.sh run.sh
```

### 3. Set up cloud build

Cloud build automates the build-and-upload process using GitHub Actions.

**Create a GitHub repository** (if needed):

```bash
cd my-sensor-module
git init
git add .
git commit -m "Initial module code"
git remote add origin https://github.com/my-org/my-sensor-module.git
git push -u origin main
```

**Add Viam credentials as GitHub secrets:**

1. In the [Viam app](https://app.viam.com), go to your organization's settings.
2. Create an API key with organization-level access (or use an existing one).
3. In your GitHub repository, go to **Settings > Secrets and variables >
   Actions**.
4. Add two secrets:
   - `VIAM_KEY_ID` -- your API key ID
   - `VIAM_KEY_VALUE` -- your API key

**Create the workflow file:**

Create `.github/workflows/deploy.yml`:

```yaml
on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    uses: viamrobotics/build-action/.github/workflows/deploy.yml@v1
    with:
      ref: ${{ github.ref }}
    secrets:
      viam_key_id: ${{ secrets.VIAM_KEY_ID }}
      viam_key_value: ${{ secrets.VIAM_KEY_VALUE }}
```

**Trigger a build:**

```bash
git add .github/workflows/deploy.yml
git commit -m "Add cloud build workflow"
git push origin main

git tag v0.1.0
git push origin v0.1.0
```

The GitHub Action runs automatically. Monitor progress in the **Actions** tab
of your GitHub repository.

### 4. Upload manually (alternative)

If you do not want to use cloud build, build and upload from the command line.

**Build locally:**

{{< tabs >}}
{{% tab name="Python" %}}

```bash
cd my-sensor-module
bash build.sh
```

{{% /tab %}}
{{% tab name="Go" %}}

```bash
cd my-sensor-module
go build -o dist/module cmd/module/main.go
tar -czf dist/archive.tar.gz -C dist module
```

{{% /tab %}}
{{< /tabs >}}

**Upload to the registry:**

```bash
viam module upload \
    --version=0.1.0 \
    --platform=linux/amd64 \
    dist/archive.tar.gz
```

To upload for multiple platforms, run the command once per platform:

```bash
viam module upload \
    --version=0.1.0 \
    --platform=linux/arm64 \
    dist/archive.tar.gz
```

### 5. Configure the module on a machine

Once your module is in the registry, any machine in your organization can use it.

1. In the [Viam app](https://app.viam.com), navigate to your machine's
   **CONFIGURE** tab.
2. Click **+** and select **Component** (or **Service**).
3. Search for your module by name or browse the registry.
4. Click **Add module** to add it to the machine.
5. Click **Add component** (or **Add service**) to create an instance.
6. Name the component and configure the attributes your module expects:

```json
{
  "source_url": "https://api.example.com/sensor/data"
}
```

7. Click **Save**.

`viam-server` downloads the module from the registry, starts it, and makes the
component available. Test it from the **CONTROL** tab.

### 6. Manage versions

**Release a new version:**

```bash
git add .
git commit -m "Add humidity calibration offset"
git tag v0.2.0
git push origin main v0.2.0
```

If using cloud build, the workflow runs automatically. For manual upload:

```bash
viam module upload --version=0.2.0 --platform=linux/amd64 dist/archive.tar.gz
```

**Automatic updates:** By default, machines track the latest version. When you
upload `v0.2.0`, all machines update automatically within a few minutes.

**Pin to a specific version:**

1. In the Viam app, go to the machine's **CONFIGURE** tab.
2. Find the module in the configuration.
3. Set the **Version** field to the specific version (e.g., `0.1.0`).
4. Click **Save**.

**View available versions:**

```bash
viam module list --module-id my-org:my-sensor-module
```

## Try It

1. Prepare your `meta.json` with the correct module ID, models, and build
   configuration.
2. Upload your module to the registry (either through cloud build or manually).
3. Navigate to a machine in the Viam app and add your module from the registry.
4. Configure a component and set the required attributes.
5. Open the **CONTROL** tab and verify the component works.
6. Upload a new version (e.g., `v0.2.0`) and verify the machine picks it up
   automatically within a few minutes.

## Troubleshooting

{{< expand "Upload fails with \"not authenticated\"" >}}

- Log in to the Viam CLI: `viam login`.
- If using an API key, verify it has organization-level access.
- Check that `VIAM_KEY_ID` and `VIAM_KEY_VALUE` are set correctly in your GitHub
  secrets (for cloud build).

{{< /expand >}}

{{< expand "Upload fails with \"invalid meta.json\"" >}}

- Verify `meta.json` is valid JSON. Run `python -m json.tool meta.json` or
  `jq . meta.json` to check.
- Confirm the `module_id` matches the format `namespace:module-name`.
- Ensure all model entries have both `api` and `model` fields.

{{< /expand >}}

{{< expand "Module not appearing in the registry" >}}

- Check the module's visibility. If it is `private`, it only appears for users
  in your organization.
- Verify the upload completed successfully.
- The module may take a minute to propagate. Refresh the page and try again.

{{< /expand >}}

{{< expand "Machine cannot find the module" >}}

- Verify the module version supports the machine's platform. If your machine
  runs `linux/arm64` but you only uploaded for `linux/amd64`, the machine
  cannot use it.
- Check the module version. If the machine is pinned to a nonexistent version,
  it will fail.
- Confirm the machine is online and connected to the cloud.

{{< /expand >}}

{{< expand "Cloud build fails in GitHub Actions" >}}

- Check the Actions tab in your GitHub repository for the build log.
- Verify your `setup.sh` and `build.sh` scripts work locally.
- Confirm the `build.path` in `meta.json` matches the actual output location.
- Ensure the GitHub secrets are set and not expired.

{{< /expand >}}

{{< expand "Module works locally but fails after deployment" >}}

- Check for hard-coded paths, missing environment variables, or dependencies
  installed on your machine but not in the build environment.
- For Python, verify all dependencies are in `requirements.txt`.
- For Go, verify the binary is compiled for the correct target architecture.
- Check the module logs in the **LOGS** tab.

{{< /expand >}}

## What's Next

- [Write an Inline Module](/build/development/write-an-inline-module/) --
  prototype quickly with an inline module before building a full deployable
  module.
- [Filter at the Edge](/build/data/filter-at-the-edge/) -- deploy filtering
  modules across your fleet to reduce data costs.
- [Add Computer Vision](/build/vision-detection/add-computer-vision/) -- deploy
  vision service modules that run ML models on your machines.
