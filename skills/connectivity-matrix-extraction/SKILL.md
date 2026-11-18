---
name: connectivity-matrix-extraction
description: Guide for extracting a connectivity matrix from an edge population of a SONATA circuit in ConnectomeUtilities format — a sparse synapse-count matrix plus a node-attribute table — using the obi-one library in the OBI sandbox. This is a local (in-sandbox) analysis task, not a launch-system campaign. Use when the user wants a connectivity/connectome matrix, an adjacency or synapse-count matrix, or a conntility ConnectivityMatrix for downstream network analysis.
license: Apache-2.0
---

# Connectivity Matrix Extraction (local, in-sandbox)

> **Generic skills this one builds on — follow them, don't re-derive:**
> - [[obi-sandbox-auth]] — obtain, exchange, and refresh OBI credentials for reading the circuit from EntityCore. The examples assume a token obtained that way.
> - [[chat-obi-sandbox-bridge]] — (Claude Chat / Claude Desktop) run the extraction on the OBI sandbox and get the matrix file back to the user.
> - [[obi-logbook]] — **mandatory.** Keep a scientific logbook for every extraction session.
> - [[obi-links]] — every entity id and sandbox file goes out as a hyperlink.
> - [[obi-circuit-simulation]] — what SONATA node/edge populations are.

## Overview

Connectivity matrix extraction reads an **edge population** of a SONATA circuit and produces a **connectivity matrix in ConnectomeUtilities (`conntility`) format**: a sparse matrix whose entries are the **number of synapses per connection**, together with a dataframe of selected **node attributes** (positions, m-type, e-type, layer, synapse class, …).

**This is a local, in-sandbox task — not a launch-system campaign.** Unlike extraction/synaptome/simulation, `connectivity_matrix_extraction` has no `generate-grid` REST endpoint and no `/task/launch` path (its registration carries no config asset label). You run it through the **`obi_one` Python library inside the OBI sandbox** (`execute-python` / `execute-shell`), against a circuit fetched from EntityCore.

## Entity Model

```
Circuit (SONATA, with ≥1 edge population)   ← input, read from EntityCore
    │   obi_one ConnectivityMatrixExtractionTask (runs in the sandbox)
    ▼
connectivity_matrix.h5   ← output, ConnectomeUtilities format (sparse synapse-count matrix + node table)
matrix_config.json       ← optional, when with_matrix_config = true
```

No new EntityCore entity is registered by this task — the outputs are files in the sandbox. Register them as assets separately if they need to persist.

## Authentication

Reading the circuit from EntityCore uses the sandbox credentials:

- `Authorization: Bearer <keycloak_token>`, `virtual-lab-id` / `project-id` headers from `$OBI_VLAB_ID` / `$OBI_PROJECT_ID`.

Read from the environment, never from memory. See [[virtual-lab-manager-api]] and [[obi-sandbox-auth]].

## Pre-requisite: Find the Circuit and its Edge Population

```
GET /api/entitycore/circuit?project_id=<project-id>
```

A circuit may have several edge populations. If it has exactly one, extraction picks it automatically; if it has more than one, you **must** name the population — otherwise the task raises an error.

## Running the Extraction (in the sandbox)

Configure a `ConnectivityMatrixExtractionScanConfig` and execute its task via `obi_one`. Conceptually:

```python
from obi_one.scientific.tasks.connectivity_matrix_extraction.config import (
    ConnectivityMatrixExtractionScanConfig,
)

config = ConnectivityMatrixExtractionScanConfig(
    initialize=ConnectivityMatrixExtractionScanConfig.Initialize(
        circuit=circuit,                     # obi_one Circuit (from the EntityCore circuit)
        edge_population="default",           # optional; auto if the circuit has a single population
        node_attributes=("x", "y", "z", "mtype", "etype", "layer", "synapse_class"),
        with_matrix_config=False,
    ),
)
# run the single-config task, writing to an output directory in the sandbox
```

The extraction loads the circuit, selects the edge population, and builds a `conntility` `ConnectivityMatrix` where the aggregation is **synapse count per connection** (`agg_func=len` over any edge property). It writes `connectivity_matrix.h5`.

> Loading a large circuit and building its matrix easily exceeds the sandbox's ~30 s tool-call timeout — run it as a background process and poll, per the "Long-running work" section of [[virtual-lab-manager-api]]. The same applies to sweeps.

## Field Reference

### ConnectivityMatrixExtractionScanConfig.Initialize

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `circuit` | `Circuit` (or list) | required | The SONATA circuit to read connectivity from |
| `edge_population` | str \| None | `None` | Edge population name. `None` → auto-select the sole population; **required** if the circuit has more than one |
| `node_attributes` | tuple[str, …] \| None | `None` → defaults | Node properties to include in the matrix's node table |
| `with_matrix_config` | bool | `false` | Also write a `matrix_config.json` pointing at the extracted matrix (and nest the matrix under `<edge_population>/single/`) |

**Default node attributes** (when `node_attributes` is `None`): `x`, `y`, `z`, `mtype`, `etype`, `layer`, `synapse_class`.

### Outputs

| File | When | Contents |
|------|------|----------|
| `connectivity_matrix.h5` | always | ConnectomeUtilities matrix: sparse synapse-count matrix + node-attribute dataframe |
| `matrix_config.json` | `with_matrix_config = true` | `{ "<edge_population>": { "single": { "description": …, "path": … } } }` |

> The task refuses to overwrite: if `connectivity_matrix.h5` (or the config file) already exists in the output directory, it raises rather than clobbering. Use a fresh output subdirectory per run.

## Parameter sweeps

`ConnectivityMatrixExtractionScanConfig` is a scan config: passing lists (e.g. several circuits, or several edge populations) turns them into scan dimensions producing one matrix per coordinate. See [[parameter-scan]] for the mechanics — but note that because this task is local-only, you drive the scan through `obi_one` in the sandbox, not via `/task/launch`.

---

## Working directory

```
/home/jovyan/<topic>/
├── logbook.md
├── results/            ← connectivity_matrix.h5 (one subdir per run / parameter setting)
├── plots/              ← adjacency plots, degree distributions, connectivity summaries
└── code/               ← the extraction + analysis scripts
```

## Logbook

Every extraction session is logged per [[obi-logbook]].

Record, as the session goes:

- **Objective** — what network question the matrix serves (degree distributions, motifs, targeted connectivity between cell types).
- **Source circuit** — its provenance, brain region, scale, neuron/synapse/connection counts, and which edge population was analysed (and why, if several exist).
- **Matrix definition** — that entries are synapse counts per connection, and which node attributes were kept.
- **Findings** — the connectivity statistics that matter scientifically (mean in/out degree, connection probabilities between m-types, etc.), with units.
- **Caveats** — the edge population's meaning, any node attributes missing, boundary effects for extracted/partial circuits.

Do **not** log obi_one call signatures, file-exists handling, or auth — see [[obi-logbook]].

## Linking

**Link everything.** Per [[obi-links]], `{domain}` matches the environment the calls ran against.

- **Entities** — `https://{domain}/app/entity/{id}`. Link the source circuit. If the matrix is registered as an asset/entity, link that too.
- **Sandbox files** — the `connectivity_matrix.h5` and any plots: take the `download_url` from `{obi}:get-sandbox-download-url` and swap `/files/` for `/lab/tree/`, keeping the rest of the path. A `results/<run>/` directory link is often more useful than a single file link.
- **Jobs** — this task runs locally in the sandbox and launches no campaign, so there is no workflows activity page to link.
