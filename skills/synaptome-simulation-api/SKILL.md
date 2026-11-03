---
name: synaptome-simulation-api
description: Guide for running a single-neuron-with-synapses (synaptome) simulation via the OBI REST APIs. Covers generating the simulation campaign from an existing synaptome Circuit, finding the launchable config, and launching with the synaptome task type. Use when the user wants to simulate a synaptome (an ME-model with afferent synapses already placed) — not to build one (see build-synaptome) and not for a synapse-free ME-model simulation (see single-cell-simulation-api).
license: Apache-2.0
---

# Synaptome Simulation: API Flow

> **Generic skills this one builds on — follow them, don't re-derive:**
> - [[obi-sandbox-auth]] — how to obtain, exchange, and refresh OBI credentials, and the offline-access consent flow before long-running jobs. Never construct auth calls ad hoc; the "Authentication" section below assumes a token obtained that way.
> - [[chat-obi-sandbox-bridge]] — (Claude Chat / Claude Desktop) run heavy work on the OBI sandbox, and get result files (voltage traces, plots) back to the user from there.
> - [[obi-logbook]] — **mandatory.** Keep a scientific logbook for every simulation session. See the Logbook section below.
> - [[obi-links]] — every entity id, sandbox file, and launched campaign handed to the user goes out as a hyperlink, never as a bare UUID or path. See the Linking section below.
> - [[parameter-scan]] — the shared generate → find-child-config → estimate → launch mechanics, and the campaign-vs-config distinction.

## What this skill is (and is not)

A **synaptome** is a single postsynaptic ME-model with afferent synapses placed on its morphology, registered as a single-cell **Circuit** (`target_simulator = NEURON`). This skill **simulates** an already-built synaptome.

- To **build** the synaptome Circuit (place the synapses) → [[build-synaptome]]. That step comes first; it produces the Circuit this skill consumes.
- To simulate a **synapse-free** ME-model → [[single-cell-simulation-api]] (uses a different input entity and a different task type; see the note below).

Pipeline:

```
build-synaptome  →  synaptome Circuit (target_simulator = NEURON)
       ↓
synaptome-simulation-api  →  simulate that Circuit
```

## Authentication (all requests)

- `Authorization: Bearer <keycloak_token>` — Keycloak (realm: SBO)
- `virtual-lab-id: <uuid>` — HTTP header, from `$OBI_VLAB_ID` in the sandbox
- `project-id: <uuid>` — HTTP header, from `$OBI_PROJECT_ID` in the sandbox

Read both from the environment, never from memory: a tool call can be redirected to another project, and the environment follows while remembered ids do not. See [[virtual-lab-manager-api]].

Base URLs:
- Staging: `https://staging.cell-a.openbraininstitute.org`
- Production: `https://cell-a.openbraininstitute.org`

## Pre-requisite: Find the Synaptome Circuit

The synaptome Circuit comes from [[build-synaptome]]. Find it in EntityCore:

```
GET /api/entitycore/circuit?project_id=<project-id>
```

A synaptome Circuit has `has_electrical_cell_models = true`, `number_neurons = 1` (plus virtual presynaptic neurons), and `target_simulator = NEURON`.

## Step 1: Generate the Campaign

**Endpoint:** `POST /api/obi-one/generated/me-model-with-synapses-circuit-simulation-scan-config-generate-grid`

The config's `initialize.circuit` references the synaptome Circuit via `MEModelWithSynapsesCircuitFromID` — this is the key difference from a synapse-free ME-model simulation, which uses `MEModelFromID`. The rest of the scan-config body (info, initialize fields, stimuli, recordings) follows the same shape and field conventions as the plain ME-model simulation in [[single-cell-simulation-api]] (Part 3); reuse that structure, swapping only the circuit `type`.

Lists on `initialize` fields become scan dimensions — see [[parameter-scan]].

## Step 2: Find the Simulation(s)

The generate endpoint creates a SimulationCampaign plus one Simulation per scan coordinate. Find the launchable Simulation(s):

```
GET /api/entitycore/simulation?simulation_campaign_id=<campaign-id>
```

## Step 3: Launch

**Endpoint:** `POST /api/obi-one/declared/task/launch`

```json
{
  "task_type": "single_neuron_synaptome_simulation_execution",
  "config_id": "<simulation-uuid>"
}
```

**Response:** `{ task_type, config_id, activity_id, job_id }`.

> **Task type matters.** A synaptome simulation must launch with `single_neuron_synaptome_simulation_execution`. The synapse-free ME-model task type `single_neuron_simulation_execution` (from [[single-cell-simulation-api]]) maps to a different config schema, so using it for a synaptome run fails validation and wastes the launch. This is a platform usage detail, not a bug — the two task types are intentionally distinct.

**Auth note:** `task/launch` requires a stored refresh token in auth-manager (browser-based login). If it fails with `token_not_found` (common with MCP/API-only access), fall back to the bluecellulab sandbox path documented in [[single-cell-simulation-api]] (Part 5), staging the synaptome Circuit rather than a bare ME-model.

## Step 4: Monitor and Collect the Output

- **Poll:** `GET /api/obi-one/declared/task/<job-id>`
- **Stream (NDJSON):** `GET /api/obi-one/declared/task/<job-id>/stream`

## Logbook

Every synaptome simulation session is logged per [[obi-logbook]] — read it for the location, format, and content rules.

Record, as the session goes:

- **Objective** — the scientific question this simulation is meant to answer.
- **Synaptome used** — the postsynaptic ME-model (morphology + e-model), and the synapse configuration it carries: synapse types, counts, placement rule, release probability, and their sources. If this skill picked up a Circuit built in the same session, cross-reference the build logbook entry rather than repeating it.
- **Simulation protocol** — stimulus type and amplitude, injection/activation site, holding potential, duration, temperature, recorded variables and locations, number of repetitions.
- **Parameter sweeps** — what varies, over what range, and the physiological reason for that range.
- **Results** — firing rate, spike count, PSP/PSC amplitudes, latency, adaptation — all with units — plus figures and what each shows.
- **Interpretation and caveats** — what the behaviour means for the objective; limits of the model (single postsynaptic cell, virtual presynaptic input, no recurrent network, etc.).
- **Provenance table** — ME-model, synaptome Circuit, campaign, and simulation IDs.

Do **not** log endpoints, launch/polling mechanics, auth-manager issues, or workaround decisions — see the "What never goes in" list in [[obi-logbook]].

## Linking

**Link everything.** Per [[obi-links]], every entity, file, and campaign mentioned in a reply goes out as a hyperlink — a bare UUID or a bare path is a defect, not a style choice. `{domain}` is `staging.openbraininstitute.org` on staging, `www.openbraininstitute.org` on production, and must match the environment the calls ran against.

- **Entities** — `https://{domain}/app/entity/{id}`. Link the synaptome Circuit simulated, the postsynaptic ME-model, and the campaign and simulations produced. Link the ids in the provenance table too.
- **Sandbox files** — voltage traces and plots: take the `download_url` from `{obi}:get-sandbox-download-url` and swap `/files/` for `/lab/tree/`, keeping the rest of the path (`home/jovyan` included).
- **Launched jobs** — `https://{domain}/app/virtual-lab/{vlab_id}/{project_id}/workflows?tactivity=simulate&ttype=single_neuron_synaptome_simulation`, always with both parameters.
