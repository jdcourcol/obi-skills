---
name: data-integration-api
description: Guide for ingesting data into the OBI platform via obi-one REST API. Covers morphology validation, registration, mesh generation, circuit registration, metadata registration (contributors, subjects, publications), and morphology metrics. Morphologies and circuits cannot be created in EntityCore — as of today they are registered only through obi-one's declared endpoints, which handle validation, format conversion, and asset generation automatically. Other entity types can be created in EntityCore, but the declared endpoints are preferred where they exist.
license: Apache-2.0
---

# Data Integration via obi-one API

> **Generic skills this one builds on — follow them, don't re-derive:**
> - [[obi-sandbox-auth]] — how to obtain, exchange, and refresh OBI credentials, and the offline-access consent flow. Never construct auth calls ad hoc; the "Authorization" section below assumes a token obtained that way.
> - [[chat-obi-sandbox-bridge]] — (Claude Chat / Claude Desktop) run heavy or scientific work on the OBI sandbox, and get result files back to the user from there.
> - [[obi-logbook]] — **mandatory.** Keep a scientific logbook for every ingestion session. See the Logbook section below.
> - [[obi-links]] — every entity id, sandbox file, and launched job handed to the user goes out as a hyperlink, never as a bare UUID or path. See the Linking section below.

## Overview

**Morphologies and circuits cannot be created in EntityCore.** As of today they are registered only through obi-one's `/declared/` endpoints (`register-morphology-with-calculated-metrics`, `circuit/register`) — there is no supported path that creates either one directly.

Other entity types *can* be created in EntityCore, but prefer obi-one's `/declared/` endpoints wherever one exists (subjects, contributors, publications), because obi-one handles:
- File validation and format conversion
- Metadata resolution (DOI, ORCID, ROR lookups)
- Asset generation (meshes, LODs, metrics)
- Async validation jobs
- Proper entity linking

## Authentication (all requests)

- `Authorization: Bearer <keycloak_token>` — Keycloak (realm: SBO)
- `virtual-lab-id: <uuid>` — HTTP header, from `$OBI_VLAB_ID` in the sandbox
- `project-id: <uuid>` — HTTP header, from `$OBI_PROJECT_ID` in the sandbox

Read both from the environment, never from memory: a tool call can be redirected to another project, and the environment follows while remembered ids do not. See [[virtual-lab-manager-api]].

Base URLs:
- Staging: `https://staging.cell-a.openbraininstitute.org/api/obi-one`
- Production: `https://cell-a.openbraininstitute.org/api/obi-one`

---

## Morphology Ingestion

### Step 1: Validate a Morphology File

Before registering, validate the file and get converted formats.

**Endpoint:** `POST /declared/test-neuron-file`

**Content-Type:** `multipart/form-data`

| Field | Type | Description |
|-------|------|-------------|
| `file` | file upload | Morphology file (.swc, .h5, or .asc) |
| `single_point_soma` | bool (query) | Convert soma to single point (default: false) |

**Response:** ZIP file containing the morphology converted to all supported formats (.swc, .h5, .asc).

If validation fails, the response contains diagnostic errors.

**Supported formats:** `.swc`, `.h5`, `.asc`

### Step 2: Register the Morphology

**Endpoint:** `POST /declared/register-morphology-with-calculated-metrics`

**Content-Type:** `multipart/form-data`

| Field | Type | Description |
|-------|------|-------------|
| `file` | file upload | Morphology file (.swc, .h5, or .asc) — **required** |
| `metadata` | string (JSON) | Entity metadata (name, brain_region, subject, contributions, …). Defaults to `{}` |

This one call registers the entity, uploads the asset, computes and registers the measurements, and — when it can — generates the GLB surface mesh. It is the whole of Steps 3 and 4 below as well, so a plain morphology ingestion needs nothing else.

**Response:**
```json
{
  "entity_id": "<cell-morphology-uuid>",
  "measurement_entity_id": "<measurement-uuid>",
  "mesh_asset_id": "<glb-asset-uuid>",
  "status": "success",
  "morphology_name": "<name>"
}
```

`mesh_asset_id` is **nullable** — the mesh is generated "when possible", so a `null` here means registration succeeded but the GLB did not. Check it, and fall back to Step 3 if you need the mesh.

> Register metadata prerequisites (subject, contributors, publications) **before** this call, so their IDs can go in `metadata`. See [Metadata Registration](#metadata-registration).

Steps 3 and 4 exist for morphologies that are **already registered** — enriching an existing entity, or filling in a mesh that came back null. They are not part of a fresh ingestion.

### Step 3: Generate a 3D Mesh from the Morphology

**Endpoint:** `POST /declared/convert-morphology-to-registered-mesh/{cell_morphology_id}`

Converts the SWC asset on an existing `cell-morphology` entity to an annotated GLB mesh and uploads it as a new asset.

**Response:**
```json
{
  "asset_id": "<glb-asset-uuid>",
  "status": "success"
}
```

**Errors:**
- 404 — Morphology not found
- 409 — GLB asset already exists
- 422 — No SWC asset on the morphology

### Step 4: Compute and Register Morphology Metrics

**Endpoint:** `POST /declared/neuron-morphology-metrics/{cell_morphology_id}/register`

Computes morphology metrics (neurite lengths, branching, etc.) and registers them as a measurement entity linked to the morphology.

**Response:**
```json
{
  "measurement_entity_id": "<uuid>",
  "measurement_kinds": [...],
  "status": "success"
}
```

### Optional: Preview Metrics Without Registering

**Endpoint:** `GET /declared/neuron-morphology-metrics/{cell_morphology_id}/measurement-kinds?morphology_format=swc`

Returns the computed measurement kinds without persisting anything.

### Optional: Get Morphology Metrics (after registration)

**Endpoint:** `GET /declared/neuron-morphology-metrics/{cell_morphology_id}`

Returns the computed metrics for a morphology.

---

## Circuit Registration

### Full Circuit Upload

**Endpoint:** `POST /declared/circuit/register`

**Content-Type:** `multipart/form-data`

> **2 GB per-file limit.** The infrastructure in front of obi-one rejects any single uploaded file above 2 GB before it reaches the app, so the error is not descriptive. For circuits with larger files, use [Large Circuits](#large-circuits-2-gb-remote-validation) below.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✓ | Circuit name |
| `description` | string | ✓ | Description |
| `brain_region_id` | UUID | ✓ | Brain region entity ID |
| `subject_id` | UUID | ✓ | Subject entity ID |
| `build_category` | string | ✓ | e.g. "experimental", "reconstructed" |
| `target_simulator` | string | ✓ | e.g. "NEURON" |
| `circuit_archive` | file | ✓ | `.tar.gz` SONATA circuit archive |
| `scale_override` | string | | e.g. "pair", "microcircuit", "mesocircuit" |
| `parent_circuit_id` | UUID | | Direct parent circuit (the one this was derived from, **not** its root; the root is determined automatically). See "Parent vs. root" below |
| `derivation_type` | string | | Required if parent set: e.g. "customization" |
| `atlas_id` | UUID | | Brain atlas entity ID |
| `license_id` | UUID | | License entity ID (required if `authorized_public=true`) |
| `contact_email` | string | | Contact email |
| `published_in` | string | | Publication reference |
| `experiment_date` | string | | ISO date |
| `contributions` | JSON string | | `{"Agent Name": {"type": "person", "role": "..."}}` |
| `publications` | JSON string | | `{"10.1234/doi": {"type": "..."}}` |
| `overview_image` | file | | Pre-computed overview image (.png/.webp) |
| `sim_designer_image` | file | | Simulation designer image (.png) |
| `authorized_public` | bool | | Make publicly visible (default: false). Keep `false` unless the user is a maintainer (see Important Rules) |
| `dry_run` | bool | | Validate without registering (default: false) |

**Response:**
```json
{
  "circuit_id": "<uuid>",
  "status": "draft",
  "number_neurons": 1000,
  "number_synapses": 500000,
  "number_connections": 10000,
  "scale": "microcircuit"
}
```

**What happens:**
1. Archive is extracted, SONATA config validated
2. Circuit entity created in `draft` lifecycle_status
3. Async validation job triggered (checks SONATA conformance)
4. If validation passes → status becomes `active`

### Trigger Re-Validation

**Endpoint:** `POST /declared/circuit/{circuit_id}/validate?force=false`

Re-triggers validation for a draft circuit. Pass `force=true` to re-validate active/disqualified circuits. Pass `generate_assets=true` to also generate additional assets after successful validation (default `false`).

### Generate Circuit Assets

**Endpoint:** `POST /declared/circuit/{circuit_id}/generate-assets?force=false`

Generates compressed circuit and connectivity matrices for an active circuit.

### Modifying a circuit

There's no dedicated "circuit editing" API — modifications are direct file edits on the extracted circuit tree, done as ordinary code (no obi-one endpoint for this). General pattern:

1. Copy the extracted/staged circuit directory (don't mutate the original — keep it for the control/baseline condition). **If the circuit was staged, its files are symlinks into the read-only `/data` mount** — copy with `shutil.copytree(..., symlinks=False)` (resolve the links to real files) so your edits don't fail with a read-only error or try to write back into the shared store.
2. Locate the target file (e.g. a specific `.hoc` biophysical model file — for multi-cell circuits, be careful to target the right cell type, not e.g. an interneuron file when you meant the pyramidal cell).
3. Apply a targeted, auditable change (e.g. a string replacement of a named conductance value) — assert the expected original value is present before replacing, so a silent no-op can't happen.
4. Record what changed in a small sidecar file (e.g. `modifications.json`) alongside the modified circuit: parameter name, original value, new value, factor, rationale.
5. Save the modification code itself somewhere reproducible.

Split "copy" and "modify" into separate tool calls/steps where the execution environment has a short timeout — a transient failure shouldn't lose both steps at once.

### Registering a (modified) circuit to EntityCore

Use the `obi_one` Python SDK's `register_circuit` helper (pre-installed in the OBI sandbox), which returns the new `Circuit` entity with its `.id`.

> **Parent vs. root.** The **parent** is the direct predecessor a circuit was derived from; the **root** is the topmost ancestor. Never pass an existing circuit's `root_circuit_id` as the new circuit's parent. Do not set `root` yourself: it defaults to `parent.root_circuit_id or parent.id`.
> - Derived **from** an existing circuit (e.g. a modified version): pass it as `parent`, with a `derivation_type`.
> - Same lineage as an existing circuit (e.g. reusing its metadata): look up that circuit's own parent via its derivation link (`client.search_entity(entity_type=models.Derivation, query={"generated__id": "<existing-circuit-id>"})`, take `.used` of the circuit-to-circuit derivation, ignoring `emodel_circuit` ones) and pass it as `parent`. If there is none, register without a parent.

```python
import os
from entitysdk.client import Client
from entitysdk.token_manager import TokenFromEnv
from entitysdk.common import ProjectContext
from entitysdk import models, types
from obi_one.db_sdk.registration.circuit.register import register_circuit

vlab_id    = os.environ.get("OBI_VLAB_ID", "")
project_id = os.environ.get("OBI_PROJECT_ID", "")

client = Client(
    token_manager=TokenFromEnv("OBI_ACCESS_TOKEN"),
    project_context=ProjectContext(virtual_lab_id=vlab_id, project_id=project_id),
    environment=os.environ.get("OBI_ENVIRONMENT", "production"),
)

# Typically reuse brain_region and subject from the source circuit
brain_region = client.get_entity(entity_id="<BRAIN_REGION_ID>", entity_type=models.BrainRegion)
subject      = client.get_entity(entity_id="<SUBJECT_ID>",      entity_type=models.Subject)

result = register_circuit(
    client=client,
    circuit_path="<PATH_TO_MODIFIED_CIRCUIT_DIR>",
    name="<descriptive-name>",
    description="<what changed and why>",
    build_category=types.CircuitBuildCategory.computational_model,
    brain_region=brain_region,
    subject=subject,
    target_simulator=types.TargetSimulator.NEURON,
    skip_additional_assets=True,  # avoids slow asset generation (plots, matrices)
    skip_validation=True,         # existing circuits may have minor SONATA issues
)
print(f"Registered circuit ID: {result.id}")

# Verify it landed
fetched = client.get_entity(entity_id=result.id, entity_type=models.Circuit)
print(f"Confirmed: {fetched.name} — {fetched.id} (neurons={fetched.number_neurons})")
```

Sending raw HTTP requests directly to EntityCore is also fine for reads/simple writes. Prefer `entitysdk` for two reasons:

1. **Path-staging** — when the operation needs to resolve an asset that lives on a given filesystem, or correctly set circuit paths inside simulation configs, `entitysdk`'s [`staging`](https://github.com/openbraininstitute/entitysdk/tree/main/src/entitysdk/staging) module (`stage_circuit`, `stage_simulation`, `stage_simulation_result`, `stage_sonata_from_memodel`) handles that.
2. **Faster asset downloads in the OBI sandbox** — the sandbox has the entitycore private S3 bucket mounted locally at `/data/aws_s3_internal/private/<vlab_id>/<project_id>/assets/...`. A raw `requests.get(.../download)` always goes over the network regardless. `entitysdk`'s `fetch_*` methods (default strategy `local_or_download`) and `stage_*` methods (which call `fetch_*` internally) check that local mount first — a plain filesystem stat — and only fall back to a real download if the asset isn't there. This only works if the client is configured with a local store:
   ```python
   local_store = entitysdk.LocalAssetStore(prefix="/data")
   client = entitysdk.Client(local_store=local_store, token_manager=..., project_context=...)
   ```
   Without this, `entitysdk`'s own `download_*` methods (strategy `download_only`) get no benefit over raw HTTP either — the speedup is specific to `fetch_*`/`stage_*` + a configured `LocalAssetStore`, not `entitysdk` in general. (Source: `openbraininstitute/prod-platform-architecture#219`.)

   **Decision rule — stage over download:** with a `LocalAssetStore` configured, reach for a `stage_*` helper (or `fetch_*`) first and treat `download_*`/raw `requests.get(.../download)` as the fallback for when no local store is mounted. The `FetchFileStrategy` enum captures the spectrum: `link_or_download` (staging default — symlink from the mount, else download), `copy_or_download`, down to `download_only` (what `download_*` forces). The larger the entity, the more this matters: a circuit's `sonata_circuit` directory is effectively un-downloadable over plain HTTP (unbounded S3 prefix, times out), but `stage_circuit` resolves it as cheap symlinks regardless of size. So for circuits specifically, staging isn't just faster — it's the only reliable way to get the full SONATA tree.

### Large Circuits (> 2 GB): remote validation

For circuits with any file above 2 GB (or when the caller lacks the memory/disk/time for in-process validation), use `register_circuit_with_remote_validation` from `obi_one.db_sdk.registration.circuit.remote`. It uploads directly to EntityCore via `entitysdk` (no 2 GB limit), then runs validation and asset generation as remote jobs via `POST /declared/circuit/{id}/validate?generate_assets=true`. It accepts an uncompressed circuit folder or a `.gz` archive (a `.gz` is extracted locally; the compressed asset is regenerated remotely). Returns `(circuit, job_id)`.

| Need | Use |
|------|-----|
| Large circuit, file > 2 GB, or constrained environment (e.g. 2 GB RAM sandbox) | `register_circuit_with_remote_validation` |
| Small circuit, archive < 2 GB, HTTP only | `POST /declared/circuit/register` |
| Full in-process control (debugging, offline full validation) | `register_circuit` / `register_circuit_from_metadata` (see above) |

```python
circuit, job_id = register_circuit_with_remote_validation(
    client=client,                # entitysdk Client WITH a project context
    obi_one_client=obi_one_client,  # httpx.Client, base_url ends in /api/obi-one, Authorization header set
    circuit_path="path/to/circuit_folder_or_config_or_archive.gz",
    name=..., description=..., build_category=..., target_simulator=...,
    brain_region=..., subject=...,  # resolved entities (see obi_one.db_sdk.registration.circuit.resolve)
    dry_run=True,                 # always dry-run first, then repeat with False
)
```

Then poll `lifecycle_status` on the Circuit (`draft` → `active` or `disqualified`) or `GET /declared/task/{job_id}`. Additional assets appear shortly after the circuit becomes `active`.

**Gotchas:**
- No project context on the `client` raises `ValueError` up front (`dry_run=True` works without one).
- The token on `obi_one_client` is fixed at creation; create the client right before registering.
- If starting validation fails, the `draft` circuit stays registered: retry `POST /declared/circuit/{id}/validate?generate_assets=true` rather than re-registering (the name already exists).
- `dry_run` does not run full SONATA validation (MOD/HOC checks); a circuit can still end up `disqualified` and needs fixing and re-registration (or `force=true` re-validation).
- Client-side metadata computation still loads node properties into memory, so very large circuits can still hit memory limits.
- Circuit customization (`/declared/circuit/customize`) has the same 2 GB per-file limit and no workaround yet. Say so explicitly if asked.

---

## EM Cell Mesh Registration

### Upload a Mesh to an Existing Entity

**Endpoint:** `POST /declared/{entity_id}/register-mesh`

**Content-Type:** `multipart/form-data`

| Field | Type | Description |
|-------|------|-------------|
| `file` | file | GLB mesh file |
| `lod_mesh_format` | string | Source format for LOD generation (default: "obj") |

**Response:**
```json
{
  "entity_id": "<uuid>",
  "glb_asset_id": "<uuid>",
  "task_job_id": "<uuid>",
  "status": "pending"
}
```

Automatically triggers an async LOD (Level of Detail) generation job.

### Generate LOD from Existing Mesh

**Endpoint:** `POST /declared/{entity_id}/generate-lod`

Triggers LOD generation for an entity that already has a mesh asset. Prefers OBJ; falls back to GLB.

---

## Metadata Registration

### Contributors (Persons & Organizations)

**Look up:** `GET /declared/contributor?identifier=<orcid-or-ror-url>`

Resolves metadata from ORCID (persons) or ROR (organizations). Returns whether already registered.

**Register:** `POST /declared/contributor?identifier=<orcid-or-ror-url>`

Creates the person/organization entity from the resolved metadata.

**Identifier formats:**
- ORCID: `https://orcid.org/0000-0002-1234-5678`
- ROR: `https://ror.org/03yrm5c26`

**Response (person):**
```json
{
  "id": "<uuid>",
  "pref_label": "Jean-Denis Courcol",
  "given_name": "Jean-Denis",
  "family_name": "Courcol",
  "orcid": "https://orcid.org/..."
}
```

### Subjects (Species/Strain)

**Search:** `GET /declared/subject?name=<name>`

**Register:** `POST /declared/subject`

```json
{
  "name": "Average Rat",
  "description": "Average adult Wistar rat",
  "sex": "male",
  "weight": 300.0,
  "age_value": 7776000,
  "age_period": "postnatal",
  "species_id": "<species-uuid>",
  "strain_id": "<strain-uuid>"
}
```

**Required:** `name`, `description`, `species_id`, `sex`, `age_value`, `age_period`. `strain_id` and `weight` are optional.

> **`age_value` is a duration, not a plain number of days.** The underlying model is a `timedelta` (EntityCore's `SubjectUserUpdate` types it as an ISO-8601 `duration` string), and obi-one's schema exposes it as a bare `number` — which is interpreted as **seconds**. So a 90-day-old rat is `7776000` (90 × 86400), *not* `90`. Passing `90` will not fail validation; it will silently register a 90-second-old animal. Convert explicitly, and state the age in the logbook in the units a reader expects.

**Other fields:** `weight` is in **grams**. `age_min` / `age_max` exist on the entity for expressing an age range instead of a point value.

**Note:** Duplicate detection normalizes names (case-insensitive, strips punctuation).

### Publications

**Register:** `POST /declared/publication/register`

```json
{
  "DOI": "10.1038/s41586-024-07487-2"
}
```

Fetches metadata (title, authors, year, abstract) from Crossref and creates the entity. Returns 409 if already registered.

---

## Electrophysiology Protocol Validation

**Endpoint:** `POST /declared/validate-electrophysiology-protocol-nwb-file`

Validates NWB files containing electrophysiology recordings before they are registered.

---

## Complete Morphology Ingestion Workflow

```
1. Validate file
   POST /declared/test-neuron-file (upload .swc/.h5/.asc)
   → ZIP with converted formats + validation result

2. Register metadata prerequisites (if not already present)
   - POST /declared/subject (species/strain)
   - POST /declared/contributor?identifier=<orcid>
   - POST /declared/publication/register (DOI)
   → collect their IDs for the metadata payload below

3. Register the morphology — entity, asset, metrics and mesh in one call
   POST /api/obi-one/declared/register-morphology-with-calculated-metrics
   multipart: file=<.swc/.h5/.asc>, metadata=<JSON with name, brain_region,
                                               subject, contributions, …>
   → { entity_id, measurement_entity_id, mesh_asset_id, status, morphology_name }

4. If mesh_asset_id came back null, generate the mesh separately
   POST /api/obi-one/declared/convert-morphology-to-registered-mesh/{id}
   → GLB mesh asset created

5. (Optional) Use in simulations or visualization
```

Step 3 replaces what would otherwise be three calls (entity creation, metrics, mesh). Only reach for the individual endpoints when working on a morphology that is **already registered**.

## Complete Circuit Ingestion Workflow

```
1. Register prerequisites
   - Subject (POST /declared/subject)
   - Contributors (POST /declared/contributor)
   - Publications (POST /declared/publication/register)

2. Upload circuit
   POST /declared/circuit/register (multipart: archive + metadata)
   → Circuit created in draft status, validation job triggered

3. Wait for validation
   GET /api/entitycore/circuit/{id}
   → Check lifecycle_status: "draft" → "active" (or "disqualified")

4. Generate additional assets (if needed)
   POST /declared/circuit/{id}/generate-assets
   → Compressed circuit + connectivity matrices

5. Circuit is ready for simulation campaigns
```

## Important Rules

- **Never create morphologies or circuits via EntityCore.** As of today they can only be registered through obi-one `/declared/` endpoints (`register-morphology-with-calculated-metrics`, `circuit/register`). This restriction is specific to these two types — other entities can be created in EntityCore, though the `/declared/` endpoints are preferred where they exist, since they resolve identifiers and generate assets. Reading from EntityCore (e.g. `GET /api/entitycore/circuit/{id}` to poll validation status) is fine — the rule is about creating these two types.
- **Always validate morphology files first** — invalid files will cause downstream failures in mesh generation and simulations.
- **Circuit archives must be valid SONATA** — the async validation job checks conformance and will set status to `disqualified` if it fails.
- **Circuit files above 2 GB** — never use `POST /declared/circuit/register`; use `register_circuit_with_remote_validation` (see Large Circuits).
- **Register circuits with `authorized_public=false` unless the user is a maintainer.** Most users are not. This applies to every registration route (`POST /declared/circuit/register`, `register_circuit`, `register_circuit_with_remote_validation`). Only set it to `true` if the user explicitly says they are a maintainer and want the circuit public.
- **Contributors use persistent identifiers** — ORCID for persons, ROR for organizations. Don't create them with freeform names.
- **Duplicate checking is automatic** — the API returns 409 Conflict if entities already exist.

---

## Working directory

When ingestion runs from the sandbox, all files go under the conversation's topic directory on the persistent volume, per the convention in [[obi-logbook]] — one directory per scientific topic (`smith-2025-ca1-morphologies`, `v1-circuit-upload`), created before any work starts, reused if it already exists:

```
/home/jovyan/<topic>/
├── logbook.md
├── source/             ← the files as received, untouched
├── converted/          ← validated/converted formats returned by obi-one
├── results/            ← metrics, validation reports
├── plots/
└── code/
```

`logbook.md`, `results/`, `plots/`, and `code/` are fixed by [[obi-logbook]]. **Keep `source/` pristine** — never overwrite the received files with converted ones; provenance depends on being able to go back to exactly what was handed over.

## Logbook

Every ingestion session is logged per [[obi-logbook]] — read it for the location, format, and content rules. An ingestion logbook is the provenance record for the data: months later it is what tells a scientist where a morphology or circuit came from and whether it can be trusted.

Record, as the session goes:

- **What is being ingested and why** — the dataset, its scientific purpose, who produced it.
- **Source & provenance** — original source (lab, publication DOI, public repository), contributors (with ORCID/ROR), subject species/strain/age, brain region, experimental protocol behind the data.
- **Scientific properties of each item** — for morphologies: cell type, reconstruction method, and the computed metrics (total dendritic length, branch counts, soma diameter — with units); for circuits: neuron counts, populations, cell types, connectivity scale.
- **Validation outcome in scientific terms** — "3 of 40 morphologies rejected for disconnected neurites", not the HTTP status that reported it.
- **Registered entities** — a provenance table of what now exists in EntityCore: type, name, ID.
- **Caveats** — anything about the data a downstream modeller must know: unusual staining/shrinkage correction, partial reconstructions, mixed protocols, missing metadata.

Do **not** log endpoints, `multipart/form-data` details, tokens, job polling, or 409/422 handling — see the "What never goes in" list in [[obi-logbook]].

## Linking

**Link everything.** Per [[obi-links]], every registered entity and every file mentioned in a reply goes out as a hyperlink — a bare UUID or a bare path is a defect, not a style choice. `{domain}` is `staging.openbraininstitute.org` on staging, `www.openbraininstitute.org` on production, and must match the environment the registration calls ran against — a staging link to a production entity is worse than no link.

- **Registered entities** — `https://{domain}/app/entity/{id}`. This is the main deliverable of an ingestion session: every morphology, circuit, mesh, subject, contributor, and publication that was registered should reach the user as a link, and the provenance table's id cells should be links too. The route resolves any entity type, so no per-type URL is needed.
- **Sandbox files** — source files staged for upload, validation reports, and metric summaries: take the `download_url` from `{obi}:get-sandbox-download-url` and swap `/files/` for `/lab/tree/`, keeping the rest of the path (`home/jovyan` included).
- **Jobs** — ingestion registers entities rather than launching a campaign, so there is normally no workflows activity page to link. If a session goes on to launch one (e.g. skeletonization of registered meshes), link it per [[obi-links]].
