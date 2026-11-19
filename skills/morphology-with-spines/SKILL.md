---
name: morphology-with-spines
description: Guide for loading and analyzing a neuron morphology with dendritic spines from the OBI platform. Use when the user wants to work with spiny morphologies — download a CellMorphology's spine-annotated reconstruction, inspect its spine table, access spine meshes/skeletons, access classified spine types, compute spine statistics, or measure path distances between spines or from spines to the soma. The data lives as either a CellMorphology or Circuit entity whose property has_segmented_spines is True. For CellMorphologies this is a separate .h5 asset labelled morphology_with_spines, for Circuits this is part of the SONATA circuit asset. In both cases the .h5 is loaded and analyzed with the morph-spines library, and path distances are computed with ConnectomeUtilities.
license: Apache-2.0
---

# Neuron Morphology with Spines — Load & Analyze

> **Client-agnostic.** The Python snippets below work from any environment with network access to the OBI APIs and appropriate credentials — an OBI sandbox Python execution tool, a local script, another agent surface. In **Claude Chat**, this compute runs via the OBI MCP connector's `execute-python`/`execute-shell` tools; getting results back to the user is a separate concern, handled by [[chat-obi-sandbox-bridge]]. Throughout this doc, `{obi}` stands for whatever that connector is named when one is in use (e.g. `{obi}:execute-python`); when no such connector exists, run the same code directly.

> **Generic skills this one builds on — follow them, don't re-derive:**
> - [[obi-sandbox-auth]] — how to obtain, exchange, and refresh OBI credentials. The `entitysdk` client below assumes a token obtained that way (`$OBI_ACCESS_TOKEN`). Never construct auth calls ad hoc.
> - [[chat-obi-sandbox-bridge]] — (Claude Chat / Claude Desktop) run the download/analysis on the OBI sandbox and get result files (tables, plots, exported meshes) back to the user from there.
> - [[obi-logbook]] — **mandatory** whenever this skill is used in service of a scientific question. See the Logbook section below.
> - [[obi-links]] — every entity id and sandbox file handed to the user goes out as a hyperlink, never as a bare UUID or path. See the Linking section below.
> - [[virtual-lab-manager-api]] — the virtual-lab / project context (`$OBI_VLAB_ID`, `$OBI_PROJECT_ID`) that scopes every EntityCore query. Read it from the environment, never from memory.
> - [[obi-circuit-simulation]] — when the spiny morphologies come from a `Circuit` (not a `CellMorphology`): how to find, **stage** (prefer symlink over download), and read a SONATA circuit. The "Alternative source — a Circuit with segmented spines" section below picks up from the staged `circuit_config.json`.

> **Library docs:** Use the **`context7`** MCP server (when available) for up-to-date API documentation on `entitysdk`, the morph-spines library, ConnectomeUtilities (`conntility`), and `bluepysnap`. Call `context7:resolve-library-id("<library>")` then `context7:query-docs(...)`. The morph-spines source and format spec live at `github.com/openbraininstitute/morph-spines`; the path-distance calculator at `github.com/openbraininstitute/ConnectomeUtilities`.

## Overview

A neuron morphology *with spines* is a standard neurite reconstruction (soma + dendrites + axon) enriched with per-spine data: each dendritic spine's position, orientation, type classification, and — optionally — its skeleton and surface mesh. On the OBI platform this is modelled as:

```
CellMorphology (has_segmented_spines = True)        Circuit (has_segmented_spines = True)
    └── asset  label = morphology_with_spines           └── SONATA asset → circuit_config.json
        │      (HDF5, morph-with-spines format)              └── nodes[pop].config["alternate_morphologies"]["h5v1"]
        │  download with entitysdk                               │  (single merged .h5, all neurons)
        │                                                        │  path read with bluepysnap
        └──────────────────────────┬────────────────────────────┘
                                    ▼
    neuron.h5  ──  load_morphology_with_spines()  (morph-spines library)
        │
        ▼
    MorphologyWithSpines  →  skeleton + spine table + spine meshes/skeletons
```

Two libraries, two jobs:

- **`entitysdk`** — find the `CellMorphology` entity, ascertain that its has_segmented_spines property is True, and download its `morphology_with_spines` asset to a local `.h5` file.
- **`morph-spines`** — load that `.h5`, read the spine table, and access spine skeletons, meshes, and type classifications for analysis.

For **path distances along the dendrite** (spine↔spine or spine↔soma), a third library — **ConnectomeUtilities** (`conntility`) — operates on the plain neuron morphology plus rows of the spine table. See "Path distances" below.

A plain `CellMorphology` has a `morphology` asset (`.asc`/`.swc`/`.h5`) with no spine data. The spine-annotated reconstruction is a **separate asset on the same entity**, distinguished by its label `morphology_with_spines`. Not every `CellMorphology` has one — only reconstructions that carry spine annotations (typically derived from EM). Whether a given `CellMorphology` has that asset is told by its **`has_segmented_spines`** property (`True`/`False`) — the same property that flags a `Circuit` — so query it to decide before listing or downloading assets. Select by label, don't assume the first asset.

### Two entry points

Spiny morphologies reach you from **two kinds of entity** — the loading/analysis once you have the `.h5` is identical; only how you locate the file differs. In **both**, the `has_segmented_spines` property tells you up front whether spine data is present:

1. **`CellMorphology`** with `has_segmented_spines == True` — a single reconstruction. Download its asset labelled `morphology_with_spines` (Steps 1–3 below).
2. **`Circuit`** with `has_segmented_spines == True` — the circuit's SONATA asset *embeds* the spiny morphologies in a single merged `.h5`, referenced from `circuit_config.json`. Read that path with bluepysnap, then load the same way (see "From a Circuit with segmented spines" below).

Either way you end up with a morph-with-spines `.h5` that `load_morphology_with_spines()` opens.

## Prerequisites

```bash
pip install entitysdk morph-spines
```

`morph-spines` pulls in `h5py`, `numpy`, `pandas`, NeuroM/MorphIO (spine skeletons), and `trimesh` (spine meshes). On the OBI sandbox these are generally already present; install into the session if `import morph_spines` fails.

For path-distance analysis (spine↔spine, spine↔soma) you also need **ConnectomeUtilities** (`conntility`):

```bash
pip install git+https://github.com/openbraininstitute/ConnectomeUtilities
```

When the source is a **`Circuit`** (rather than a `CellMorphology`), you also need **bluepysnap** to read `circuit_config.json`:

```bash
pip install bluepysnap
```

## Step 1 — Build the entitysdk client

Scope the client to the current virtual lab / project from the environment, never from remembered ids — a tool call can be redirected to another project and the environment follows while literals do not (see [[virtual-lab-manager-api]]).

```python
import os
from pathlib import Path
from entitysdk import Client, ProjectContext, models
from entitysdk.token_manager import TokenFromEnv

client = Client(
    token_manager=TokenFromEnv("OBI_ACCESS_TOKEN"),   # see [[obi-sandbox-auth]]
    project_context=ProjectContext(
        virtual_lab_id=os.environ["OBI_VLAB_ID"],
        project_id=os.environ["OBI_PROJECT_ID"],
    ),
    environment=os.environ.get("OBI_ENVIRONMENT", "production"),
)
```

## Step 2 — Find the CellMorphology entity

If the user already has the morphology id, skip to Step 3. Otherwise search EntityCore. `search_entity` returns an iterator; take as many as you need. Make sure to include a query for the has_segmented_spines property being True to ensure presence of a morphology_with_spines asset.

```python
# Or search — e.g. by m-type
it = client.search_entity(
    entity_type=models.CellMorphology,
    query={
        "mtype__pref_label": "L5_TPC:A",
        "has_segmented_spines": True  # To ensure presence of morphology_with_spines
    },
    limit=10,
)
for m in it:
    print(m.id, m.name)
```

If a list of entities has already been populated, query each entity's **`has_segmented_spines`** property - same semantics as on a `Circuit`. When `True`, the `CellMorphology` has a `morphology_with_spines` asset; when `False`, it is a plain reconstruction and there is nothing to download. This is the cheap pre-check — it avoids listing/downloading assets just to find out.

```python
# Keep only morphologies that actually have segmented spines
spiny = [m for m in it if getattr(m, "has_segmented_spines", False)]

# Or gate a single entity before Step 3
if not getattr(morphology, "has_segmented_spines", False):
    raise RuntimeError(
        f"CellMorphology {morphology.id} has has_segmented_spines=False — "
        "no morphology_with_spines asset to download."
    )
```

Inspecting the assets directly (Step 3) is the fallback if the property is unavailable on an older entity.

## Step 3 — Download the `morphology_with_spines` asset

The spine data is the asset **labelled `morphology_with_spines`** — select on that label rather than taking an arbitrary asset. `morphology_with_spines` is a valid `entitysdk` `AssetLabel`.

```python
out_dir = Path("<local-dir>/morphologies")
out_dir.mkdir(parents=True, exist_ok=True)

downloaded = list(
    client.download_assets(
        (morphology.id, models.CellMorphology),
        selection={"label": "morphology_with_spines"},
        output_path=out_dir,
    )
)

if not downloaded:
    raise RuntimeError(
        f"CellMorphology {morphology.id} has no 'morphology_with_spines' asset — "
        "it is a plain reconstruction without spine annotations."
    )

h5_path = downloaded[0].path   # the local .h5 file
print("downloaded:", h5_path)
```

Notes:
- `download_assets` takes either an `Entity` or a `(id, entity_type)` tuple, a `selection` filter, and an `output_path` **directory**. It returns `DownloadedAssetFile` objects; `.path` is the written file and `.asset` the asset metadata.
- To inspect assets first (e.g. confirm the label exists before downloading), use `client.get_entity_assets(morphology.id, entity_type=models.CellMorphology)` or `client.select_assets(morphology, {"label": "morphology_with_spines"})`.
- The file is in the **morph-with-spines HDF5 format** (groups `/morphology`, `/edges`, `/spines`, `/soma`). It can hold one or multiple neurons keyed by neuron name.

## Alternative source — a Circuit with segmented spines

This skill also applies to **`Circuit`** entities whose `has_segmented_spines` property is `True`. In that case there is no separate `morphology_with_spines` asset to download — the circuit's **SONATA asset already contains the spiny morphologies in a single merged `.h5` file**, and `circuit_config.json` points to it. You locate that file instead of doing Steps 1–3, then join the single-morphology flow at Step 4.

First materialize the circuit's SONATA directory (prefer staging/symlink over download — see [[obi-circuit-simulation]]), which gives you a `circuit_config.json`. Then read the path with **bluepysnap**: the spiny-morphology `.h5` is listed **per node population** under `alternate_morphologies.h5v1`.

```python
from bluepysnap import Circuit

network = Circuit("<path>/circuit_config.json")

# Pick a biophysical node population (the one carrying morphologies).
# network.nodes.population_names lists them; it is a biophysical node
# population if its .type is "biophysical"
pop_name = "biophysical_neurons"
pop = network.nodes[pop_name]
if pop.type != "biophysical":
    raise ValueError("Non-biophysical node population selected")
cfg = network.nodes[pop].config
h5_path = cfg["alternate_morphologies"]["h5v1"]   # merged spiny morphologies .h5
print("spiny morphologies:", h5_path)
```

Notes:
- `alternate_morphologies.h5v1` lives **inside a node population's config**, not at the top level of the file. Only `biophysical` populations have it; virtual populations do not — so iterate `network.nodes.population_names` and skip populations where the key is absent.
- bluepysnap resolves the SONATA `$BASE_DIR` manifest for you, so `cfg["alternate_morphologies"]["h5v1"]` comes back as a usable path rather than the raw `$BASE_DIR/...` string stored in the JSON.
- The referenced `.h5` is one **merged** morph-with-spines file holding **all** the circuit's neurons, each keyed by its node name (see "Multiple neurons in one file"). This is the same format `load_morphology_with_spines()` reads — so continue at Step 4 with this `h5_path`, then select the neuron(s) you care about by key.
- The circuit also carries plain per-cell morphologies under `morphologies_dir` (`.h5`/`.swc`); those have **no** spine data. The spiny data is only in the `alternate_morphologies.h5v1` merged file.

For reference, the raw `circuit_config.json` entry looks like this (node population → `alternate_morphologies.h5v1`):

```json
"nodes": [
  {
    "nodes_file": "$BASE_DIR/synaptome-nodes.h5",
    "populations": {
      "biophysical_neurons": {
        "type": "biophysical",
        "morphologies_dir": "$BASE_DIR/morphologies",
        "alternate_morphologies": {
          "h5v1": "$BASE_DIR/morphologies/merged_spiny_morphologies.h5"
        }
      }
    }
  }
]
```

## Step 4 — Load with morph-spines

This step is easy for files from `CellMorphology` entities, as they will contain only a single morphology. 

```python
from morph_spines import load_morphology_with_spines

m = load_morphology_with_spines(
    str(h5_path),
    load_meshes=True,           # eager mesh load; set False for lazy/skeleton-only work
)
```
`load_meshes=False` keeps the load light when you only need the spine table and skeletons; mesh access then happens on demand.

In the case of `Circuit` entities, the morphology name of the neuron needs to be specified as well (see "Multiple neurons in one file" below). Use the `.get` function of the node population with the neuron's id to get the morphology name. The name to use is the part past the first "/".

```python
from morph_spines import load_morphology_with_spines

morph_name = pop.get(<neuron id>, "morphology").split("/")[1]

m = load_morphology_with_spines(
    str(h5_path),
    morphology_name=morph_name,
    load_meshes=True,           # eager mesh load; set False for lazy/skeleton-only work
)
```


## Step 5 — Analyze

### Spine table (per-spine properties)

Each spine is a row with position, orientation, placement on the neurite, and (optionally) type/volume/neck-diameter. Mandatory columns include `afferent_center_{x,y,z}`, `afferent_surface_{x,y,z}`, `afferent_section_id`, `afferent_segment_id`, `afferent_section_pos`, `spine_length`, `spine_orientation_vector_{x,y,z}`, `spine_rotation_{x,y,z,w}`, `spine_morphology`, `spine_id`. Optional: `spine_volume`, `spine_neck_diameter`, `spine_type`.

```python
# Spine count and type distribution
n_spines = m.spines.spine_count
types = [m.spines.spine_type(i) for i in range(n_spines)]

import collections
print("spines:", n_spines)
print("type distribution:", collections.Counter(types))
# valid spine_type values: undefined, thin, long_thin, mushroom, stubby, filopodium, branched
```

### Spine meshes (surface geometry via trimesh)

```python
mesh = m.spines.spine_mesh(0)                     # trimesh.Trimesh for spine 0
print(mesh.vertices.shape, mesh.faces.shape)

# Head region only (drop the neck) — uses the head/neck triangle classification
head_mesh = m.spines.spine_mesh(0, include_neck=False)

# Per-spine head volume / surface area for a morphometric summary
import numpy as np
volumes = np.array([m.spines.spine_mesh(i).volume for i in range(n_spines)])
print("mean spine head volume:", volumes.mean())
```

Head/neck classification lets you separate head from neck triangles (`include_head` / `include_neck`); branched spines may have multiple heads. See the format spec for the triangle-ordering details.

### Spine locations on the neuron morphology

Spine locations on the neuron morphology are exposed via NeuroM/MorphIO; the neurite skeleton itself follows the H5 v1 morphology structure (`/points`, `/structure`). Use NeuroM for standard morphometrics (branch orders, path lengths, section counts) on the skeleton, and the spine table for spine density along neurites (`afferent_section_id` / `afferent_section_pos` locate each spine on the arbor as in the SONATA standard).

```python
# Example: spine density per section
import collections
by_section = collections.Counter(
    m.spines.spine_table["afferent_section_id"]  # or the equivalent accessor
)
```

> Confirm exact accessor names against the installed version — the library evolves. Use `context7` for `morph-spines`, or inspect in-session: `import morph_spines; help(morph_spines)` and `help(type(m.spines))`.

### Validation (when a file looks suspect)

```python
from morph_spines import validate_morph_with_spines_file

result = validate_morph_with_spines_file(str(h5_path), check_data_integrity=True)
print(result)             # human-readable summary
assert result.is_valid    # programmatic gate before trusting the data
```

### Path distances (spine↔spine, spine↔soma)

Path distances **along the dendritic tree** — between two spines, or from a spine to the soma — are computed with the **`conntility.subcellular.MorphologyPathDistanceCalculator`** class from the [ConnectomeUtilities](https://github.com/openbraininstitute/ConnectomeUtilities) package. This is a graph/tree distance along the neurite, not the straight-line Euclidean distance between spine positions.

Instantiate the calculator with the **plain neuron morphology** (a `morphio.Morphology` — the neurite skeleton of the loaded morphology). This can be constructed from the `.morphology` property of the morphology with spines using `.to_morphio()`. The spine table's placement columns are exactly the input format `path_distances` expects, so you feed it **rows of the spine table** directly.

```python
import pandas as pd
from conntility.subcellular import MorphologyPathDistanceCalculator

# `m` is the morphology-with-spines that is to be analyzed
morph = m.morphology.to_morphio()
calc = MorphologyPathDistanceCalculator(morph)

# Rows of the spine table locate each spine on the arbor via
# afferent_section_id / afferent_segment_id / afferent_segment_offset —
# the column names path_distances() reads by default.
spine_locs = m.spines.spine_table   # confirm accessor name in-session

# Pairwise path distances between all spines → 2-D numpy array (n_spines × n_spines)
d_spine_spine = calc.path_distances(spine_locs)

# Path distance from each spine to the SOMA.
# The soma is represented as a one-row DataFrame at the soma point:
soma = pd.DataFrame({
    "afferent_section_id": [0],
    "afferent_segment_id": [0],
    "afferent_segment_offset": [0.0],
})
d_spine_soma = calc.path_distances(spine_locs, soma)   # shape (n_spines, 1)
```

Notes:
- **Section id is base-1**: `afferent_section_id = 0` is the soma, `1` is the first real section. That is why the soma row uses section/segment `0` and offset `0.0`; the soma is treated as a point, so path distance is measured as if from the root of a root section.
- `path_distances(locs_from, locs_to=None)` is **pairwise within `locs_from`** when `locs_to` is omitted, and **cross** between the two sets otherwise (`spines → soma` here). It returns a 2-D `numpy` array; index it with the row order of the input frames.
- The default column names (`afferent_section_id`, `afferent_segment_id`, `afferent_segment_offset`) match both the spine table and bluepysnap afferent-synapse output, so no `str_*` overrides are needed. Override them only if your frame uses different names.
- Building the calculator precomputes section-to-section tables; it pays off when looking up many distances on **one** morphology. Reinstantiate per neuron for a multi-neuron file.
- ConnectomeUtilities also exposes `calc.nearest_neighbor_distances(...)` for clustering analyses of grouped locations (e.g. spines by type), with optional random-control normalization.

Install if missing: `pip install git+https://github.com/openbraininstitute/ConnectomeUtilities` (or the packaged `connectome-utilities`/`conntility` distribution available in your environment).

### Innervation of spines
This part is only relevant for the case of Circuit entities. In that case, an edge population may contain a property "spine_id" that specifies which spine is innervated by the synapse. The value can be directly used to load the corresponding spine with morph-spines as described above. A "spine_id" of -1 indicates innervation of a shaft instead of a spine.

```python
edge_pop = network.edges[<edgepop_name>]
 if "spine_id" in edge_pop.property_names:
    innervated_spines = edge_pop.afferent_edges(<neuron id>, properties="spine_id")
    print(f"{(innervated_spines == -1).sum()} synapses are on shafts")
    syns_per_spine = innervated_spines.value_counts().drop(-1)  # Drop -1 -> shafts
    syns_per_spine_dist = syns_per_spine.value_counts()  # Distribution of synapses per spine
    print(syns_per_spine_dist)  
```

## Multiple neurons in one file

A single `morphology_with_spines` H5 can contain several neurons keyed by name (`/edges/<name>`, `/morphology/<name>`, …). When the user cares about a specific neuron, select it by its key; don't assume a single-neuron file. If you need to split or recombine files, `morph-spines` provides `merge_morphologies_with_spines(...)` (with optional renaming) and the `write_*` helpers.

This is exactly the shape of the **Circuit** case: `alternate_morphologies.h5v1` is one merged file holding every neuron in the population, each keyed by its node name. Match the circuit's node ids/names to the H5 neuron keys to pull the right neuron.

## Common pitfalls

- **Taking the wrong asset.** A `CellMorphology` usually also has a plain `morphology` asset (`.asc`/`.swc`/`.h5`) with no spines. Always select `label == "morphology_with_spines"`; the plain asset will load as a morphology but has no spine table.
- **No spine asset at all.** Many morphologies are not spine-annotated. Check `has_segmented_spines` first (Step 2) rather than discovering the gap by an empty download; still handle the empty-download case explicitly (Step 3) rather than indexing `[0]` blindly.
- **`output_path` is a directory.** `download_assets` writes into the directory you pass and returns the file path(s) — don't pass a file name.
- **Meshes vs skeletons.** `spine_mesh(i)` returns surface geometry (trimesh); spine *skeletons* and the neurite skeleton are separate (NeuroM/MorphIO). Pick the representation your analysis needs.
- **Coordinate frame.** `spines_are_centered` controls whether spine coordinates are in the local spine frame or the global neuron frame — set it to match what your downstream computation expects.
- **Path distance ≠ Euclidean distance.** `MorphologyPathDistanceCalculator` measures distance *along the neurite tree*, not straight-line distance between 3D positions. Use it for spine↔spine and spine↔soma; use the spine positions (`afferent_center_*`) only if you actually want Euclidean distance.
- **Base-1 section ids for path distances.** ConnectomeUtilities treats `afferent_section_id = 0` as the soma and `1` as the first real section. The spine table already follows this (afferent columns are SONATA-based), so pass rows straight through; the soma is the one-row frame with section/segment `0` and offset `0.0`. However, the neurom library uses different indexing where `0` is the first real section. 
- **Circuit source: wrong morphology dir.** For a `Circuit` with `has_segmented_spines`, the spiny `.h5` is the **merged** file at `nodes[pop].alternate_morphologies.h5v1` — not the plain per-cell files under `morphologies_dir`, which carry no spine data. Read the path with bluepysnap (`network.nodes[pop].config`) so `$BASE_DIR` is resolved; don't hand-parse the raw JSON.
- **Incorrect data types for path distance inputs.** The MorphologyPathDistance calculator expects the columns of input dataframes to have specific data types (int64 for "afferent_section_id" and "afferent_segment_id"; float64 for "afferent_segment_offset"). While these are the data types that SONATA and morph-spines typically return, this can fail in the case of empty results (e.g., zero afferent synapses). Such empty results may have object data types. Empty results should be explicitly excluded and data types checked or cast accordingly.

---

## Working directory

All files this workflow produces go under the conversation's topic directory on the persistent volume, per the convention in [[obi-logbook]] — one directory per scientific topic, created before any work starts, reused if it already exists:

```
/home/jovyan/<topic>/
├── logbook.md
├── morphologies/       ← morphology_with_spines .h5 files pulled from EntityCore
├── results/            ← spine tables, type distributions, morphometric summaries
├── plots/              ← spine-type histograms, density-along-neurite, mesh renders
└── code/
```

`logbook.md`, `results/`, `plots/`, and `code/` are fixed by [[obi-logbook]]; `morphologies/` holds this workflow's inputs. `/tmp` is ephemeral overlay storage and will not survive the session — don't leave downloads there.

## Logbook

Every scientific session is logged per [[obi-logbook]] — read it for the location, format, and content rules.

Record, as the session goes:

- **Objective** — what the spine analysis is for (comparing spine densities across cell types, characterizing a reconstruction, feeding synapse placement) and why spine-level detail is needed.
- **Input morphology** — which `CellMorphology` or `Circuit`, by name: cell type / m-type, species and brain region, provenance of the spine annotation (e.g. EM reconstruction), and how many neurons the file holds. For a circuit source, note that it was the `has_segmented_spines` merged morphology file and which node population it came from.
- **What was computed** — spine count, type distribution, density along neurites, head volumes / neck diameters, path distances (spine↔spine, spine↔soma), and over which neurites — with units.
- **Findings** — the actual numbers and their scientific reading (e.g. "mushroom spines concentrated on proximal apical dendrite; mean head volume ≈ 0.08 µm³"), not raw tables.
- **Caveats** — missing head/neck classification, spines without type labels, reconstruction/segmentation artefacts inherited from the source, truncated arbors at the volume boundary.
- **Provenance** — the `CellMorphology` id and asset (`morphology_with_spines`), or the `Circuit` id and the `alternate_morphologies.h5v1` file it provided.

Do **not** log the download mechanics, entitysdk/morph-spines call signatures, or auth — see the "What never goes in" list in [[obi-logbook]].

## Linking

**Link everything.** Per [[obi-links]], every entity and file mentioned in a reply goes out as a hyperlink — a bare UUID or a bare path is a defect, not a style choice. `{domain}` is `staging.openbraininstitute.org` on staging, `www.openbraininstitute.org` on production, and must match the environment the download ran against.

- **Entities** — `https://{domain}/app/entity/{id}`. Link the source `CellMorphology`, or the source `Circuit` when the spiny morphologies came from one.
- **Sandbox files** — downloaded `.h5` files, spine tables, and plots: take the `download_url` from `{obi}:get-sandbox-download-url` and swap `/files/` for `/lab/tree/`, keeping the rest of the path (`home/jovyan` included).
- **Jobs** — this is a read-and-analyze task; it launches no campaign, so there is no workflows activity page to link.
