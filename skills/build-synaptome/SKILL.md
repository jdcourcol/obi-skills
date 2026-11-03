---
name: build-synaptome
description: Guide for building a single-neuron synaptome — placing synaptic models onto the morphology of an existing ME-model — via the obi-one and EntityCore REST APIs. The synaptome is registered as a Circuit (single-cell, NEURON target) that can then be simulated. Use when the user wants to place afferent synapses on an ME-model, build a single-neuron synaptome, or prepare a synaptome for single-neuron-with-synapses simulation.
license: Apache-2.0
---

# Build Synaptome: End-to-End API Flow

> **Generic skills this one builds on — follow them, don't re-derive:**
> - [[obi-sandbox-auth]] — how to obtain, exchange, and refresh OBI credentials, and the offline-access consent flow before long-running jobs. Never construct auth calls ad hoc; the "Authentication" section below assumes a token obtained that way.
> - [[chat-obi-sandbox-bridge]] — (Claude Chat / Claude Desktop) run heavy work on the OBI sandbox, and get result files back to the user from there.
> - [[obi-logbook]] — **mandatory.** Keep a scientific logbook for every synaptome-building session. See the Logbook section below.
> - [[obi-links]] — every entity id, sandbox file, and launched campaign handed to the user goes out as a hyperlink, never as a bare UUID or path. See the Linking section below.
> - [[synaptome-simulation-api]] — where the resulting synaptome Circuit goes next: single-neuron-with-synapses simulation. Read it if the user wants to simulate what they build here. (For a synapse-free ME-model simulation instead, see [[single-cell-simulation-api]].)

## Overview

A synaptome is a **single postsynaptic ME-model with afferent synapses placed on its morphology**. Building one takes an existing ME-model (morphology + e-model) and attaches one or more *synapse groups* — each group is a synaptic physiology model placed at locations chosen by a placement strategy. The output is registered as a single-cell **Circuit** (`target_simulator = NEURON`) that is ready for single-neuron-with-synapses simulation.

This is a launch-system-backed task. The flow is the standard obi-one scan → estimate → launch pattern (see [[parameter-scan]] for the shared mechanics):

1. **Generate the campaign** → `POST` to obi-one, which creates task-config entities in EntityCore
2. **Find the child config ID** → `GET` EntityCore
3. **Estimate cost** (optional) → `POST` to obi-one with the *child* config ID
4. **Launch the job** → `POST` to obi-one with the *child* config ID → registers a Circuit

## Entity Model

```
MEModel (postsynaptic)   ← input, from EntityCore
    │
    ▼
circuit_single_build__campaign (task-config)      ← parent, grid-scan parameters
    │
    ├── circuit_single_build__config (task-config) ← child, one per grid coordinate, LAUNCHABLE
    │       └── task_config_generator_id → campaign ID
    │
    └── task-activity (circuit_single_build__config_generation)  ← links campaign → configs

on launch → task-activity (circuit_single_build__execution)
         → Circuit (single-cell synaptome, target_simulator = NEURON)   ← output
```

> **Note the task type is `circuit_single_build`, not `build_synaptome`.** The scan-config class is `MEModelSynapticModelPlacementScanConfig`; the frontend labels this workflow "ME-model Synapse Placement" / "Build synaptome" under the **Build** activity.

## Authentication (all requests)

- `Authorization: Bearer <keycloak_token>` — Keycloak (realm: SBO)
- `virtual-lab-id: <uuid>` — HTTP header, from `$OBI_VLAB_ID` in the sandbox
- `project-id: <uuid>` — HTTP header, from `$OBI_PROJECT_ID` in the sandbox

Read both from the environment, never from memory. See [[virtual-lab-manager-api]].

Base URLs:
- Staging: `https://staging.cell-a.openbraininstitute.org`
- Production: `https://cell-a.openbraininstitute.org`

## Pre-requisite: Find the Postsynaptic ME-model

```
GET /api/entitycore/memodel?project_id=<project-id>
```

An ME-model bundles a morphology and an e-model (`morphology_id`, `emodel_id`). Its morphology must have a `subject` — the build fails to register the Circuit without one.

## Step 1: Generate the Campaign

**Endpoint:** `POST /api/obi-one/generated/me-model-synaptic-model-placement-scan-config-generate-grid`

**Request body** (`MEModelSynapticModelPlacementScanConfig`):

```json
{
  "type": "MEModelSynapticModelPlacementScanConfig",
  "info": {
    "type": "Info",
    "campaign_name": "L5 PC synaptome",
    "campaign_description": "AMPA/NMDA synapses on apical dendrites"
  },
  "initialize": {
    "type": "MEModelSynapticModelPlacementScanConfig.Initialize",
    "me_model": { "type": "MEModelFromID", "id_str": "<memodel-uuid>" }
  },
  "synaptic_models": {
    "excitatory": { "type": "<SynapticModel>", "...": "..." }
  },
  "distributions": {},
  "morphology_locations": {
    "apical_random": { "type": "<MorphologyLocations>", "...": "..." }
  },
  "synapse_groups": {
    "exc_group": {
      "type": "SynapticModelPlacer",
      "synaptic_model": { "block_dict_name": "synaptic_models", "block_name": "excitatory" },
      "placement_strategy": { "block_dict_name": "morphology_locations", "block_name": "apical_random" }
    }
  }
}
```

The four dictionaries compose the synaptome:

| Dict | Block group | Role |
|------|-------------|------|
| `synaptic_models` | Synaptic physiology | Available synaptic physiology models |
| `distributions` | Synaptic physiology | Distributions used by the synaptic models |
| `morphology_locations` | Synapse groups | Reusable placement strategies (where synapses land on the morphology) |
| `synapse_groups` | Synapse groups | Each `SynapticModelPlacer` ties one synaptic model to one placement strategy |

**Placement strategy → presynaptic neurons:** the number of locations produced by a placement strategy equals the number of synapses. Each *location group* becomes a distinct presynaptic (virtual) neuron — so a grouped strategy (e.g. Random Grouped Morphology Locations) controls how many presynaptic neurons the synapses come from; a strategy without grouping places all synapses from a single presynaptic neuron.

### Response

`200 OK` with the **campaign ID** as a plain string. See [[parameter-scan]] for lists-become-dimensions and the campaign-vs-config distinction.

## Step 2: Find the Child Config ID

```
GET /api/entitycore/task-config?task_config_type=circuit_single_build__config&task_config_generator_id=<campaign-id>
```

## Step 3: Estimate Cost (optional)

**Endpoint:** `POST /api/obi-one/declared/task/estimate`

```json
{ "task_type": "circuit_single_build", "config_id": "<child-config-uuid>" }
```

## Step 4: Launch the Job

**Endpoint:** `POST /api/obi-one/declared/task/launch`

```json
{ "task_type": "circuit_single_build", "config_id": "<child-config-uuid>" }
```

**Response:** `{ task_type, config_id, activity_id, job_id }`.

## Step 5: Monitor and Collect the Output

- **Poll:** `GET /api/obi-one/declared/task/<job-id>`
- **Stream (NDJSON):** `GET /api/obi-one/declared/task/<job-id>/stream`

On success, the task registers a **Circuit** (single-cell synaptome). Find it via the execution activity's generated entity, or:

```
GET /api/entitycore/circuit?project_id=<project-id>
```

The synaptome Circuit has `has_electrical_cell_models = true`, `number_neurons = 1` (plus virtual presynaptic neurons), and `target_simulator = NEURON`.

## Field Reference

### MEModelSynapticModelPlacementScanConfig.Initialize

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `me_model` | `MEModelFromID` | ✓ | Postsynaptic ME-model providing the morphology + e-model |

### SynapticModelPlacer (each entry in `synapse_groups`)

| Field | Type | Description |
|-------|------|-------------|
| `synaptic_model` | reference | Points into the `synaptic_models` dict (`block_dict_name` + `block_name`) |
| `placement_strategy` | reference | Points into the `morphology_locations` dict |

## Critical: Campaign ID ≠ Launchable Config ID

The generate endpoint returns the **campaign** ID; `/task/estimate` and `/task/launch` need the **child config** ID from Step 2. Passing the campaign ID to launch raises a 500. See [[parameter-scan]].

---

## Working directory

All files go under the conversation's topic directory on the persistent volume, per [[obi-logbook]]:

```
/home/jovyan/<topic>/
├── logbook.md
├── models/             ← the postsynaptic ME-model pulled from EntityCore
├── results/            ← registered synaptome circuit(s); one subdir per parameter setting
├── plots/              ← synapse-placement figures
└── code/
```

## Logbook

Every synaptome-building session is logged per [[obi-logbook]] — read it for location, format, and content rules.

Record, as the session goes:

- **Objective** — what the synaptome is for (e.g. studying dendritic integration of a given input pathway) and why this ME-model.
- **Postsynaptic cell** — the ME-model: cell type (m-type/e-type), brain region, the morphology and e-model behind it.
- **Synaptic inputs** — for each synapse group: the presynaptic pathway it represents, the synaptic physiology model (receptor kinetics, conductance), how many synapses, and where on the morphology they land (which dendritic domains) and why.
- **Presynaptic structure** — how many virtual presynaptic neurons the placement strategy produced and the biological rationale.
- **Result** — the registered synaptome Circuit and its scientific description.
- **Caveats** — assumptions in synapse counts/locations, missing pathways, idealisations.

Do **not** log endpoints, campaign-vs-config ID mechanics, cost estimates, job polling, or auth — see the "What never goes in" list in [[obi-logbook]].

## Linking

**Link everything.** Per [[obi-links]], `{domain}` is `staging.openbraininstitute.org` on staging, `www.openbraininstitute.org` on production, matching the environment the calls ran against.

- **Entities** — `https://{domain}/app/entity/{id}`. Link the input ME-model and the output synaptome Circuit; link id cells in the provenance table.
- **Sandbox files** — placement figures and summaries: take the `download_url` from `{obi}:get-sandbox-download-url` and swap `/files/` for `/lab/tree/`, keeping the rest of the path.
- **Launched campaigns** — synaptome building lives under the **Build** activity: `https://{domain}/app/virtual-lab/{vlab_id}/{project_id}/workflows?tactivity=build&ttype=circuit_single_build_campaign`.
