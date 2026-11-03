---
name: single-cell-simulation-api
description: Guide for building ME (morpho-electric) models and running single-neuron simulations via the OBI REST APIs. Covers ME model creation (morphology + e-model compatibility check + registration), e-feature extraction, simulation campaigns, and real-time simulation via the small-scale-simulator. Also covers using bluecellulab in the sandbox as a workaround when task/launch has auth issues.
license: Apache-2.0
---

# Single Cell / ME Model Building & Simulation

> **Generic skills this one builds on — follow them, don't re-derive:**
> - [[obi-sandbox-auth]] — how to obtain, exchange, and refresh OBI credentials, and the offline-access consent flow. Note that the auth-manager issues described in Part 5 are governed by this skill; consult it before falling back to the bluecellulab workaround.
> - [[chat-obi-sandbox-bridge]] — (Claude Chat / Claude Desktop) run bluecellulab and other heavy work on the OBI sandbox, and get result files (voltage traces, plots) back to the user from there.
> - [[obi-logbook]] — **mandatory.** Keep a scientific logbook for every model-building or simulation session. See the Logbook section below.
> - [[obi-links]] — every entity id, sandbox file, and launched campaign handed to the user goes out as a hyperlink, never as a bare UUID or path. See the Linking section below.

## Overview

This skill covers the **synapse-free** single neuron (ME model). To simulate an ME-model **with synapses** (a synaptome), use [[synaptome-simulation-api]] — it takes a synaptome Circuit as input and launches with a different task type. Don't reuse the task types below for a synaptome run.

Single-cell workflows in OBI:

1. **ME Model Building** — Combine a morphology + e-model to create a single-neuron model (ME model)
2. **E-Feature Extraction** — Extract electrophysiological features from experimental traces
3. **ME Model Simulation (campaign)** — Generate simulation campaigns for parameter sweeps
4. **Real-time Simulation** — Interactive simulation via the small-scale-simulator
5. **Sandbox Simulation (bluecellulab)** — Direct simulation in the Python sandbox (workaround for auth-manager issues with task/launch)

## Authentication (all requests)

- `Authorization: Bearer <keycloak_token>` — Keycloak (realm: SBO)
- `virtual-lab-id: <uuid>` — HTTP header, from `$OBI_VLAB_ID` in the sandbox
- `project-id: <uuid>` — HTTP header, from `$OBI_PROJECT_ID` in the sandbox

Read both from the environment, never from memory: a tool call can be redirected to another project, and the environment follows while remembered ids do not. See [[virtual-lab-manager-api]].

Base URLs:
- Staging: `https://staging.cell-a.openbraininstitute.org`
- Production: `https://cell-a.openbraininstitute.org`

---

## Part 1: ME Model Building

### Concept

An ME model (single-neuron model) = **morphology** (cell shape) + **e-model** (electrical behaviour). The e-model must have an **e-type compatible with the morphology**. The platform enforces this via a compatibility check.

### Step 1: Find a Morphology

```
GET /api/entitycore/cell-morphology?project_id=<project-id>
```

Key fields: `id`, `name`, `brain_region`, `subject` (contains `species`, `strain`).

### Step 2: Find Compatible E-Models

E-models have an `etypes` field (list of electrical types like "cAC", "cNAC", "bNAC", etc.). The e-model must be compatible with the morphology's geometry.

```
GET /api/entitycore/emodel?project_id=<project-id>
```

Key fields: `id`, `name`, `brain_region`, `species`, `etypes`, `exemplar_morphology`.

**Compatibility guidance for the user:**
- The e-model's **species** should match the morphology's species
- The e-model's **brain region** should be compatible (same or parent region)
- The e-model's **e-type** determines the electrical behaviour class — select one appropriate for the target neuron type (e.g., `cAC` for continuous accommodating, `bNAC` for burst non-accommodating)
- Some e-models are blacklisted and cannot be combined with any morphology (legacy models with known issues)

### Step 3: Check Compatibility

Before creating the ME model, verify the morphology + e-model pair is viable:

**Endpoint:** `POST /api/small-scale-simulator/single-neuron/compatibility/run`

**Request body:**
```json
{
  "morphology_id": "<cell-morphology-uuid>",
  "emodel_id": "<emodel-uuid>"
}
```

**Response:**
```json
{
  "data": {
    "compatible": true,
    "morphology_id": "<uuid>",
    "emodel_id": "<uuid>",
    "error": null
  }
}
```

If `compatible: false`, the `error` field explains why (e.g., section type mismatch, missing mechanisms). **Do not proceed** if incompatible — advise the user to select a different e-model.

### Step 4: Create the ME Model

**Endpoint:** `POST /api/small-scale-simulator/single-neuron`

**Request body:**
```json
{
  "name": "My L5 TPC Model",
  "description": "L5 thick-tufted pyramidal cell model",
  "emodel_id": "<emodel-uuid>",
  "morphology_id": "<cell-morphology-uuid>",
  "brain_region_id": "<brain-region-uuid>",
  "species_id": "<species-uuid>",
  "strain_id": "<strain-uuid-or-null>"
}
```

- `brain_region_id`: typically from the morphology's brain region
- `species_id`: from the morphology's `subject.species.id`
- `strain_id`: from the morphology's `subject.strain.id` (nullable)

**Response:**
```json
{
  "data": {
    "id": "<new-memodel-uuid>",
    "name": "My L5 TPC Model",
    ...
  }
}
```

### E-Type Reference

Common electrical types:

| E-Type | Description |
|--------|-------------|
| `cAC` | Continuous accommodating |
| `cNAC` | Continuous non-accommodating |
| `bNAC` | Burst non-accommodating |
| `bAC` | Burst accommodating |
| `cSTUT` | Continuous stuttering |
| `bSTUT` | Burst stuttering |
| `dNAC` | Delayed non-accommodating |
| `dSTUT` | Delayed stuttering |

---

## Part 2: E-Feature Extraction (EModel Building Step 1)

### Concept

Extracts electrophysiological features from experimental recordings using BluePyEModel. Produces feature/protocol configurations for the optimisation stage.

### Pre-requisite

```
GET /api/entitycore/electrical-cell-recording?project_id=<project-id>
```

### Endpoint

`POST /api/obi-one/generated/e-model-e-feature-extraction-scan-config-generate-grid`

### Request Body

```json
{
  "type": "EModelEFeatureExtractionScanConfig",
  "info": {
    "type": "Info",
    "campaign_name": "EFeature Extraction",
    "campaign_description": "Extract features from patch-clamp recordings"
  },
  "initialize": {
    "type": "EModelEFeatureExtractionScanConfig.ExtractionInitialize",
    "electrical_cell_recording": [
      {
        "type": "ElectricalCellRecordingFromID",
        "id_str": "<electrical-cell-recording-uuid>"
      }
    ]
  },
  "efeatures_by_protocol": { ... },
  "settings": { ... }
}
```

### Entity Flow

Uses generic task-config entities:
- Campaign: `task-config` (type: `efeature_extraction__campaign`)
- Child config: `task-config` (type: `efeature_extraction__config`)

### Find child configs and launch

```
GET /api/entitycore/task-config?task_config_type=efeature_extraction__config&task_config_generator_id=<campaign-id>
```

```json
POST /api/obi-one/declared/task/launch
{
  "task_type": "efeature_extraction",
  "config_id": "<child-config-uuid>"
}
```

---

## Part 3: ME Model Simulation (Campaign-Based)

### Concept

Generate simulation campaigns for parameter sweeps. Creates SimulationCampaign + Simulation entities.

### Step 1: Generate Campaign

**Endpoint:** `POST /api/obi-one/generated/me-model-simulation-scan-config-generate-grid`

**Request body:**

```json
{
  "type": "MEModelSimulationScanConfig",
  "info": {
    "type": "Info",
    "campaign_name": "Single Neuron Simulation",
    "campaign_description": "Step current injection on L5_TPC"
  },
  "initialize": {
    "type": "MEModelSimulationScanConfig.Initialize",
    "circuit": {
      "type": "MEModelFromID",
      "id_str": "<memodel-uuid>"
    },
    "simulation_length": 1000.0,
    "v_init": -80.0,
    "random_seed": 1,
    "extracellular_calcium_concentration": 1.1
  },
  "stimuli": {
    "step_current": {
      "type": "<StimulusType>",
      ...
    }
  },
  "recordings": {
    "soma_voltage": {
      "type": "<RecordingType>",
      ...
    }
  },
  "morphology_locations": {},
  "neuronal_manipulations": {},
  "timestamps": {}
}
```

### Initialize Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `circuit` | `MEModelFromID` or list | required | ME model(s) to simulate |
| `simulation_length` | float or list | 1000.0 | Duration in ms — **min 1.0, max 12000.0** |
| `v_init` | float or list | -80.0 | Initial membrane potential (mV) |
| `random_seed` | int or list | 1 | Random seed |
| `extracellular_calcium_concentration` | float or list | 1.1 | Ca²⁺ (mM) |

### Step 2: Find Simulations

```
GET /api/entitycore/simulation?simulation_campaign_id=<campaign-id>
```

### Step 3: Launch

```json
POST /api/obi-one/declared/task/launch
{
  "task_type": "single_neuron_simulation_execution",
  "config_id": "<simulation-uuid>"
}
```

**Note:** This requires a stored refresh token in auth-manager (browser-based login). See Part 5 for workaround.

---

## Part 4: Real-Time Simulation (Small-Scale-Simulator)

For interactive exploration — no campaign needed, immediate results.

**Endpoint:** `POST /api/small-scale-simulator/single-neuron/simulation/run?model_id=<memodel-uuid>&realtime=True`

**Request body:**
```json
{
  "record_from": [
    {"section": "soma", "offset": 0.5}
  ],
  "conditions": {
    "celsius": 34.0,
    "v_init": -80.0,
    "hypamp": 0.0
  },
  "current_injection": {
    "inject_to": "soma",
    "stimulus": {
      "stimulus_type": "current_clamp",
      "stimulus_protocol": "step",
      "amplitudes": [0.1, 0.2, 0.3]
    }
  },
  "type": "current-injection",
  "duration": 1000.0
}
```

**Response:** Binary simulation results (streaming SONATA format).

---

## Part 5: Sandbox Simulation with bluecellulab (Workaround)

When `task/launch` fails with `token_not_found` (auth-manager doesn't have a refresh token for the session — common with MCP/API-only access), use bluecellulab directly in the Python sandbox.

### Via the OBI MCP sandbox (`execute-python` tool)

```python
from entitysdk.client import Client
from entitysdk.staging.memodel import stage_sonata_from_memodel
from entitysdk.models import MEModel
from bluecellulab import Cell, Simulation
from pathlib import Path
import numpy as np

# Setup
output_dir = Path("/tmp/memodel_sim")
output_dir.mkdir(exist_ok=True)

# Fetch and stage the ME model as a SONATA circuit
client = Client()  # uses the session token
memodel = client.get_entity(entity_id="<memodel-uuid>", entity_type=MEModel)
circuit_config_path = stage_sonata_from_memodel(
    client=client,
    memodel=memodel,
    output_dir=output_dir,
)

# Create the cell
# NOTE: Cell() does NOT take a circuit config or a node_id. Its real signature is
#   Cell(template_path, morphology_path, cell_id=None, record_dt=None,
#        template_format="v5", emodel_properties=None)
# To go from a staged SONATA directory to a cell, load it through CircuitSimulation
# instead of constructing Cell directly:
#
#   from bluecellulab.circuit_simulation import CircuitSimulation
#   sim = CircuitSimulation(<simulation_config>)
#   sim.instantiate_gids([(<population>, 0)], add_stimuli=False)
#   cell = sim.cells[(<population>, 0)]
#
# ⚠️ Unverified wiring: stage_sonata_from_memodel returns a *circuit* config, while
# CircuitSimulation expects a *simulation* config. Confirm which path the staged
# output feeds before relying on this — run it once in the sandbox and fix this
# snippet with what actually works.

# Add stimulus
cell.add_step(
    start_time=200.0,   # ms
    stop_time=800.0,    # ms
    level=0.2,          # nA
)

# Add recording
cell.add_voltage_recording(section=cell.soma, segx=0.5)

# Run simulation
sim = Simulation()
sim.add_cell(cell)
sim.run(1000.0, dt=0.025)  # 1000 ms, 0.025 ms timestep

# Get results
time = cell.get_time()
voltage = cell.get_voltage_recording(section=cell.soma, segx=0.5)

# Return results
result = {
    "time_ms": time.tolist()[:100],  # first 100 points as sample
    "voltage_mV": voltage.tolist()[:100],
    "total_points": len(time),
    "duration_ms": float(time[-1]),
    "spike_count": int(np.sum(np.diff((np.array(voltage) > -20).astype(int)) == 1)),
}
result
```

### Key bluecellulab API

Signatures below are from the bluecellulab source, not inferred.

| Method | Description |
|--------|-------------|
| `Cell(template_path, morphology_path, cell_id=None, record_dt=None, template_format="v5", emodel_properties=None)` | Construct a cell from a **hoc template + morphology file**. It takes neither a circuit config nor a `node_id`. |
| `CircuitSimulation(simulation_config, dt=None, record_dt=None, base_seed=None, …)` | Load a SONATA simulation; then `instantiate_gids(cells, …)` and read cells from `sim.cells`. This is the route for SONATA-based loading. |
| `cell.add_step(start_time, stop_time, level, section=None, segx=0.5)` | Current clamp step (nA) |
| `cell.add_ramp(start_time, stop_time, start_level, stop_level, section=None, segx=0.5)` | Ramp injection |
| `cell.add_voltage_recording(section, segx)` | Record voltage |
| `sim.run(duration, dt=0.025)` | Run simulation |
| `cell.get_time()` | Get time array |
| `cell.get_voltage_recording(section, segx)` | Get voltage trace |

### Staging the ME model

The key step is `stage_sonata_from_memodel` from `entitysdk.staging.memodel` — this downloads the morphology, e-model HOC, parameters, and ion channel mechanisms, assembles them into a SONATA circuit directory, and returns the `circuit_config.json` path. bluecellulab then loads from that config.

---

## When to Use What

| Use case | API/Tool | Notes |
|----------|----------|-------|
| Build an ME model | `POST /api/small-scale-simulator/single-neuron` | Check compatibility first |
| Quick interactive sim | `POST /api/small-scale-simulator/single-neuron/simulation/run` | Streaming results |
| Parameter sweep campaign | obi-one generate-grid + task/launch | Requires browser session for auth |
| Sim without browser auth | bluecellulab in sandbox | Use when task/launch gives token_not_found |
| E-feature extraction | obi-one generate-grid + task/launch | For building new e-models |

---

## ME Model Building Workflow Summary

```
1. User selects a morphology (cell-morphology entity)
   GET /api/entitycore/cell-morphology?project_id=...

2. User browses e-models (filter by compatible species/region/etype)
   GET /api/entitycore/emodel?project_id=...

3. Check compatibility
   POST /api/small-scale-simulator/single-neuron/compatibility/run
   → { compatible: true/false, error: "..." }

4. If compatible → create ME model
   POST /api/small-scale-simulator/single-neuron
   → { data: { id: "<new-memodel-uuid>" } }

5. Simulate the new ME model
   - Real-time: POST /api/small-scale-simulator/single-neuron/simulation/run
   - Campaign: obi-one generate-grid → task/launch
   - Sandbox: bluecellulab (see Part 5)
```

## Advising Users on E-Model Selection

When helping a user choose an e-model for their morphology:

1. **Match species** — The e-model's species must match the morphology's subject species
2. **Match brain region** — Same region or a compatible parent/child region
3. **Consider e-type** — The electrical behaviour type should match the expected physiology:
   - Pyramidal cells → typically `cAC` (continuous accommodating)
   - Interneurons → varies: `bNAC`, `cNAC`, `cSTUT`, etc.
   - If unsure, suggest running the compatibility check with a few candidates
4. **Always run the compatibility check** — Even if species/region match, the morphology geometry may be incompatible with the e-model's mechanism placement (e.g., missing apical dendrite for a model that expects one)
5. **Blacklisted e-models** — Some legacy e-models are known to be incompatible with all morphologies. If the compatibility check fails repeatedly, suggest trying a different e-type family.

---

## EntityCore Types Reference

| Entity | API Route | Description |
|--------|-----------|-------------|
| `memodel` | `/api/entitycore/memodel` | Combined morphology + electrical model |
| `emodel` | `/api/entitycore/emodel` | Electrical model (HOC + params + mechanisms) |
| `cell-morphology` | `/api/entitycore/cell-morphology` | Neuron morphology |
| `simulation-campaign` | `/api/entitycore/simulation-campaign` | Campaign entity |
| `simulation` | `/api/entitycore/simulation` | Individual simulation |
| `electrical-cell-recording` | `/api/entitycore/electrical-cell-recording` | Experimental ephys traces |

---

## Working directory

All files this workflow produces go under the conversation's topic directory on the persistent volume, per the convention in [[obi-logbook]] — one directory per scientific topic (`l5-pyramidal-me-model`, `ca1-int-rheobase`), created before any work starts, reused if it already exists:

```
/home/jovyan/<topic>/
├── logbook.md
├── morphologies/       ← downloaded morphologies
├── emodels/            ← e-models, hoc/mod files
├── me-model/           ← the assembled ME model, staged as SONATA for bluecellulab
├── results/            ← voltage traces, spike times; one subdir per condition
├── plots/
└── code/
```

`logbook.md`, `results/`, `plots/`, and `code/` are fixed by [[obi-logbook]]; the rest are this workflow's inputs. When running bluecellulab in the sandbox (Part 5), stage the circuit under `me-model/` rather than `/tmp` — `/tmp` is ephemeral overlay storage and will not survive.

## Logbook

Every ME-model-building, e-feature-extraction, or single-cell simulation session is logged per [[obi-logbook]] — read it for the location, format, and content rules.

Record, as the session goes:

- **Objective** — the scientific question this single cell is meant to answer.
- **Morphology chosen** — name, cell type (m-type), brain region, species/strain, reconstruction provenance, and why this morphology over the alternatives.
- **E-model chosen** — name, e-type, what firing behaviour it reproduces, and the reason it was paired with this morphology. If the compatibility check drove the choice, record the *biological* reason the e-type suits the m-type, not the check itself.
- **E-features** (when extracting) — which experimental traces were used, which features were extracted, and their values with units and spread across cells.
- **Simulation protocol** — stimulus type and amplitude, injection site, holding potential, duration, temperature, recorded variables and locations, number of repetitions.
- **Parameter sweeps** — what varies, over what range, and the physiological reason for that range.
- **Results** — firing rate, spike count, threshold current (rheobase), latency, adaptation, voltage amplitudes — all with units — plus figures and what each shows.
- **Interpretation and caveats** — what the cell's behaviour means for the objective; limits of the model (single cell in isolation, no network input, parameter fitted at a different temperature, etc.).
- **Provenance table** — morphology, e-model, ME-model, campaign and simulation IDs.

If work ran through bluecellulab in the sandbox rather than the campaign path (Part 5), that is a plumbing detail: the logbook records the same science either way, and the choice only appears as a caveat if it changed the simulation itself (e.g. a different solver setting or timestep).

Do **not** log endpoints, launch/polling mechanics, auth-manager issues, or the workaround decision itself — see the "What never goes in" list in [[obi-logbook]].

## Linking

**Link everything.** Per [[obi-links]], every entity, file, and campaign mentioned in a reply goes out as a hyperlink — a bare UUID or a bare path is a defect, not a style choice. `{domain}` is `staging.openbraininstitute.org` on staging, `www.openbraininstitute.org` on production, and must match the environment the calls ran against.

- **Entities** — `https://{domain}/app/entity/{id}`. Link the morphology and e-model chosen, the ME model created, the electrical recordings used for e-feature extraction, and the campaigns and simulations produced. Link the ids in the provenance table too.
- **Sandbox files** — voltage traces, plots, and staged models: take the `download_url` from `{obi}:get-sandbox-download-url` and swap `/files/` for `/lab/tree/`, keeping the rest of the path (`home/jovyan` included). This applies to bluecellulab output from Part 5 exactly as it does to campaign results.
- **Launched jobs** — `https://{domain}/app/virtual-lab/{vlab_id}/{project_id}/workflows?tactivity={activity}&ttype={type}`, always with both parameters:
  - ME model build → `tactivity=build&ttype=memodel`
  - ME model simulation campaign → `tactivity=simulate&ttype=single_neuron_simulation`
  - ME-model *circuit* simulation → `tactivity=simulate&ttype=me_model_circuit_simulation`

  (Synaptome build and synaptome simulation links live in [[build-synaptome]] and [[synaptome-simulation-api]].)

  Real-time small-scale-simulator runs (Part 4) and sandbox bluecellulab runs (Part 5) are not campaigns and have no activity page — link the entities and the output files instead.
