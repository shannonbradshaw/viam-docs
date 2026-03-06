---
linkTitle: "Module Reference"
title: "Module Developer Reference"
weight: 40
layout: "docs"
type: "docs"
description: "Reference for module developers: lifecycle, interfaces, meta.json schema, CLI commands, environment variables, and registry rules."
date: "2025-03-05"
aliases:
  - /development/module-reference/
---

This page is a quick-reference for module developers. For step-by-step
instructions, see [Write a Module](/build-modules/write-a-driver-module/) and
[Deploy a Module](/build-modules/deploy-a-module/).

## Module lifecycle

Every module — local or registry — runs as a separate child process alongside
`viam-server`, communicating over [gRPC](#module-protocol).

1. `viam-server` starts and checks for configuration updates (if online).
2. If the module has a [`first_run`](#first-run-scripts) script that hasn't
   succeeded yet, `viam-server` runs it before starting the module.
3. `viam-server` starts each configured module as a child process, passing it a
   socket address.
4. The module registers its models and APIs with `viam-server` via the
   [`Ready` RPC](#module-protocol).
5. For each configured resource, `viam-server` calls `ValidateConfig` to check
   attributes and discover dependencies.
6. `viam-server` starts required dependencies first. If a required dependency
   fails, the resource that depends on it does not start.
7. `viam-server` calls `AddResource` to create each resource. The module's
   constructor runs, typically calling `Reconfigure` to read config.
8. The resource is available for use.
9. When the user changes configuration, `viam-server` calls
   `ReconfigureResource`. Your `Reconfigure` method should complete
   within 15 seconds to avoid blocking other operations.
10. On shutdown, `viam-server` sends `RemoveResource` for each resource, then
    terminates the module process.

### First-run scripts

If your module needs one-time setup (installing system packages, downloading
models, etc.), set the `first_run` field in [`meta.json`](#metajson-schema) to
the path of a setup script inside your archive.

- The script runs **before** the module entrypoint, with the same
  [environment variables](#environment-variables) available to the module.
- If the script exits with a non-zero status, reconfiguration is aborted.
  Currently running modules continue with their previous configuration.
- On success, a `.first_run_succeeded` marker file is created next to the
  module binary. The script will not re-run unless this marker is deleted
  or a new module version is installed.
- The default timeout is 1 hour, configurable via `first_run_timeout` in the
  module config.

### Crash recovery

If a module process crashes, `viam-server` automatically restarts it:

1. `viam-server` detects the exit and marks the module as failed.
2. The machine's status changes to **initializing**.
3. `viam-server` retries every 5 seconds.
4. On success, `viam-server` re-adds all resources in dependency order.
5. The machine returns to **running**.

If the module keeps crashing, `viam-server` retries indefinitely. Check the
**LOGS** tab for crash tracebacks.

### Communication

By default, modules communicate with `viam-server` over a Unix domain socket
with a randomized name.

TCP mode is used automatically on Windows or when the Unix socket path would
exceed the OS limit (103 characters on macOS). Force TCP mode by setting
`"tcp_mode": true` in the module config or setting `VIAM_TCP_SOCKETS=true`.

### Timeouts

| Event | Timeout | Behavior on timeout |
|-------|---------|---------------------|
| Module startup (ready check) | 5 minutes (configurable via `VIAM_MODULE_STARTUP_TIMEOUT`) | Module marked as failed; retry begins |
| Config validation | 5 seconds | Validation fails; resource does not start |
| Resource removal during shutdown | 20 seconds (all resources combined) | Resources orphaned |
| Module closure (removal + process stop) | ~30 seconds total | Process killed |
| First-run setup script | 1 hour (configurable via `first_run_timeout`) | Module startup fails |
| Crash restart retry interval | 5 seconds | Next attempt after delay |

## Module protocol

Modules communicate with `viam-server` over gRPC using the `ModuleService`
defined in `proto/viam/module/v1/module.proto`:

| RPC | Direction | Purpose |
|-----|-----------|---------|
| `Ready` | server → module | Handshake: module returns its supported API/model pairs. |
| `AddResource` | server → module | Create a new resource instance from config. |
| `ReconfigureResource` | server → module | Update an existing resource with new config. |
| `RemoveResource` | server → module | Destroy a resource instance. |
| `ValidateConfig` | server → module | Validate config and return implicit dependencies. |

The module also connects back to the parent `viam-server` to access other
resources (dependencies) on the machine.

## Resource interfaces (Go)

### Config validation

```go
type ConfigValidator interface {
    Validate(path string) (requiredDependencies, optionalDependencies []string, err error)
}
```

Return required dependency names in the first slice. `viam-server` ensures they
are ready before calling your constructor. Return optional dependency names in
the second slice. These are passed to the constructor if available, but their
absence does not block startup.

### Resource interface

Every resource must implement:

```go
type Resource interface {
    Name() Name
    Reconfigure(ctx context.Context, deps Dependencies, conf Config) error
    DoCommand(ctx context.Context, cmd map[string]interface{}) (map[string]interface{}, error)
    Close(ctx context.Context) error
}
```

Plus the methods defined by the specific API (e.g., `Readings` for sensor,
`GetImage` for camera).

### Constructor signature

```go
func(ctx context.Context, deps resource.Dependencies,
    conf resource.Config, logger logging.Logger) (ResourceT, error)
```

### Helper traits

Embed these in your resource struct to get default implementations:

| Trait | Effect |
|-------|--------|
| `resource.Named` | Interface for `Name()` and `DoCommand()`. Embed and set via `conf.ResourceName().AsNamed()`. |
| `resource.TriviallyCloseable` | `Close()` returns nil. |
| `resource.TriviallyReconfigurable` | `Reconfigure()` returns nil (no-op). |
| `resource.AlwaysRebuild` | `Reconfigure()` returns `MustRebuildError` (always re-create). |
| `resource.TriviallyValidateConfig` | `Validate()` returns no deps and no error. |

### Useful functions

| Function | Description |
|----------|-------------|
| `resource.NativeConfig[*T](conf)` | Convert config attributes to a typed struct. |
| `<api>.FromProvider(deps, name)` | Type-safe dependency lookup (e.g., `sensor.FromProvider(deps, "my-sensor")`). |
| `conf.ResourceName().AsNamed()` | Create a `Named` implementation from config. |
| `module.ModularMain(models...)` | Convenience entry point for simple modules. |
| `module.NewModuleFromArgs(ctx)` | Create a module from CLI args (for custom entry points). |
| `module.NewLoggerFromArgs(name)` | Create a logger that routes to `viam-server`. |

## Resource interfaces (Python)

### Config validation

```python
@classmethod
def validate_config(cls, config: ComponentConfig) -> Tuple[Sequence[str], Sequence[str]]:
    # Return (required_deps, optional_deps)
    return [], []
```

### Constructor

```python
@classmethod
def new(cls, config: ComponentConfig,
        dependencies: Mapping[ResourceName, ResourceBase]) -> Self:
    instance = cls(config.name)
    instance.reconfigure(config, dependencies)
    return instance
```

### Reconfigure

```python
def reconfigure(self, config: ComponentConfig,
                dependencies: Mapping[ResourceName, ResourceBase]) -> None:
    # Update internal state from new config
    pass
```

### Close

```python
async def close(self):
    # Clean up connections, stop background tasks, release hardware.
    # Must be idempotent (safe to call multiple times).
    pass
```

The default `close()` on `ResourceBase` is a no-op.

### EasyResource base class

For simple modules, inherit from both the API base class and `EasyResource` to
get default `new()`, `validate_config()`, and automatic model registration:

```python
from viam.components.sensor import Sensor
from viam.resource.easy_resource import EasyResource
from viam.module.module import Module

class MySensor(Sensor, EasyResource):
    MODEL = "my-org:my-module:my-sensor"

    async def get_readings(self, **kwargs):
        return {"temperature": 23.5}

if __name__ == '__main__':
    import asyncio
    asyncio.run(Module.run_from_registry())
```

### Useful functions

| Function | Description |
|----------|-------------|
| `Module.from_args()` | Create a module from CLI args. |
| `Module.run_from_registry()` | Discover and register all imported resource classes, then start. |
| `Module.run_with_models(*models)` | Register explicit model classes, then start. |
| `getLogger(name)` | Create a logger (`from viam.logging import getLogger`). |
| `config.attributes.fields` | Access raw config attributes (no typed config equivalent to Go). |

### Python vs. Go defaults

In Python, the default behavior when you don't implement a method differs from Go:

| Behavior | Go | Python |
|----------|-----|--------|
| Seamless reconfigure | Implement `Reconfigure()` | Implement `reconfigure()` (called if your class satisfies the `Reconfigurable` protocol) |
| Rebuild on config change | Embed `resource.AlwaysRebuild` | Omit `reconfigure()` (default — module destroys and re-creates the resource) |
| No-op reconfigure | Embed `resource.TriviallyReconfigurable` | No equivalent — implement an empty `reconfigure()` instead |
| No-op close | Embed `resource.TriviallyCloseable` | Default on `ResourceBase` |
| Skip config validation | Embed `resource.TriviallyValidateConfig` | Default on `EasyResource` |

## meta.json schema

Every module has a `meta.json` file that describes the module to the registry.
The full schema is available at `https://dl.viam.dev/module.schema.json`.

```json
{
  "$schema": "https://dl.viam.dev/module.schema.json",
  "module_id": "my-org:my-module",
  "visibility": "private",
  "url": "https://github.com/my-org/my-module",
  "description": "Short description of the module.",
  "models": [
    {
      "api": "rdk:component:sensor",
      "model": "my-org:my-module:my-sensor",
      "description": "A short description of this model.",
      "markdown_link": "README.md#my-sensor"
    }
  ],
  "entrypoint": "run.sh",
  "first_run": "setup.sh",
  "markdown_link": "README.md",
  "build": {
    "setup": "./setup.sh",
    "build": "./build.sh",
    "path": "module.tar.gz",
    "arch": ["linux/amd64", "linux/arm64"]
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `$schema` | string | No | JSON Schema URL for editor validation. |
| `module_id` | string | Yes | `namespace:name` or `org-id:name`. |
| `visibility` | string | Yes | `private`, `public`, or `public_unlisted`. |
| `url` | string | No | Source repo URL. Required for cloud builds. |
| `description` | string | Yes | Short description shown in the registry. |
| `models` | array | Yes | List of API/model pairs the module provides. |
| `models[].api` | string | Yes | Resource API (e.g., `rdk:component:sensor`). |
| `models[].model` | string | Yes | Model triplet (e.g., `my-org:my-module:my-sensor`). |
| `models[].description` | string | No | Short model description (max 100 chars). |
| `models[].markdown_link` | string | No | Path to model docs within the repo. |
| `entrypoint` | string | Yes | Path to the executable inside the archive. |
| `first_run` | string | No | Path to a one-time setup script. Runs before the entrypoint on first install and after version updates. See [First-run scripts](#first-run-scripts). |
| `markdown_link` | string | No | Path to README used as registry description. |
| `build` | object | No | Build configuration for local and cloud builds. |
| `build.setup` | string | No | One-time setup command (e.g., install dependencies). |
| `build.build` | string | No | Build command (e.g., `make module.tar.gz`). |
| `build.path` | string | No | Path to built artifact. Default: `module.tar.gz`. |
| `build.arch` | array | No | Target platforms. Default: `["linux/amd64", "linux/arm64"]`. |
| `build.darwin_deps` | array | No | Homebrew dependencies for macOS builds (e.g., `["go", "pkg-config"]`). |

## CLI commands

All module CLI commands are under `viam module`. You must be logged in
(`viam login`) to use commands that interact with the registry.

### Create and generate

| Command | Description |
|---------|-------------|
| `viam module create --name <name>` | Register a module in the registry and generate `meta.json`. |
| `viam module generate` | Scaffold a complete module project with templates (interactive prompts). |

`generate` flags: `--name`, `--language` (`python` or `go`), `--visibility`,
`--public-namespace`, `--resource-subtype`, `--model-name`, `--register`,
`--dry-run`.

### Build

| Command | Description |
|---------|-------------|
| `viam module build local` | Run the build command from `meta.json` locally. |
| `viam module build start --version <semver>` | Start a cloud build for all configured platforms. |
| `viam module build list` | List cloud build jobs and their status. |
| `viam module build logs --id <build-id>` | Stream logs from a cloud build job. |

`build start` flags: `--ref` (git ref, default: `main`), `--platforms`,
`--token` (for private repos), `--workdir`.

During builds, the environment variables `VIAM_BUILD_OS` and `VIAM_BUILD_ARCH`
are set to the target platform. See [Environment variables](#environment-variables).

### Upload and update

| Command | Description |
|---------|-------------|
| `viam module upload --version <semver> --platform <platform>` | Upload a built archive to the registry. |
| `viam module update` | Push updated `meta.json` to the registry. |
| `viam module update-models --binary <path>` | Auto-detect models from a binary and update `meta.json`. |
| `viam module download --id <module-id>` | Download a module from the registry. |

`upload` flags: `--tags` (platform constraints), `--force` (skip validation),
`--upload` (path to archive).

### Development loop

| Command | Description |
|---------|-------------|
| `viam module reload-local --part-id <id>` | Build locally, transfer to machine, configure, and restart. |
| `viam module reload --part-id <id>` | Build in cloud, transfer to machine, configure, and restart. |
| `viam module restart --part-id <id>` | Restart a running module without rebuilding. |

`reload-local` flags: `--part-id` (target machine part), `--no-build` (skip
build), `--local` (run entrypoint directly on localhost instead of bundling),
`--model-name` (add a resource to config with this model triple),
`--name` (name the added resource), `--resource-name` (name the resource
instance), `--id` (module ID, alternative to `--name`),
`--cloud-config` (path to `viam.json`, alternative to `--part-id`),
`--workdir` (subdirectory containing `meta.json`), `--home-dir` (remote
user's home directory), `--no-progress` (hide transfer progress).

## Environment variables

These environment variables are available inside a running module process
(including [first-run scripts](#first-run-scripts)):

| Variable | Description |
|----------|-------------|
| `VIAM_MODULE_NAME` | The module's name from config. |
| `VIAM_MODULE_DATA` | Path to the module's persistent data directory. |
| `VIAM_MODULE_ROOT` | Parent directory of the module executable. |
| `VIAM_MODULE_ID` | Registry module ID (registry modules only). |
| `VIAM_HOME` | Path to the Viam home directory (`~/.viam`). |

Custom environment variables can be added in the module's machine config under
the `env` field.

The following variables are set during [cloud builds](#build), not at runtime:

| Variable | Description |
|----------|-------------|
| `VIAM_BUILD_OS` | Target operating system (e.g., `linux`, `darwin`). |
| `VIAM_BUILD_ARCH` | Target architecture (e.g., `amd64`, `arm64`). |

The following variable controls module startup behavior:

| Variable | Description |
|----------|-------------|
| `VIAM_MODULE_STARTUP_TIMEOUT` | Override the default 5-minute startup timeout (e.g., `10m`, `30s`). |

## Supported platforms

| Platform | Cloud build support |
|----------|-------------------|
| `linux/amd64` | Yes |
| `linux/arm64` | Yes |
| `linux/arm32v6` | No |
| `linux/arm32v7` | No |
| `darwin/amd64` | No |
| `darwin/arm64` | Yes |
| `windows/amd64` | Yes |
| `any` | No (use for platform-independent modules) |

## Common gotchas

**Always call `Reconfigure` from your constructor.**
Your constructor and `Reconfigure` should share the same config-reading logic.
The typical pattern is for the constructor to create the struct, then call
`Reconfigure` to populate it from config. This avoids duplicating config
parsing and ensures a newly created resource is fully configured.

**Clean up in `Close()`.**
If your resource starts background goroutines, opens connections, or holds
hardware handles, `Close()` must stop them. Leaked goroutines accumulate across
reconfigurations and can cause instability.
In Python, `close()` must be idempotent — it may be called more than once.

**Return the right dependency names from `Validate`.**
Dependencies listed as required in `Validate` (Go) or `validate_config`
(Python) must match actual resource names in the machine config. If the name
is wrong, `viam-server` waits for a resource that will never exist, and your
resource will not start. Use optional dependencies for resources that improve
functionality but aren't strictly needed.

**Prefer `Reconfigure` over `AlwaysRebuild`.**
`AlwaysRebuild` (Go) or omitting `reconfigure()` (Python) causes the resource
to be destroyed and re-created on every config change. This is simpler but
causes a brief availability gap. Implementing `Reconfigure` to update state
in-place provides seamless reconfiguration.

## Registry validation rules

| Rule | Constraint |
|------|-----------|
| Module name | 1-200 characters, `^[a-zA-Z0-9][-\w]*$` (must start with alphanumeric; may contain hyphens, underscores, letters, digits) |
| Module version | Semantic versioning 2.0.0 (e.g., `1.2.3`) |
| Package name | `^[\w-]+$` |
| Metadata fields | Max 16 key-value pairs |
| Metadata key/value size | Max 500 KB each |
| Compressed package | Max 50 GB |
| Decompressed contents | Max 250 GB |
| Single file in package | Max 25 GB |
| Model namespace | Must match org namespace if org has one. Cannot use reserved namespace `rdk`. |
| Public modules | Require org to have a public namespace. Cannot use `public_unlisted` → `private` if external orgs are using the module. |
