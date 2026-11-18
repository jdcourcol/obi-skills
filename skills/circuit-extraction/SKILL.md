---
name: circuit-extraction
description: Guide for extracting a sub-circuit from a larger SONATA circuit via the obi-one and EntityCore REST APIs. A neuron set defines which cells to keep; the extracted circuit carries all morphologies, hoc files, and mod files needed to simulate it standalone, and is registered as a new Circuit derived from the parent. Use when the user wants to extract a microcircuit, cut out a sub-population, or produce a smaller simulatable circuit from a big one.
license: Apache-2.0
---

# Circuit Extraction: End-to-End API Flow

> **Generic skills this one builds on — follow them, don't re-derive:**
> - [[obi-sandbox-auth]] — obtain, exchange, and refresh OBI credentials, and the offline-access consent flow before long-running jobs. The "Authentication" section assumes a token obtained that way.
> - [[chat-obi-sandbox-bridge]] — (Claude Chat / Claude Desktop) run heavy work on the OBI sandbox and get result files back to the user.
> - [[obi-logbook]] — **mandatory.** Keep a scientific logbook for every extraction session.
> - [[obi-links]] — every entity id, sandbox file, and launched campaign goes out as a hyperlink.
> - [[obi-circuit-simulation]] — what a SONATA circuit is and where the extracted circuit goes next (simulation). Read it alongside this one.
> - [[parameter-scan]] — the shared scan → estimate → launch mechanics and the campaign-vs-config distinction.

## Overview

Circuit extraction produces a **sub-circuit** of an existing SONATA circuit, defined by a *neuron set* (which cells to keep). The output circuit contains all the morphologies, hoc files, and mod files required to simulate the extracted circuit on its own, and is registered as a new **Circuit** derived from the parent (with a `derivation` link back to it).

This is a launch-system-backed task following the standard scan → estimate → launch pattern:

1. **Generate the campaign** → `POST` to obi-one → creates task-config entities in EntityCore
2. **Find the child config ID** → `GET` EntityCore
3. **Estimate cost** (optional) → `POST` to obi-one with the child config ID
4. **Launch** → `POST` to obi-one with the child config ID → registers the sub-circuit

> In the web app this is the **Extract** activity ("Circuit extraction"); it is feature-flagged.

## Entity Model

```
Circuit (parent)   ← input, from EntityCore
    │
    ▼
circuit_extraction__campaign (task-config)      ← parent, grid-scan parameters
    │
    ├── circuit_extraction__config (task-config) ← child, one per grid coordinate, LAUNCHABLE
    │       └── task_config_generator_id → campaign ID
    │
    └── task-activity (circuit_extraction__config_generation)

on launch → task-activity (circuit_extraction__execution)
         → Circuit (sub-circuit; derivation → parent, root_circuit_id → topmost ancestor)   ← output
```

## Authentication (all requests)

- `Authorization: Bearer <keycloak_token>` — Keycloak (realm: SBO)
- `virtual-lab-id: <uuid>` — HTTP header, from `$OBI_VLAB_ID`
- `project-id: <uuid>` — HTTP header, from `$OBI_PROJECT_ID`

Read from the environment, never from memory. See [[virtual-lab-manager-api]].

Base URLs: `https://staging.cell-a.openbraininstitute.org` (staging) / `https://cell-a.openbraininstitute.org` (production).

## Pre-requisite: Find the Parent Circuit

```
GET /api/entitycore/circuit?project_id=<project-id>
```

Pick a circuit that has morphologies and electrical cell models (`has_morphologies`, `has_electrical_cell_models`) if the extracted circuit is to be biophysically simulated.

> **Do not extract from these legacy circuits.** Two circuits are legacy builds that will most likely fail extraction. The legacy IDs below are **stable across all environments** (staging and production). Match by ID, not by name.
>
> | Name | Legacy ID (exclude) |
> |------|---------------------|
> | `nbS1-O1` | `96203ca0-8fbf-4e3a-9c9e-b23309020109` |
> | `nbS1-HEX0` | `8956d530-f9b0-4b6c-a660-f3cd7074ac55` |
>
> A newer, working build of the *same* circuit exists under a **different ID that varies by environment** — so there is no fixed replacement ID to hard-code; resolve it at runtime by name. When choosing or listing a parent circuit:
>
> - Filter these two IDs out of the candidates.
> - If the user asks for `nbS1-O1` or `nbS1-HEX0`, select the circuit with that name whose ID is **not** the legacy one above (the newer build).
> - If the only circuit matching the name is the legacy one (no replacement found in this environment), do not proceed — flag it to the user.

## Step 1: Generate the Campaign

**Endpoint:** `POST /api/obi-one/generated/circuit-extraction-scan-config-generate-grid`

**Request body** (`CircuitExtractionScanConfig`):

```json
{
  "type": "CircuitExtractionScanConfig",
  "info": {
    "type": "Info",
    "campaign_name": "L4 microcircuit",
    "campaign_description": "Extract layer-4 excitatory sub-circuit"
  },
  "initialize": {
    "type": "CircuitExtractionScanConfig.Initialize",
    "circuit": { "type": "CircuitFromID", "id_str": "<parent-circuit-uuid>" }
  },
  "neuron_sets": {
    "l4_exc": { "type": "<NeuronSet>", "...": "..." }
  }
}
```

The **neuron set** is what defines the extraction. If you omit one, the config defaults to `AllBiophysicalNeurons` ("Default: All Biophysical Neurons") — the whole biophysical population — or `AllPointNeurons` for point-neuron circuits. Provide a specific neuron set (by property, node-set name, id list, etc.) to cut out a genuine sub-population.

### Response

`200 OK` with the **campaign ID** as a plain string. Pass lists to sweep (e.g. several neuron sets) — see [[parameter-scan]].

## Step 2: Find the Child Config ID

```
GET /api/entitycore/task-config?task_config_type=circuit_extraction__config&task_config_generator_id=<campaign-id>
```

## Step 3: Estimate Cost (optional)

`POST /api/obi-one/declared/task/estimate` with `{ "task_type": "circuit_extraction", "config_id": "<child-config-uuid>" }`.

## Step 4: Launch the Job

`POST /api/obi-one/declared/task/launch` with `{ "task_type": "circuit_extraction", "config_id": "<child-config-uuid>" }` → `{ task_type, config_id, activity_id, job_id }`.

## Step 5: Monitor and Collect

- **Poll:** `GET /api/obi-one/declared/task/<job-id>`
- **Stream:** `GET /api/obi-one/declared/task/<job-id>/stream`

On success a new **Circuit** is registered. It links back to the parent via a `derivation` (`root_circuit_id` points to the topmost ancestor, which equals the parent only if the parent is itself a root), and reports its own `number_neurons`, `number_synapses`, `number_connections`, and `scale`.

## Field Reference

### CircuitExtractionScanConfig.Initialize

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `circuit` | `CircuitFromID` | ✓ | Parent SONATA circuit to extract from |

### neuron_sets

Dictionary of neuron-set blocks. Each defines a criterion for which cells to keep (by node-set name, property query, id list, or a combined set). Defaults to all biophysical (or all point) neurons if none is given.

## Critical: Campaign ID ≠ Launchable Config ID

The generate endpoint returns the campaign ID; estimate/launch need the child config ID from Step 2. See [[parameter-scan]].

---

## Working directory

```
/home/jovyan/<topic>/
├── logbook.md
├── results/            ← extracted sub-circuit(s); one subdir per neuron set / parameter setting
├── plots/              ← population / connectivity summaries of the sub-circuit
└── code/
```

## Logbook

Every extraction session is logged per [[obi-logbook]].

Record, as the session goes:

- **Objective** — what the sub-circuit is for and why this parent circuit.
- **Parent circuit** — its provenance: source model, brain region, build category, scale, neuron and synapse counts.
- **Selection criterion** — the neuron set in scientific terms: which cell types / layers / region the extraction keeps, and the reasoning. This is the heart of the record.
- **Result** — the extracted circuit's neuron/synapse/connection counts and scale, and how it relates biologically to the parent.
- **Caveats** — boundary effects (afferents from cut cells become virtual/severed), altered connectivity statistics, any population that was dropped.

Do **not** log endpoints, campaign-vs-config mechanics, cost estimates, job polling, or auth — see [[obi-logbook]].

## Linking

**Link everything.** Per [[obi-links]], `{domain}` matches the environment the calls ran against (`staging.openbraininstitute.org` / `www.openbraininstitute.org`).

- **Entities** — `https://{domain}/app/entity/{id}`. Link the parent circuit and the extracted sub-circuit; link id cells in the provenance table (parent in → sub-circuit out).
- **Sandbox files** — connectivity/population plots: `download_url` from `{obi}:get-sandbox-download-url` with `/files/` → `/lab/tree/`.
- **Launched campaigns** — extraction lives under the **Extract** activity: `https://{domain}/app/virtual-lab/{vlab_id}/{project_id}/workflows?tactivity=extract&ttype=circuit_extraction_campaign`.
