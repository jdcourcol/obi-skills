---
name: obi-circuit-simulation
description: Client-agnostic technical reference for the OBI/BBP (Open Brain Institute / Blue Brain Project) circuit and simulation stack — what a SONATA circuit is, how to find/stage/download one from EntityCore (prefer staging/symlinking over downloading, especially for large entities like circuits), how to configure and run a simulation via the obi-one API, and what simulation outputs look like. Invoke whenever a task touches circuit acquisition (staging or download) or configuring/running/polling an obi-one simulation — regardless of the scientific question behind it or which client/agent surface is in use. This is platform mechanics, not scientific workflow guidance; for disease-modeling process see [[disease-modeling]].
license: Apache-2.0
---

# OBI Circuit & Simulation — Technical Reference

This skill describes the OBI/BBP platform mechanics: what a circuit is, how to get one, simulate it, and read the results (modifying and registering a circuit: see [[data-integration-api]]). It says nothing about *why* you'd change a given parameter — that's a scientific decision made elsewhere (e.g. by [[disease-modeling]] or directly by the user).

> **Client-agnostic.** The Python/HTTP snippets below work from any environment with network access to the OBI APIs and appropriate credentials — an OBI sandbox Python execution tool, a local script, another agent surface. In **Claude Chat**, this compute runs via the OBI MCP connector's `execute-python`/`execute-shell` tools; getting results back to the user is a separate concern, handled by [[chat-obi-sandbox-bridge]]. Throughout this doc, `{obi}` stands for whatever that connector is named when one is in use (e.g. `{obi}:execute-python`); when no such connector exists, run the same code directly.

> **Generic skills this one builds on — follow them, don't re-derive:**
> - [[obi-sandbox-auth]] — how to obtain the credentials every snippet here assumes. In particular, **a Neurodamus/CoreNeuron simulation of more than 20 neurons requires the offline-access consent flow before launch** — obi-one routes anything above that scale to cluster resources automatically. Run that check before configuring the campaign, not after. (Brian2 and inait simulations never need it.)
> - [[chat-obi-sandbox-bridge]] — (Claude Chat / Claude Desktop) getting circuits, result files, and plots back to the user.
> - [[obi-logbook]] — **mandatory** whenever this skill is used in service of a scientific question. See the Logbook section below.
> - [[obi-links]] — every entity id, sandbox file, and launched campaign handed to the user goes out as a hyperlink, never as a bare UUID or path. The "Frontend links" section below is its circuit/simulation specialisation.

> **Library docs:** Use the **`context7`** MCP server (when available) for up-to-date API documentation on `entitysdk`, `obi-one`, `bluecellulab`, `bluepyopt`, etc. Call `context7:resolve-library-id("<library>")` then `context7:query-docs(...)`.

> **OpenAPI specs — consult before guessing.** Both obi-one and EntityCore expose a machine-readable OpenAPI spec. Before constructing a request body for an unfamiliar endpoint, or after any request fails with an unhelpful error (422, 500, wrong param names), fetch the relevant spec and grep it. This eliminates trial-and-error on query params, schema type names, and request body shapes.
> ```python
> import requests, os, json
> env = os.environ.get("OBI_ENVIRONMENT", "staging")
> prefix = "staging." if env == "staging" else ""
> specs = {
>     "entitycore": f"https://{prefix}openbraininstitute.org/api/entitycore/openapi.json",
>     "obi-one":    f"https://{prefix}openbraininstitute.org/api/obi-one/openapi.json",
> }
> for name, url in specs.items():
>     r = requests.get(url)
>     path = f"/tmp/{name}-openapi.json"
>     with open(path, "w") as f:
>         json.dump(r.json(), f)
>     print(f"{name}: {len(r.json().get('paths', {}))} paths → {path}")
> ```
> Then grep locally: `grep -i "neuron_set\|simulation_id\|ConstantCurrent" /tmp/obi-one-openapi.json | head -40`
> Useful patterns: `components.schemas` for type names and field nullability; endpoint `.parameters` for query param names; `.requestBody` for request shape.

---

## What a circuit is

A **circuit** in this stack is a [SONATA](https://sonata-extension.readthedocs.io/) (Scalable Open Network Architecture) format description of a network of biophysically detailed neuron models — node populations (cell positions, morphologies, biophysics) and edge populations (synaptic connectivity), plus simulation-ready config. See the [SONATA spec](https://sonata-extension.readthedocs.io/) for the on-disk format details, and the [SONATA paper](https://doi.org/10.1371/journal.pcbi.1007696) for design rationale (binary HDF5 for large per-node/per-edge data, human-readable CSV/JSON for shared/type-level attributes and config; populations exist so a node/edge set can be reused across multiple circuit configs without duplication — e.g. simulate region V1 alone, V2 alone, or V1+V2 together from the same underlying files).

Two disconnected "worlds" exist in this platform, and it matters which one you're in:

- **`Circuit` entities** — a full or partial network (any brain region/cell composition; pick one that fits the task at hand rather than defaulting to a familiar example). Simulated via `CircuitSimulationScanConfig`. Each node's morphology+biophysics pairing lives only inside the circuit's SONATA files (see below) — it is not a queryable entity.
- **`MEModel` (morpho-electrical model) entities** — a single cell's morphology + electrical model, tracked as its own row in EntityCore. Simulated via a separate `MEModelSimulationScanConfig`.

**These two worlds have no automatic link.** `Circuit` (in `entitycore/app/db/model.py`) has no foreign key to `MEModel`, and only single-cell-scoped entities (`SingleNeuronSimulation`, `SingleNeuronSynaptome`, `SingleNeuronSynaptomeSimulation`) reference `me_model_id`. `obi_one.utils` only ships `register_circuit`, not `register_emodel` — if you need the emodel/single-cell path, you must use `entitysdk` directly (manual entity creation); this is a known platform limitation as of 2026-07. This reference covers both paths.

### SONATA file layout, concretely

A circuit's `circuit_config.json` never embeds data directly — it's a manifest of references, resolved through a chain of separate files:

```
circuit_config.json
├── components.biophysical_neuron_models_dir  →  directory of .hoc files
├── components.morphologies_dir                →  directory of .swc/.asc/.h5 files
├── node_sets_file                              →  node_sets.json (named neuron/compartment groupings)
└── networks
    ├── nodes: [{ nodes_file: "nodes.h5", populations: {name: {type: "biophysical"|"virtual"|...}} }, ...]
    └── edges: [{ edges_file: "edges.h5", populations: {name: {type: "chemical"|"electrical"}} }, ...]
```

- `nodes`/`edges` are **arrays** — a circuit's full node/edge set is the *union* over every listed file, and each file can itself hold multiple named **populations**. This exists for reuse/composability (see rationale above), not because one file can't hold everything.
- A **population** is a storage-level partition inside the H5 file — its own contiguous node-ID space (0..size) and a fixed property schema. It is *not* the same thing as a **node set** (`node_sets.json`) — a node set is an arbitrary, named *query/filter* (e.g. `"Default: All Biophysical Neurons"`, or an mtype filter), evaluated against one or more populations. Simulation `inputs`/`reports` target **node sets**, never populations directly.
- Per-node columns of interest (verified by inspecting a real circuit's `nodes.h5` — inspect any given circuit's own file rather than assuming, since exact columns can vary): `x`,`y`,`z` (position) + `orientation_w/x/y/z` (quaternion — not always simple rotation angles; e.g. `orientation_w=0.986, x=-0.083, y=0.0, z=-0.146` was observed on one circuit), `mtype`,`etype`,`layer`,`region`,`synapse_class`,`morph_class` (classification), `model_type` (`biophysical`/`virtual`/...), `morphology` and `model_template` (**references, not paths** — resolved against the `components` dirs above), `me_combo` (legacy string identifying the morphology+emodel combo, predecessor to today's `MEModel` entity), and a `dynamics_params/` subgroup with per-node overrides like `holding_current`/`threshold_current` (used by BlueCelluLab's v6 HOC templates, which require these `EmodelProperties` explicitly rather than reading them off the HOC file).
- Categorical columns (`mtype`, `etype`, `layer`, `region`, ...) are stored as **integer indices into a `@library` sub-group**, not raw strings per row — e.g. `nodes/<population>/0/mtype` is a small-int array, and the actual string values live once in `nodes/<population>/0/@library/mtype`. This is the paper's "shared/type-level attributes stored once" efficiency principle applied inside the H5 file itself, not just via separate CSV type-files.
- Resolution rules (from `bluepysnap`): `morphology` name → `{morphologies_dir}/{name}.{ext}` (non-SWC formats route through a separate `alternate_morphologies` dir/container instead). `model_template` is `"<schema>:<resource>"` (e.g. real example: `"hoc:CA1_pyr_cACpyr_oh140807_A0_idF_2019030511545"`) → `{biophysical_neuron_models_dir}/{resource}.{schema}`.
- For a single ME-model's SONATA representation: the node file has exactly one row, the edge file is empty/absent — same format, degenerate case.
- Real circuits commonly mix a `biophysical` population (the actual simulated cells) with one or more `virtual` populations (external spike-source input, e.g. a projection from another region) — each in its own nodes.h5/edges.h5, which is exactly the reuse/composability pattern from the SONATA paper: the intrinsic circuit can be simulated with different external input populations swapped in without touching its own files. Population names are circuit-specific — always inspect a given circuit's own file rather than assuming a name.
- `.mod` mechanism files are **not referenced anywhere in the circuit_config.json chain** — but a downloaded circuit package does ship a `mod/` directory of them anyway (for portability/compilation), separate from `components/`. They must still be pre-compiled (`nrnivmodl`) and loaded into NEURON's shared mechanism registry ahead of time; HOC templates call them by their `nmodl_suffix` name only, trusting they're already loaded — nothing in the circuit/simulation config points at them by path.

The **simulation config** is a separate JSON that *references* a circuit config (via `network`, a file path — not embedded) and adds the dynamic/experiment layer: `run` (dt, tstop, random_seed), `conditions` (v_init, spike_location), `inputs` (stimuli, keyed by name, each with a `node_set` target + `module`/`input_type`), `reports` (recordings, keyed by name — each report gets its own output file named after its key), `output` (only controls `output_dir`/`spikes_file`/optional `log_file` — not per-report filenames).

---

## EntityCore entity map (single-cell + circuit pipeline)

Confirmed from `entitycore`'s `app/db/model.py` (relationships) and `app/db/types.py` (asset labels). Useful for constructing correct queries/FKs instead of guessing.

**Model-definition entities:**

| Entity | Key FKs | Assets |
|---|---|---|
| `CellMorphology` | — | `morphology`: `.asc`/`.swc`/`.h5` |
| `IonChannelModel` | — | `neuron_mechanisms`: `.mod` (channel kinetics); figures/thumbnails |
| `EModel` | `exemplar_morphology_id → CellMorphology`; M2M `ion_channel_models → IonChannelModel[]` (via `ion_channel_model__emodel`) | `emodel_optimization_output`: JSON (fitted params + scores — **source of truth**); `neuron_hoc`: `.hoc` (**derived, runnable artifact**, exported from the JSON via templates — regenerate the HOC from JSON, don't hand-edit HOC to change the model) |
| `MEModel` | `morphology_id → CellMorphology`; `emodel_id → EModel` | — (pure pairing entity) |
| `MEModelCalibrationResult` | `calibrated_entity_id → MEModel` | holding current / threshold calibration |

**Single-cell simulation entities** (all `me_model_id → MEModel` — this is the "MEModel world"):
`SingleNeuronSimulation`, `SingleNeuronSynaptome` (ME-model + synthetic synapses), `SingleNeuronSynaptomeSimulation`.

**Circuit/campaign entities** (the "Circuit world" — no link to `MEModel`, see above):

| Entity | Kind | Notes |
|---|---|---|
| `Circuit` | Entity | `root_circuit_id → Circuit` (self-referential: the **topmost** ancestor in the derivation hierarchy, not the direct predecessor — the direct **parent** is a `Derivation` link, `used` = parent, `generated` = this circuit; when registering a derived circuit pass only the parent and the root is derived from it, see [[data-integration-api]]); `atlas_id → BrainAtlas`; fields: `build_category`, `scale`, `has_morphologies`/`has_point_neurons`/`has_electrical_cell_models`/`has_spines` (bools), `number_neurons`, `number_synapses`, `number_connections`. Real assets (verified on a live circuit): `circuit.gz` (single archive — the **download fallback**), `sonata_circuit` (directory asset — **stage this with `stage_circuit`, never download directly**: over raw HTTP it's an unbounded S3 prefix that times out, but staging resolves it as symlinks from the mounted store at any size), plus visualization extras (`main.png`, `circuit_visualization.webp`, `network_stats_*.webp`, `node_stats.webp`, `circuit_connectivity_matrices` (directory)). No FK to `MEModel` (see above) — a circuit's per-neuron biophysics lives only in its SONATA files. |
| `SimulationCampaign` | Entity | overall campaign spec/config asset |
| `SimulationGeneration` | Activity (not Entity) | provenance: "the config-generation process ran" |
| `Simulation` | Entity | one grid point's SONATA simulation config; `simulation_campaign_id → SimulationCampaign`; carries `scan_parameters` (JSON, e.g. which amplitude this one is) |
| `SimulationExecution` | Activity (not Entity) | provenance: run status/timing, polled via `used__id` |
| `SimulationResult` | Entity | `simulation_id → Simulation`; the actual output data (see below) |

`generate-grid` creates: 1× `SimulationCampaign` + 1× `SimulationGeneration` (activity) + N× `Simulation` (one per grid point) — **pure config generation, nothing has run yet.**
`run-batch` creates: 1× `SimulationExecution` (activity) per simulation, and on completion, 1× `SimulationResult` per execution (found via the execution's `generated` field).

---

## Finding and acquiring a circuit from EntityCore

> **Stage, don't download — especially for large entities.** When a `LocalAssetStore` is available (always the case in the OBI sandbox, where the entitycore S3 bucket is mounted at `/data`), **prefer `entitysdk`'s `stage_circuit` over downloading `circuit.gz`.** Staging fetches the `sonata_circuit` directory with strategy `link_or_download`: it checks the local mount first and **symlinks** the already-present files instead of transferring any bytes. For a circuit this is the difference between an instant filesystem `symlink` and copying/transferring gigabytes — the cost of staging is independent of circuit size, so the larger the entity, the more decisive the win. Download `circuit.gz` only as a fallback when **no** local store is mounted (e.g. a local script with network-only access). The same rule applies to every large directory-backed entity (circuits, staged simulations, SONATA-from-ME-model) — reach for the matching `stage_*` helper before any `download_*`/`requests.get(.../download)` path. See [Staging a circuit](#staging-a-circuit-preferred) below and the "Decision rule — stage over download" note in [[data-integration-api]] ("Registering a (modified) circuit to EntityCore").

### List/filter circuits

Filter server-side by name pattern and size rather than pulling everything and filtering client-side. The `size` field in asset metadata is unreliable (`-1` or `0` for many circuits even when a real file exists) — don't use it to pre-screen; sort/filter by `number_neurons` instead.

```python
import os, requests

token = os.environ.get("OBI_ACCESS_TOKEN", "")
vlab = os.environ.get("OBI_VLAB_ID", "")
project = os.environ.get("OBI_PROJECT_ID", "")
headers = {
    "Authorization": f"Bearer {token}",
    "virtual-lab-id": vlab,
    "project-id": project,
}
env = os.environ.get("OBI_ENVIRONMENT", "staging")
base = f"https://{'staging.' if env == 'staging' else ''}openbraininstitute.org/api/entitycore"

# Filter server-side for what THIS task actually needs — brain region, scale, name pattern —
# don't default to a specific circuit name/family just because it's familiar or was used before.
# Pick filters from the user's request and/or a quick browse of what's available; number_neurons
# is a reasonable knob for "small enough to iterate quickly" but should stay in the low tens, not a fixed circuit.
resp = requests.get(f"{base}/circuit", headers=headers, params={
    "name__ilike": "%<PATTERN_RELEVANT_TO_THIS_TASK>%",  # e.g. a brain region or circuit family from the user's ask
    "number_neurons__lte": 20,
    "page_size": 20,
})
circuits = resp.json().get("data", [])
for c in circuits:
    gz = next((a for a in c.get("assets", []) if a.get("path") == "circuit.gz" and not a.get("is_directory")), None)
    if gz:
        print(f"{c['name']}  neurons={c.get('number_neurons')}  circuit_id={c['id']}  asset_id={gz['id']}")
```

### Staging a circuit (preferred)

This is the default way to materialise a circuit on the OBI sandbox. `stage_circuit` resolves the `sonata_circuit` directory asset and writes a ready-to-use circuit tree (returning the `circuit_config.json` path) **without downloading** when a `LocalAssetStore` is configured — it symlinks each file from the mounted `/data` bucket. That sidesteps the exact problem that makes the raw directory asset unusable over HTTP (an unbounded S3 prefix that times out): staging never enumerates it over the network, it stats and links the local mount. Cost is independent of circuit size.

```python
import os
from pathlib import Path
import entitysdk
from entitysdk.client import Client
from entitysdk.token_manager import TokenFromEnv
from entitysdk.common import ProjectContext
from entitysdk import models
from entitysdk.staging import stage_circuit

client = Client(
    token_manager=TokenFromEnv("OBI_ACCESS_TOKEN"),
    project_context=ProjectContext(
        virtual_lab_id=os.environ["OBI_VLAB_ID"],
        project_id=os.environ["OBI_PROJECT_ID"],
    ),
    environment=os.environ.get("OBI_ENVIRONMENT", "staging"),
    # The mounted entitycore bucket — this is what turns staging into a symlink
    # instead of a download. Without it, stage_circuit still works but falls back
    # to a real download (strategy link_or_download).
    local_store=entitysdk.LocalAssetStore(prefix="/data"),
)

circuit = client.get_entity(entity_id="<CIRCUIT_ID>", entity_type=models.Circuit)
circuit_config_path = stage_circuit(
    client,
    model=circuit,
    output_dir=Path("<LOCAL_DIR>/circuit"),
)
# circuit_config_path -> <LOCAL_DIR>/circuit/circuit_config.json, ready for bluecellulab / Neurodamus.
```

Inspect the staged tree before assuming file layout — it varies by circuit. For single-neuron circuits, the key file is typically a `.hoc` biophysical model under `components/biophysical_neuron_models/`. Because staged files may be symlinks into the read-only mount, **copy any file you intend to edit** before modifying it (see "Modifying a circuit" in [[data-integration-api]]).

### Download `circuit.gz` — fallback when no local store is mounted

Use this only when staging is not available (e.g. a local script with network-only access and no mounted `/data` bucket). **If you do download, always take the `circuit.gz` asset — never the `sonata_circuit` directory asset.** The directory is an unbounded S3 prefix that will time out; `circuit.gz` is a single archived file. The download endpoint returns a 307 redirect to a signed S3 URL — follow it (most HTTP clients do this by default with `allow_redirects=True`).

```python
circuit_id = "<CIRCUIT_ID>"
asset_id   = "<circuit.gz ASSET_ID>"
out_path   = "<LOCAL_PATH>/circuit.gz"

resp = requests.get(
    f"{base}/circuit/{circuit_id}/assets/{asset_id}/download",
    headers=headers, allow_redirects=True, stream=True
)
resp.raise_for_status()
with open(out_path, "wb") as f:
    for chunk in resp.iter_content(chunk_size=65536):
        f.write(chunk)
```

Then extract with `tar -xzf circuit.gz -C <dest>/` and inspect the extracted tree before assuming file layout — it varies by circuit. For single-neuron circuits, the key file is typically a `.hoc` biophysical model under `components/biophysical_neuron_models/`.

---

## Configuring and running a simulation (obi-one)

The obi-one API drives simulations via a config object → validate → generate-grid → run-batch → poll pipeline. Prefer calling these existing obi-one endpoints over reimplementing simulation logic from scratch in a sandbox — obi-one's `CircuitSimulationScanConfig` is the supported, reproducible path. For a standalone `MEModel` instead of a `Circuit`, the equivalent type is `MEModelSimulationScanConfig` (`initialize.circuit` takes an ME-model ID; `stimuli`/`recordings`/`neuronal_manipulations` follow the same dict-of-blocks shape) — its generate endpoint is `/generated/me-model-simulation-scan-config-generate-grid`. Both types execute through the same backend (`bluecellulab.CircuitSimulation`, run on the small-scale-simulator/Bluenaas service via `run-batch`) — there is no separate execution path for single-cell vs. circuit.

> **`scale=single` circuits are synaptomes — use [[synaptome-simulation-api]], not `CircuitSimulationScanConfig`.** A synaptome is registered as a `Circuit`, but the generic config here produces a campaign that runs yet won't open in the UI. Route off `scale`: `single` → [[synaptome-simulation-api]]; all other scales → `CircuitSimulationScanConfig` below.

**Stimulus/recording targeting:** stimuli default to the soma (class names like `ConstantCurrentClampSomaticStimulus` bake this in) or target a whole `neuron_set`. As of a recent obi-one change (`MorphologyLocationsReference`, merged 2026-07-31 — check this is still current), stimuli can *also* target a set of non-soma points generated by a `morphology_locations` block (`RandomMorphologyLocations`, `ClusteredMorphologyLocations`, `PathDistanceMorphologyLocations`, and grouped/path-distance variants) — there is no way to specify one literal `(section_id, offset)` directly through the scan-config API; constrain a generator to produce exactly one point instead. **Recordings do not yet support morphology-location targeting** — only soma recording is exposed through the scan-config API, even though the underlying SONATA `reports` format supports arbitrary sections.

### Config shape

> Multi-neuron circuits only. If `Circuit.scale == "single"` (a synaptome), stop and use [[synaptome-simulation-api]] — this shape produces a campaign the UI cannot open (see the routing note above).

```python
def make_config(circuit_id, campaign_name, amplitude=0.4):
    return {
        "type": "CircuitSimulationScanConfig",
        "info": {
            "campaign_name": campaign_name,
            "campaign_description": "..."
        },
        "initialize": {
            "type": "CircuitSimulationScanConfig.Initialize",
            "circuit": {"type": "CircuitFromID", "id_str": circuit_id},
            "simulation_length": 1000.0,
            "v_init": -80.0,
            "random_seed": 1
        },
        "recordings": {"soma_voltage": {"type": "SomaVoltageRecording"}},
        "stimuli": {
            "current_step": {
                "type": "ConstantCurrentClampSomaticStimulus",
                "amplitude": amplitude,
                "duration": 600.0,
                "timestamp_offset": 200.0
                # no "delay" field — use timestamp_offset for start time (see Known Issues)
                # do NOT add neuron_set here (see Known Issues)
            }
        },
        "timestamps": {},
        "synaptic_manipulations": {}
    }
```

**Always include a real stimulus** — an empty `"stimuli": {}` silently runs a simulation with no injected current, producing no spikes and no meaningful output.

### Validate, then smoke-test generate

```python
base_obi = f"https://{'staging.' if env == 'staging' else ''}openbraininstitute.org/api/obi-one"

resp = requests.post(f"{base_obi}/config-validation/validate",
                     headers={**headers, "Content-Type": "application/json"},
                     json={"state": config})
result = resp.json()
print(f"valid={result['valid']}  errors={result['errors']}")
```

**Validate passing ≠ generate-grid will succeed.** `/config-validation/validate` only checks schema/types; it does not catch generate-grid failures (e.g. the `neuron_set` bug below). Always smoke-test `generate-grid` on a real circuit — e.g. by sweeping stimulus amplitude on a control circuit — before committing to a full campaign, since a generate failure here means every dependent campaign will fail too.

```python
resp = requests.post(
    f"{base_obi}/generated/circuit-simulation-scan-config-generate-grid",
    headers={**headers, "Content-Type": "application/json"}, json=config)
resp.raise_for_status()
campaign_id = resp.json()  # plain string
```

### Frontend links

Full rules in [[obi-links]]; this is the circuit/simulation specialisation of them. Nothing goes out as a bare UUID or a bare sandbox path.

To hand the user a link to view the simulation campaign in the web app instead of (or alongside) raw entity IDs:

```
https://{domain}/app/virtual-lab/{vlab_id}/{project_id}/data/view/small-microcircuit-simulation/{simulation_campaign_id}/overview
```

- `{domain}` — `staging.openbraininstitute.org` for staging, `www.openbraininstitute.org` for production. Must match whichever `OBI_ENVIRONMENT` was actually used for the API calls — don't hand out a staging link for something registered against production or vice versa.
- `{vlab_id}`, `{project_id}` — from `OBI_VLAB_ID`/`OBI_PROJECT_ID`.
- `{simulation_campaign_id}` — the **`SimulationCampaign` entity's id** (i.e. `campaign_id` above) — **not** the `Circuit` id. Verified the hard way: an earlier version of this doc assumed the trailing UUID was the circuit ID; it's actually the campaign ID from `generate-grid`.

For any *other* entity referenced along the way (`Circuit`, `MEModel`, `CellMorphology`, etc.) there's a simpler generic link that doesn't need the type/vlab/project path above — just the entity's own UUID:

```
https://{domain}/app/entity/{entity_id}
```

The frontend resolves this catch-all route to the correct type-specific detail page regardless of entity type. Prefer this whenever you have an entity ID but not (or don't need) the more specific `data/view/{type}/{id}/{section}` path — e.g. linking to the `Circuit` a campaign was built from.

#### Watching the launched campaign

The campaign also shows up on the workflows activity page, which is the natural "is it done yet?" link to hand over right after launching:

```
https://{domain}/app/virtual-lab/{vlab_id}/{project_id}/workflows?tactivity=simulate&ttype={simulation_type}
```

`{simulation_type}` follows the scale of the circuit being simulated: `single_neuron_circuit_simulation`, `paired_neuron_circuit_simulation`, `small_microcircuit_simulation`, `microcircuit_simulation`, `region_circuit_simulation`, `whole_brain_circuit_simulation`, or `me_model_circuit_simulation`. A circuit *extraction* campaign uses `?tactivity=extract&ttype=circuit_extraction_campaign` instead. Always send both parameters — with `ttype` alone the page stays on its default Build tab and lists nothing. The full table lives in [[obi-links]].

#### Files produced on the sandbox

Circuits, result files, and plots written under the topic directory open directly in JupyterLab:

```
https://{hub_host}/user/{username}/{server_id}/lab/tree/{absolute path, minus the leading slash}
```

Take it from the `download_url` returned by `{obi}:get-sandbox-download-url` — same URL with `/files/` swapped for `/lab/tree/` (`download_url.replace("/files/", "/lab/tree/", 1)`). The path keeps its `home/jovyan` prefix, since the Jupyter root is `/`. Link the containing directory when there are several files. The link is tied to the current sandbox instance, so hand it out in the reply and keep plain paths in the logbook. See [[obi-links]].

### List simulations in a campaign and launch them

```python
base_ec = f"https://{'staging.' if env == 'staging' else ''}openbraininstitute.org/api/entitycore"

resp = requests.get(f"{base_ec}/simulation", headers=headers,
                    params={"simulation_campaign_id": campaign_id, "page_size": 50})
simulation_ids = [s["id"] for s in resp.json().get("data", [])]

# Fire run-batch. Confirm HTTP 202 via streaming (read first chunk only, then stop) —
# run-batch streams progress indefinitely; consuming the whole body hangs.
# Read-side timeouts on the streaming response raise ConnectionError, not Timeout — catch both.
sss_base = f"https://{'staging.' if env == 'staging' else ''}cell-a.openbraininstitute.org/api/small-scale-simulator"
with requests.Session() as s:
    try:
        resp = s.post(f"{sss_base}/circuit/simulation/run-batch", headers=headers,
                      json={"simulation_ids": simulation_ids}, stream=True, timeout=10)
        if resp.status_code in (200, 202):
            for chunk in resp.iter_content(chunk_size=256):
                break  # confirm streaming started, then stop
        else:
            print(f"ERROR: {resp.status_code} {resp.text[:300]}")
    except (requests.exceptions.Timeout, requests.exceptions.ConnectionError):
        pass  # expected — streaming body doesn't close cleanly
```

### Poll until done

```python
# Correct query param is used__id (not simulation_id) — confirmed from EntityCore OpenAPI spec.
# The wrong param silently returns records with sim_id=None.
resp = requests.get(f"{base_ec}/simulation-execution", headers=headers,
                    params={"used__id": sim_id, "page_size": 5})
executions = resp.json().get("data", [])
e = executions[0] if executions else {}
print(f"status={e.get('status', 'no record yet')}  ended={e.get('end_time')}")
```

Call this repeatedly (as separate calls, not in a loop with `time.sleep()` — many sandbox execution environments cap a single call at ~30s, and a `sleep(20)` will hit that) until status is `done`. A 9-neuron, 1000ms simulation typically takes 3–10 minutes wall-clock.

---

## Simulation outputs

Once execution completes, results are stored as `simulation-result` entities linked to the execution's `generated` field — not on the simulation itself:

```python
resp = requests.get(f"{base_ec}/simulation-execution", headers=headers,
                    params={"used__id": sim_id, "page_size": 5})
e = resp.json().get("data", [{}])[0]
result_id = next((g["id"] for g in (e.get("generated") or []) if g.get("type") == "simulation_result"), None)

resp = requests.get(f"{base_ec}/simulation-result/{result_id}", headers=headers)
assets = resp.json().get("assets", [])
for a in assets:
    if a.get("is_directory"):
        continue
    dl = requests.get(f"{base_ec}/simulation-result/{result_id}/assets/{a['id']}/download",
                      headers=headers, allow_redirects=True, stream=True)
    dl.raise_for_status()
    with open(f"<OUT_DIR>/{a['path']}", "wb") as f:
        for chunk in dl.iter_content(chunk_size=65536):
            f.write(chunk)
```

Output files are HDF5 in SONATA report format. **Population names are circuit-specific — always inspect the file's groups (e.g. `list(f["spikes"].keys())`) rather than assuming a name.**

- **`spikes.h5`** — `spikes/<population>/timestamps` (spike times) + `spikes/<population>/node_ids` (which neuron fired).
- **`soma_voltage.h5`** — `report/<population>/data`, shape `[timesteps, n_neurons]`; the time axis is in `report/<population>/mapping/time` as `[t_start, t_stop, dt]`.

```python
import h5py
with h5py.File("spikes.h5") as f:
    population = next(iter(f["spikes"].keys()))  # don't hardcode a population name
    spike_times = f[f"spikes/{population}/timestamps"][:]
    spike_nodes = f[f"spikes/{population}/node_ids"][:]
with h5py.File("soma_voltage.h5") as f:
    population = next(iter(f["report"].keys()))
    voltage = f[f"report/{population}/data"][:]
    t_start, t_stop, dt = f[f"report/{population}/mapping/time"][:]
```

For structured access instead of raw HDF5 indexing (filtering by population/node IDs/time window, or built-in plotting), `bluepysnap` provides `SpikeReport`/`PopulationSpikeReport` (`bluepysnap/spike_report.py`) and `CompartmentReport`/`SomaReport` (`bluepysnap/frame_report.py`) on top of the same files — worth reaching for once analysis gets more involved than a couple of array slices.

What you do with these arrays (plotting, statistics) is presentation/analysis, not platform mechanics — see [[disease-modeling]] or the task at hand for that.

---

## Known Issues (as of 2026-07, staging)

- **`scale=single` circuits use the synaptome endpoint, not this one** — a `CircuitSimulationScanConfig` campaign for one runs but won't open in the UI, and `/config-validation/validate` won't catch it. See [[synaptome-simulation-api]] and the routing note above.
- **`neuron_set` in stimulus blocks is broken.** Do not add a `neuron_set` field to any stimulus block in `CircuitSimulationScanConfig` — even a valid value causes a 500 on `generate-grid`. Omit the field entirely.
- **No `delay` field on `ConstantCurrentClampSomaticStimulus`.** Use `timestamp_offset` (ms) instead; `delay` causes a 422 `extra_forbidden` error.
- **Poll with `used__id`, not `simulation_id`.** The correct EntityCore query param on `/simulation-execution` is `used__id=<sim_id>`. The wrong name silently returns all records with `sim_id=None` rather than erroring.
- **`run-batch` raises `ConnectionError`, not `Timeout`,** when the streaming response times out on read. Catch both `requests.exceptions.Timeout` and `requests.exceptions.ConnectionError`. Better: `stream=True`, read only the first chunk to confirm 202, then stop.
- **No `register_emodel` helper.** `obi_one.utils` ships `register_circuit` but not an emodel equivalent — use `entitysdk` directly for that entity type.
- **Empty `stimuli: {}` is a silent no-op**, not an error — always include a real stimulus.
- **Don't `sleep()` inside a single tool/execution call** if the environment has a short per-call timeout (commonly ~30s) — poll via repeated separate calls instead.

---

## Working directory

When this runs on the OBI sandbox, all files go under the conversation's topic directory on the persistent volume, per the convention in [[obi-logbook]] — one directory per scientific topic (`v1-microcircuit`, `alzheimer-ca1`), created before any work starts, reused if it already exists:

```
/home/jovyan/<topic>/
├── logbook.md
├── circuit-original/   ← the downloaded circuit, unmodified
├── circuit-modified/   ← the modified copy (+ modifications.json sidecar)
├── results/            ← spikes.h5, soma_voltage.h5; one subdir per condition
├── plots/
└── code/
```

`logbook.md`, `results/`, `plots/`, and `code/` are fixed by [[obi-logbook]]; the circuit directories are this workflow's inputs. **Keep `circuit-original/` untouched** — modify a copy, never the download, so the baseline stays reproducible and the diff stays inspectable.

## Logbook

This skill is platform mechanics, but it is almost always used in service of a scientific question — and that session gets a logbook, per [[obi-logbook]]. Read that skill for the location, format, and content rules; the split is:

- **Everything on this page is plumbing and stays out of the logbook** — endpoints, config JSON shapes, `generate-grid`/`run-batch` calls, polling params, the Known Issues above, auth.
- **The scientific decisions made while using it go in** — see below.

Record, as the session goes:

- **Circuit selection** — name, brain region, species/strain, neuron count, cell types and populations present, and why this circuit fits the question (in particular, why this scale).
- **Modifications** — every parameter changed: quantity, baseline → new value, **units**, which population / cell type / compartment / synapse type it applies to, and the **citation or reasoning** justifying it. Log this *before* running with it.
- **Simulation protocol** — stimulus type, amplitude, timing, target cells, duration, timestep, temperature, recorded variables and locations, seeds/repetitions. State explicitly what is held identical between conditions being compared.
- **Conditions** — what is the control, what is perturbed, and what each is testing.
- **Results** — firing rates, spike counts, latencies, voltage amplitudes with units; figures and what each shows.
- **Interpretation and caveats** — what the outcome means for the question, including negative results, and the limits of the model (network size, missing afferents, short duration).
- **Provenance table** — source circuit ID, registered modified-circuit ID, campaign IDs, simulation IDs.

A technical fact only enters the logbook when it changes how a result should be read — and then it is written as a scientific caveat ("simulation run for 500 ms, so late adaptation is not captured"), never as an incident report.
