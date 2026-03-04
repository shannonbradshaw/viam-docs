---
linkTitle: "Write an Inline Module"
title: "Write an Inline Module"
weight: 10
layout: "docs"
type: "docs"
description: "Build a custom inline module using the Generic component and DoCommand pattern."
date: "2025-01-30"
aliases:
  - /build/development/write-an-inline-module/
---

## What Problem This Solves

Viam provides built-in support for many types of hardware and software, but your
project may require something Viam does not support out of the box -- a custom
sensor protocol, a proprietary actuator, or application-specific logic. Modules
let you add that support yourself.

An inline module is the fastest way to get started. It runs inside the
`viam-server` process, so there is no separate binary to build or deploy. You
write your logic, point `viam-server` at it, and your custom component is
available immediately. This makes inline modules ideal for prototyping, one-off
integrations, and simple extensions.

## Concepts

### What is a module?

A module is a package that adds support for custom hardware or software to Viam.
When `viam-server` starts, it loads your module and makes the resources defined
in it available alongside built-in components and services. You interact with
module-provided resources the same way you interact with built-in ones --
through the Viam app, the SDKs, or the API.

### Inline modules vs. separate-process modules

There are two ways to run a module:

- **Inline modules** run inside the `viam-server` process. They share memory and
  the process lifecycle with `viam-server`. This is simpler to set up but means
  the module cannot have conflicting dependencies or crash independently of the
  server.
- **Separate-process modules** run as their own process, communicating with
  `viam-server` over a Unix socket. This provides isolation, allows any
  dependencies, and lets you write modules in any language. Separate-process
  modules are what you use for production deployments.

Start with an inline module when you are prototyping. Move to a separate-process
module when you need isolation, complex dependencies, or want to distribute the
module to other users.

### The Generic component

When no specific component API (camera, sensor, motor, etc.) fits your use case,
use the Generic component. Generic provides a single method -- `DoCommand` --
that accepts an arbitrary JSON command and returns a JSON response.

Use Generic when:

- Your component does not map to any existing Viam API.
- You need a simple command-response interface.
- You are prototyping and do not yet know which API you will use long-term.

### The DoCommand pattern

`DoCommand` takes a map of string keys to arbitrary values and returns a map in
the same format. A common pattern is to include an `action` key that selects
what the component should do:

```json
{"action": "toggle"}
{"action": "set_speed", "speed": 100}
{"action": "get_status"}
```

The component inspects the command, performs the requested action, and returns a
result:

```json
{"status": "on"}
{"speed": 100, "direction": "forward"}
{"error": "unknown action"}
```

## Steps

### 1. Generate a module scaffold

The Viam CLI generates a complete module project with the correct file
structure, boilerplate, and metadata:

```bash
viam module generate --resource-subtype=generic-component
```

The generator asks for several inputs:

| Prompt | What to enter | Why |
|--------|---------------|-----|
| Module name | `my-inline-module` | A short, descriptive name |
| Language | `python` or `go` | Your implementation language |
| Visibility | `private` | Keep it private while developing |
| Namespace | Your organization namespace | Scopes the module to your org |
| Resource type | `component` | Already set by the flag |
| Model name | `my-toggle` | The model name for your component |
| Cloud build | `no` | Not needed for inline modules |
| Register | `yes` | Registers the module with Viam |

### 2. Understand the generated files

{{< tabs >}}
{{% tab name="Python" %}}

```
my-inline-module/
  src/
    models/
      my_toggle.py      # Your component logic goes here
    main.py             # Module entrypoint
  requirements.txt      # Python dependencies
  meta.json             # Module metadata for Viam
```

- `src/models/my_toggle.py` contains the model class where you implement your
  component logic. This is where you spend most of your time.
- `src/main.py` is the entrypoint that registers your model and starts the
  module. You rarely need to edit this.
- `meta.json` describes the module to Viam -- its ID, models, entrypoint, and
  build configuration.

{{% /tab %}}
{{% tab name="Go" %}}

```
my-inline-module/
  my_toggle.go          # Your component logic goes here
  cmd/
    module/
      main.go           # Module entrypoint
  go.mod                # Go module dependencies
  meta.json             # Module metadata for Viam
```

- `my_toggle.go` contains the model struct and methods. This is where you
  implement your logic.
- `cmd/module/main.go` is the entrypoint that registers the model and starts the
  module server.
- `meta.json` serves the same purpose as in Python.

{{% /tab %}}
{{< /tabs >}}

### 3. Implement validate_config

The validate method runs every time the machine configuration changes. It checks
that the attributes passed to your component are valid and declares dependencies
on other components.

This example validates that a `board_name` and `pin` attribute are present:

{{< tabs >}}
{{% tab name="Python" %}}

```python
@classmethod
def validate_config(cls, config: ComponentConfig) -> Tuple[Sequence[str], Sequence[str]]:
    fields = config.attributes.fields
    if "board_name" not in fields:
        raise Exception("missing required board_name attribute")
    if "pin" not in fields:
        raise Exception("missing required pin attribute")
    board_name = fields["board_name"].string_value
    return [board_name], []  # required deps, optional deps
```

The return value is a tuple of two lists:

1. **Required dependencies** -- component names that must exist and be ready
   before your component starts.
2. **Optional dependencies** -- component names that your component can use if
   available but does not require.

If validation raises an exception, `viam-server` logs the error and does not
create the component.

{{% /tab %}}
{{% tab name="Go" %}}

```go
type Config struct {
    BoardName string `json:"board_name"`
    Pin       string `json:"pin"`
}

func (cfg *Config) Validate(path string) ([]string, error) {
    if cfg.BoardName == "" {
        return nil, fmt.Errorf("board_name is required")
    }
    if cfg.Pin == "" {
        return nil, fmt.Errorf("pin is required")
    }
    return []string{cfg.BoardName}, nil
}
```

The `Validate` method returns a slice of dependency names and an error. Config
struct fields are populated automatically from JSON attributes using struct tags.

{{% /tab %}}
{{< /tabs >}}

### 4. Implement reconfigure

The `reconfigure` method runs when the component is first created and again
whenever its configuration changes. Use it to read attributes and store
references to dependencies.

{{< tabs >}}
{{% tab name="Python" %}}

```python
@classmethod
def new(cls, config: ComponentConfig,
        dependencies: Mapping[ResourceName, ResourceBase]) -> Self:
    component = cls(config.name)
    component.reconfigure(config, dependencies)
    return component

def reconfigure(self, config: ComponentConfig,
                dependencies: Mapping[ResourceName, ResourceBase]) -> None:
    fields = config.attributes.fields
    board_name = fields["board_name"].string_value
    self.pin = fields["pin"].string_value
    self.is_on = False

    # Resolve the board dependency.
    for name, dep in dependencies.items():
        if name.name == board_name:
            self.board = dep
            break
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
func newMyToggle(
    ctx context.Context,
    deps resource.Dependencies,
    conf resource.Config,
    logger logging.Logger,
) (resource.Resource, error) {
    cfg, err := resource.NativeConfig[*Config](conf)
    if err != nil {
        return nil, err
    }

    board, err := board.FromDependencies(deps, cfg.BoardName)
    if err != nil {
        return nil, fmt.Errorf("board %q not found: %w", cfg.BoardName, err)
    }

    return &MyToggle{
        Named:  conf.ResourceName().AsNamed(),
        board:  board,
        pin:    cfg.Pin,
        isOn:   false,
        logger: logger,
    }, nil
}
```

{{% /tab %}}
{{< /tabs >}}

### 5. Implement do_command

`DoCommand` is the core entry point for your component's logic. This example
implements a toggle switch that flips a GPIO pin on or off:

{{< tabs >}}
{{% tab name="Python" %}}

```python
async def do_command(self, command: Mapping[str, ValueTypes], *,
                     timeout: Optional[float] = None,
                     **kwargs) -> Mapping[str, ValueTypes]:
    if "action" not in command:
        return {"error": "missing action field"}

    action = command["action"]

    if action == "toggle":
        self.is_on = not self.is_on
        pin = await self.board.gpio_pin_by_name(self.pin)
        await pin.set(self.is_on)
        return {"status": "on" if self.is_on else "off"}

    if action == "get_status":
        return {"status": "on" if self.is_on else "off"}

    return {"error": f"unknown action: {action}"}
```

{{% /tab %}}
{{% tab name="Go" %}}

```go
func (c *MyToggle) DoCommand(ctx context.Context,
    cmd map[string]interface{}) (map[string]interface{}, error) {

    action, ok := cmd["action"].(string)
    if !ok {
        return nil, fmt.Errorf("missing or invalid action field")
    }

    switch action {
    case "toggle":
        c.isOn = !c.isOn
        pin, err := c.board.GPIOPinByName(c.pin)
        if err != nil {
            return nil, fmt.Errorf("failed to get pin %s: %w", c.pin, err)
        }
        if err := pin.Set(ctx, c.isOn, nil); err != nil {
            return nil, fmt.Errorf("failed to set pin: %w", err)
        }
        status := "off"
        if c.isOn {
            status = "on"
        }
        return map[string]interface{}{"status": status}, nil

    case "get_status":
        status := "off"
        if c.isOn {
            status = "on"
        }
        return map[string]interface{}{"status": status}, nil

    default:
        return nil, fmt.Errorf("unknown action: %s", action)
    }
}
```

{{% /tab %}}
{{< /tabs >}}

Keep the command parsing straightforward. If your DoCommand grows complex with
many actions, consider whether a specific resource API (sensor, motor, etc.)
would be a better fit.

### 6. Test locally

#### Configure the module on your machine

1. In the [Viam app](https://app.viam.com), navigate to your machine's
   **CONFIGURE** tab.
2. Click **+** and select **Local module**, then **Local module**.
3. Set the **Executable path** to the module's entrypoint:
   - Python: `/path/to/my-inline-module/src/main.py`
   - Go: `/path/to/my-inline-module/cmd/module/main` (after building with
     `go build`)
4. Click **Create**.

#### Add the component

1. Click **+** again and select **Local module**, then **Local component**.
2. Select your module from the dropdown.
3. Set the type to **generic** and the model to your module's model triplet
   (e.g., `my-org:my-inline-module:my-toggle`).
4. Name the component (e.g., `my-toggle`).
5. Configure the attributes:

```json
{
  "board_name": "my-board",
  "pin": "8"
}
```

6. Click **Save**.

#### Send test commands

1. Navigate to the **CONTROL** tab in the Viam app.
2. Find your `my-toggle` component.
3. Open the **DoCommand** panel.
4. Enter a command:

```json
{"action": "toggle"}
```

5. Click **Execute**. You should see a response like:

```json
{"status": "on"}
```

6. Execute it again to toggle off:

```json
{"status": "off"}
```

7. Test the status query:

```json
{"action": "get_status"}
```

## Try It

1. Generate a module scaffold using `viam module generate`.
2. Implement at least `validate_config` and `do_command`.
3. Configure the module on your machine as a local module.
4. Send three different commands from the CONTROL tab's DoCommand panel:
   - A `toggle` command -- verify the status flips.
   - A second `toggle` command -- verify it flips back.
   - A `get_status` command -- verify it reports the current state.
5. Send an unknown command and confirm you get an error response.

## Troubleshooting

{{< expand "Module not loading" >}}

- Check the **LOGS** tab in the Viam app for error messages. Look for import
  errors, missing dependencies, or path issues.
- Verify the executable path in the module configuration points to the correct
  file. Paths must be absolute.
- For Python modules, confirm that the Viam SDK is installed in the Python
  environment that `viam-server` uses. Run `pip install viam-sdk` on the machine.
- For Go modules, make sure you built the binary before configuring the path.
  Run `go build -o cmd/module/main cmd/module/main.go`.

{{< /expand >}}

{{< expand "DoCommand not responding" >}}

- Confirm the component name in the CONTROL tab matches the name you configured.
  Names are case-sensitive.
- Check that the module loaded successfully. If it failed to load, the component
  will not appear in the CONTROL tab.
- Verify the command JSON is valid. The DoCommand panel expects well-formed JSON
  with double-quoted keys.

{{< /expand >}}

{{< expand "validate_config errors" >}}

- Read the error message in the LOGS tab. It tells you exactly which field
  failed validation.
- Confirm the attribute names in your configuration match what `validate_config`
  expects. Check for typos and case sensitivity.
- Make sure dependency names (returned from `validate_config`) match the names of
  actual components on the machine.

{{< /expand >}}

{{< expand "Module crashes when accessing dependencies" >}}

- Dependencies are only available after `validate_config` declares them. If you
  try to use a board or sensor that you did not list as a dependency, it will not
  be in the dependencies map.
- Check the order of operations: `validate_config` runs first, then `new` or
  `reconfigure` receives the resolved dependencies. Do not try to access
  dependencies during validation.

{{< /expand >}}

{{< expand "Changes to code not taking effect" >}}

- For inline modules, `viam-server` loads the module at startup. If you change
  the code, restart `viam-server` or use the **Restart** button in the Viam app
  to pick up the changes.
- For Python, make sure you are editing the correct file. Check the executable
  path in the module configuration.

{{< /expand >}}

## What's Next

- [Write a Module](/development/write-a-module/) -- build a full
  separate-process module with a typed resource API for production use.
- [Deploy a Module](/development/deploy-a-module/) -- package and upload
  your module to the Viam registry so other machines can use it.
- [Filter at the Edge](/data/filter-at-the-edge/) -- use modules to
  implement custom data filtering logic on your machine.
