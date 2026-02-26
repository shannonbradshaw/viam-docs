# IA Reorganization Plan

Branch: `IA-reorg`

## Overview

Three new top-level sections in `docs/`:

- `docs/understand/` — Conceptual orientation
- `docs/try/` — Guided tutorials
- `docs/build/foundation/` — Task-oriented howtos

### Diataxis approach

Every source page touched by this reorg gets cleaned up according to Diataxis:

- **Howto pages** (new, in `build/`): Prerequisites, steps, verification. No explanation, no reference tables, no tutorial narrative.
- **Reference pages** (existing, cleaned up): Attribute tables, supported resources, configuration options, API links. No procedural walkthroughs, no extended explanation.
- **Conceptual pages** (in `understand/`): The "why" and "when." No procedures.
- **Tutorials** (in `try/`): Guided, scenario-based learning journeys. These are the one place where explanation, procedure, and context mix — because that's what tutorials do.

For each source page, the plan specifies what gets extracted into the new howto and what remains as clean reference.

---

## Section 1: Understand

Move existing pages to new paths. Minimal content changes needed.

### Page 1: What is Viam

- **New path:** `docs/understand/what-is-viam.md`
- **Source:** `docs/operate/hello-world/what-is-viam.md` (move)
- **Action:** Move as-is. Add alias from old path. Update front matter (weight, section context).
- **Content changes:** None — page is comprehensive and polished.

### Page 2: Problems Viam Solves

- **New path:** `docs/understand/problems-viam-solves.md`
- **Source:** `docs/operate/hello-world/problems-viam-solves.md` (move)
- **Action:** Move as-is. Add alias from old path. Update front matter.
- **Content changes:** None — page is comprehensive and polished.

### Page 3: Reusable Configurations

- **New path:** `docs/understand/reusable-configurations.md`
- **Source (conceptual content):** `docs-dev/understand/reusable-configurations.md` (new content)
- **Related existing page:** `docs/manage/fleet/reuse-configuration.md` (keep in place — it's the how-to)
- **Action:** Create new conceptual page. The existing `reuse-configuration.md` stays where it is as the implementation how-to; the new page covers *why* and *when* to use fragments, patterns, and architectural thinking.
- **Content to write:**
  - Problems fragments solve (scaling, drift, fleet consistency)
  - 4 fragment patterns (Golden Config, Component Library, Hardware Variant, Provisioning Template)
  - When to use fragments vs. other approaches
  - Link to `manage/fleet/reuse-configuration.md` for implementation details

### Understand section index

- **New path:** `docs/understand/_index.md`
- **Action:** Create section landing page with brief intro and links to the three pages.

### Other pages currently in hello-world

These pages stay in `docs/operate/hello-world/` for now (not part of this reorg):

- `building.md` — "How to think about building a machine"
- `tutorial-desk-safari.md` — Desk Safari tutorial

---

## Section 2: Try

Move the published first-project tutorial to a new path.

### Gazebo Setup

- **New path:** `docs/try/first-project/gazebo-setup.md`
- **Source:** `docs/operate/hello-world/first-project/gazebo-setup.md` (move)
- **Action:** Move. Add alias from old path. Update front matter.
- **Content changes:** None.

### Part 1: Vision Pipeline

- **New path:** `docs/try/first-project/part-1.md`
- **Source:** `docs/operate/hello-world/first-project/part-1.md` (move)
- **Action:** Move. Add alias. Update front matter.
- **Content changes:** None.

### Part 2: Data Capture

- **New path:** `docs/try/first-project/part-2.md`
- **Source:** `docs/operate/hello-world/first-project/part-2.md` (move)
- **Action:** Move. Add alias. Update front matter.
- **Content changes:** None.

### Part 3: Control Logic

- **New path:** `docs/try/first-project/part-3.md`
- **Source:** `docs/operate/hello-world/first-project/part-3.md` (move)
- **Action:** Move. Add alias. Update front matter.
- **Content changes:** None.

### Part 4: Deploy a Module

- **New path:** `docs/try/first-project/part-4.md`
- **Source:** `docs/operate/hello-world/first-project/part-4.md` (move)
- **Action:** Move. Add alias. Update front matter.
- **Content changes:** None.

### Part 5: Productize

- **New path:** `docs/try/first-project/part-5.md`
- **Source:** `docs/operate/hello-world/first-project/part-5.md` (move)
- **Action:** Move. Add alias. Update front matter.
- **Content changes:** None.

### Try section index

- **New path:** `docs/try/_index.md`
- **Action:** Create section landing page.

### First-project index

- **New path:** `docs/try/first-project/_index.md`
- **Source:** `docs/operate/hello-world/first-project/_index.md` (move)
- **Action:** Move. Add alias. Update front matter.
- **Content changes:** None.

---

## Section 3: Build — Foundation Howtos

Task-oriented howto pages. Each howto extracts procedural content from existing pages
and leaves clean reference material behind.

### Build section index

- **New path:** `docs/build/_index.md`
- **Action:** Create. Brief intro, link to foundation howtos.

### Foundation index

- **New path:** `docs/build/foundation/_index.md`
- **Action:** Create. List of foundation howtos with descriptions.

---

### Howto 1: Connect to Cloud

- **New path:** `docs/build/foundation/connect-to-cloud.md`

**Source pages:**

| Existing page | Role |
|--------------|------|
| `operate/install/setup.md` | PRIMARY SOURCE |
| `operate/reference/viam-server/_index.md` | LINK TO |
| `operate/control/api-keys.md` | LINK TO |

**New howto — what goes in:**

- Prerequisites (computer/SBC with supported OS, internet connection)
- Steps: create account → create machine → run install command → verify connection
- Verification: machine shows as online in the Viam app
- Links to reference for: SBC OS prep, viam-agent vs manual install, API keys

**Source page cleanup — `operate/install/setup.md`:**

- **Keep:** OS compatibility checks, SBC prep links, viam-agent vs manual details, Docker notes, platform-specific install commands
- **Remove:** The procedural walkthrough ("Create a Viam account... Create a new machine...") — that moves to the howto
- **Result:** Pure installation reference. "Here are the supported platforms, install methods, and configuration options."

**Code:** No. UI/CLI-driven task.

---

### Howto 2: Add a Camera

- **New path:** `docs/build/foundation/add-a-camera.md`

**Source pages:**

| Existing page | Role |
|--------------|------|
| `operate/reference/components/camera/_index.md` | PRIMARY SOURCE |
| `operate/reference/components/camera/webcam.md` | LINK TO |

**New howto — what goes in:**

- Prerequisites (machine connected to cloud, camera physically connected or available)
- Brief "which model do I need?" decision guide (webcam → `webcam`, IP camera → `ffmpeg`, etc.) linking to reference for each
- Steps: go to CONFIGURE → add camera component → select model → set attributes → save → test in UI
- Verification: view live feed in the test panel

**Source page cleanup — `operate/reference/components/camera/_index.md`:**

- **Keep:** Model list (auto-generated), API method table, troubleshooting section, common errors
- **Remove:** The "Configuration" procedural text ("Go to your machine's CONFIGURE page, and add a model...") — that moves to the howto. The "Next steps" section (the howto handles navigation).
- **Result:** Pure camera reference. "Here are the available models, the API, and troubleshooting."

**Code:** No. UI-driven configuration.

---

### Howto 3: Capture and Sync Data

- **New path:** `docs/build/foundation/capture-and-sync-data.md`

**Source pages:**

| Existing page | Role |
|--------------|------|
| `data-ai/capture-data/capture-sync.md` | PRIMARY SOURCE |
| `data-ai/capture-data/advanced/advanced-data-capture-sync.md` | LINK TO |
| `data-ai/capture-data/advanced/how-sync-works.md` | LINK TO |

**New howto — what goes in:**

- Prerequisites (machine connected to cloud, at least one resource configured)
- Steps: add data management service → enable capture on a resource → set frequency → save → verify data in the cloud
- Brief note on frequency tradeoffs (cost vs. coverage)
- Verification: data appears in the DATA tab
- Links to: advanced config, how sync works, supported resources list

**Source page cleanup — `data-ai/capture-data/capture-sync.md`:**

- **Keep:** "How data capture and data sync work" explanation diagram and text (this is reference/explanation content — useful context). Supported resources expandable list. Links to advanced config, conditional sync, retention policies.
- **Remove:** The step-by-step "Configure data capture and sync for individual resources" procedure — that moves to the howto. The "View captured data" procedure — that's part of the howto verification. The "Stop data capture or data sync" procedures — those move to the new "Stop or Disable Data Capture" howto. The "Next steps" section.
- **Result:** Reference page explaining how data capture works, what resources support it, and linking to configuration options. No procedures.

**Code:** No. UI-driven configuration.

---

### Howto 4: Stop or Disable Data Capture

- **New path:** `docs/build/foundation/stop-data-capture.md`

**Source pages:**

| Existing page | Role |
|--------------|------|
| `data-ai/capture-data/capture-sync.md` (stop/disable sections) | PRIMARY SOURCE |

**New howto — what goes in:**

- Three procedures, clearly separated:
  1. Stop capture for a specific resource (toggle method switch off)
  2. Stop capture for all resources (toggle Capturing switch off)
  3. Disable cloud sync (toggle Syncing switch off)
- Each: 2-3 steps with screenshots
- Links to: conditional sync (for more granular control), retention policies

**Source page cleanup:** Already covered in Howto 3 plan — the stop/disable sections get removed from `capture-sync.md`.

**Code:** No. UI toggles.

---

### Howto 5: Filter Data Before Sync

- **New path:** `docs/build/foundation/filter-data.md`

**Source pages:**

| Existing page | Role |
|--------------|------|
| `data-ai/capture-data/filter-before-sync.md` | PRIMARY SOURCE (ML filtering) |
| `data-ai/capture-data/conditional-sync.md` | PRIMARY SOURCE (conditional sync) |
| `data-ai/capture-data/capture-sync.md` | LINK TO (frequency-based) |

**New howto — what goes in:**

- Framing: "You're capturing more data than you need. Choose an approach."
- Decision guide: three approaches with when to use each
  1. Lower the capture frequency (simplest — adjust Hz in config)
  2. ML-based filtering (filtered-camera module — capture only images containing specific objects)
  3. Conditional sync (sync only during certain time windows or when conditions are met)
- For each approach: brief description, when to use, link to detailed implementation
- Cost impact framing

**Source page cleanup — `data-ai/capture-data/filter-before-sync.md`:**

- **Keep:** The full `filtered-camera` walkthrough (prerequisite, add ML model, add vision service, configure filtered camera, configure capture, verify). This is itself a focused howto. Also keep: attribute tables, configuration examples.
- **Remove:** The introductory explanation of what filtering is and why you'd do it — that framing moves to the decision guide howto. The "Stop data capture" section at the end.
- **Result:** Focused howto for ML-based image filtering specifically. No preamble about filtering concepts.

**Source page cleanup — `data-ai/capture-data/conditional-sync.md`:**

- **Keep:** The sensor implementation example, the `sync-at-time` module walkthrough, the data manager configuration, the test procedure. This is a focused howto.
- **Remove:** The introductory bullet list of examples ("Only sync when on WiFi", etc.) — that moves to the decision guide. The explanation of what conditional sync is.
- **Result:** Focused howto for conditional sync implementation. No preamble.

**Code:** No. This is a decision guide linking to existing implementation howtos.

---

### Howto 6: Configure Data Pipelines

- **New path:** `docs/build/foundation/configure-data-pipelines.md`

**Source pages:**

| Existing page | Role |
|--------------|------|
| `data-ai/data/data-pipelines.md` | PRIMARY SOURCE |
| `data-ai/data/hot-data-store.md` | LINK TO |
| `data-ai/data/query.md` | LINK TO |

**New howto — what goes in:**

- Prerequisites (captured sensor data syncing to cloud)
- One concrete end-to-end example: "You're capturing temperature readings every 5 seconds. Create a pipeline that computes hourly averages."
- Steps: define pipeline (name, schedule, MQL query) → create it → wait for execution → query results
- When to use pipelines vs. raw queries
- Links to: schedule format reference, query limitations, hot data store

**Source page cleanup — `data-ai/data/data-pipelines.md`:**

- **Keep:** Schedule format table, query limitations, common issues, performance considerations. Pipeline CRUD operations (list, enable, disable, delete) with their multi-language examples. Execution history viewing. All attribute/option documentation.
- **Remove:** The introductory explanation of what pipelines are and why you'd use them — that framing is briefer in the howto. The "Create a pipeline" procedural walkthrough — the howto covers this with a concrete example instead.
- **Result:** Pipeline reference. Schedule options, query constraints, lifecycle management operations, API examples for all CRUD operations.

**Code:** Yes — Python + Go. Pipelines are created and queried programmatically. One end-to-end example.

---

### Howto 7: Sync Data to Your Database

- **New path:** `docs/build/foundation/sync-to-your-database.md`

**Source pages:**

| Existing page | Role |
|--------------|------|
| `data-ai/capture-data/advanced/advanced-data-capture-sync.md` (MongoDB section) | PRIMARY SOURCE |
| `data-ai/data/export.md` | LINK TO |
| `dev/reference/apis/data-client.md` | LINK TO |

**New howto — what goes in:**

- Framing: "Viam as ingestion layer" — your machines capture data, Viam routes it to your infrastructure
- Two patterns with decision guide:
  1. Direct MongoDB capture (real-time, on the edge) — JSON config walkthrough
  2. Cloud export via CLI/API (batch, after sync) — link to export docs
- For MongoDB: steps to configure `mongo_capture_config` in the data management service
- Things to know (not warnings — just facts): write failures don't block capture, data may reach MongoDB before cloud, latency impact at high capture rates
- Links to: export docs, DataClient API

**Source page cleanup — `data-ai/capture-data/advanced/advanced-data-capture-sync.md`:**

- **Keep:** All attribute tables (data management service attributes, data capture attributes). Cloud data retention configuration. Sync optimization settings. Remote parts configuration with examples. All JSON configuration examples.
- **Remove:** The "Capture directly to your own MongoDB cluster" procedural section — that moves to the howto. Keep the `mongo_capture_config` attributes in the attribute table.
- **Result:** Pure configuration reference for advanced data capture and sync options.

**Code:** No. MongoDB capture is JSON configuration. Export is covered in existing docs.

---

### Howto 8: Start Writing Code

- **New path:** `docs/build/foundation/start-writing-code.md`

**Source pages:**

| Existing page | Role |
|--------------|------|
| `operate/modules/control-logic.md` | PRIMARY SOURCE |
| `operate/hello-world/first-project/part-3.md` | REFERENCE (Go patterns) |
| `operate/modules/deploy-module.md` | LINK TO |
| `docs-dev/planning/module-first-pattern.md` | REFERENCE (architecture) |

**New howto — what goes in:**

- Brief module-first pattern explanation: same code runs locally (CLI) or deployed (module)
- Prerequisites (machine connected to cloud, Viam CLI installed, Go or Python)
- Steps: generate module → write logic in DoCommand → test locally → verify
- Generic example that works with any component (not LED-specific, not inspection-specific)
- Both Python AND Go
- Links to: deploy-module for packaging/upload, module dependencies, SDK references

**Source page cleanup — `operate/modules/control-logic.md`:**

- **Keep:** The hot reloading instructions, local module setup, local model configuration, job scheduling section. These are operational reference for working with modules.
- **Remove:** The "Create a module with a generic component template" walkthrough — that moves to the howto. The "Program control logic in module" section — the howto covers this. The "Test the control logic" basic walkthrough.
- **Result:** Reference for module operational concerns: hot reloading, local testing setup, job scheduling. Not a tutorial or walkthrough.

**Code:** Yes — Python + Go. The task is literally writing code.

---

## Cross-cutting concerns

### Aliases

Every moved page needs an alias from its old path so existing links don't break.

### Internal links

After moving, we need to update internal links across the docs that point to the old paths. This is a separate pass after the scaffold is in place.

### Navigation

The new sections need to appear in the site navigation. This requires updating whatever Hugo config or layout controls the sidebar.

### Source page cleanup summary

| Source page | What gets removed | What remains |
|-------------|-------------------|-------------|
| `operate/install/setup.md` | Procedural walkthrough | Installation reference (platforms, methods, options) |
| `operate/reference/components/camera/_index.md` | Configuration procedure, next steps | Model list, API table, troubleshooting |
| `data-ai/capture-data/capture-sync.md` | All procedures (configure, view, stop) | How capture/sync works, supported resources, links |
| `data-ai/capture-data/filter-before-sync.md` | Intro explanation of filtering concept | Focused filtered-camera implementation howto |
| `data-ai/capture-data/conditional-sync.md` | Intro explanation of conditional sync concept | Focused conditional sync implementation howto |
| `data-ai/data/data-pipelines.md` | Intro explanation, create walkthrough | Schedule reference, query constraints, CRUD operations |
| `data-ai/capture-data/advanced/advanced-data-capture-sync.md` | MongoDB procedural section | Attribute tables, retention, sync optimization, remote parts |
| `operate/modules/control-logic.md` | Module creation + logic writing walkthrough | Hot reloading, local testing setup, job scheduling |
