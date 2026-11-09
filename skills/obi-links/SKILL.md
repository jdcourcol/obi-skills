---
name: obi-links
description: Generic skill — every reference to something in OBI (Open Brain Institute) is handed to the user as a hyperlink, never as a bare UUID or a bare file path. Defines the three link forms — entity UUID → /app/entity/{id}, sandbox file path → the JupyterLab /user/{username}/{server_id}/lab/tree/ URL derived from the sandbox download URL, launched campaign/job → the workflows activity page (with the full list of valid tactivity/ttype values). Read it whenever a reply is about to mention an entity id, a file written in the sandbox, or a job that was just started — every OBI skill defers to this one for link construction, so read it once per session rather than re-deriving URLs.
license: Apache-2.0
---

# Linking OBI results

A UUID in a reply is a dead end. A path in a reply is a dead end. The same UUID or path as a hyperlink is one click from the thing itself — and building that link never requires an extra tool call, only string construction from values already in hand.

> **Related generic skills:** [[obi-logbook]] for what belongs in the logbook, [[chat-obi-sandbox-bridge]] for actually moving a sandbox file to the user (Claude Chat / Claude Desktop). Linking a file is *not* a substitute for bridging it — a link opens it in JupyterLab, bridging puts it in the conversation. Do both when the file is a deliverable.

---

## The one rule

**Every time you refer to something in OBI — an entity, a sandbox file, a launched job — write it as a link.** Not "when it seems useful", not "when the user asks for it": by default, always. A bare UUID or a bare path in a reply is a defect.

Three things get linked, each with its own form:

| Thing | Link form |
|---|---|
| An entity (circuit, ME-model, morphology, campaign, model, recording…) | `https://{domain}/app/entity/{entity_id}` |
| A file or directory in the OBI sandbox | `https://{hub_host}/user/{username}/{server_id}/lab/tree/{path minus leading slash}` — from `download_url`, `/files/` → `/lab/tree/` |
| A job / campaign that was just launched | `https://{domain}/app/virtual-lab/{vlab_id}/{project_id}/workflows?tactivity={activity}&ttype={type}` |

`{domain}` is `staging.openbraininstitute.org` for staging and `www.openbraininstitute.org` for production. **It must match the environment the work actually ran against** — a staging link to something registered in production (or vice versa) resolves to "not found" or, worse, to a different entity.

### How to apply it

- **Before sending any reply that mentions OBI work, scan it**: every UUID, every `/home/jovyan/…` path, every "the campaign has been launched" — each one is a link you owe the user.
- **Use markdown links with meaningful text** — `[L5 PC circuit](…)`, `[kv1_1_temperature_comparison.png](…)`, `[watch the campaign](…)` — not raw URLs pasted inline.
- **Link on first mention** in a reply. Repeating the same link in every sentence is noise; omitting it entirely is worse.
- **Don't wait to be asked**, and don't offer to produce a link later ("let me know if you want the URL") — just include it.
- **Never invent an id to link.** The link is a wrapper around a value you actually obtained; if you do not have the id or path, say so plainly rather than fabricating a plausible URL.
- **If a link cannot be built** (unknown environment, no vlab/project in context), say which piece is missing instead of silently falling back to a bare id.

---

## 1. Entities → `/app/entity/{id}`

```
https://{domain}/app/entity/{entity_id}
```

Every EntityCore id works here — circuits, ME-models, e-models, cell morphologies, EM cell meshes, ion channel models and recordings, simulation and skeletonization campaigns, simulations, meshes, subjects, publications. The route is a catch-all resolver: the frontend looks up the entity's type and redirects to the correct type-specific detail page, so **no type, vlab, or project is needed in the URL**.

Link an entity when:

- it was **found or selected** as an input ("simulating on [this L5 PC circuit](…)"),
- it was **created or registered** ("registered the modified circuit → [`a1b2…`](…)"),
- it appears in a **provenance table** — link the id cell itself.

There is also a longer, type-specific detail route (`/app/virtual-lab/{vlab_id}/{project_id}/data/view/{type}/{id}/overview`) used by some skills for a specific tab. Prefer the short `/app/entity/{id}` form unless a particular section of the detail page is the point.

---

## 2. Sandbox files → JupyterLab

A file in the sandbox opens in JupyterLab at the **same single-user server URL the sandbox tools already use**:

```
https://{hub_host}/user/{username}/{server_id}/lab/tree/{absolute path, minus the leading slash}
```

**Keep the whole path, `home/jovyan` included.** The Jupyter root is `/`, not `/home/jovyan`.

### Build it from `download_url` — don't assemble it from parts

`{obi}:get-sandbox-download-url` returns exactly the same prefix with `/files/` in place of `/lab/tree/`:

```
https://{hub_host}/user/{username}/{server_id}/files/home/jovyan/<path>
```

so the whole construction is one replacement:

```python
lab_url = download_url.replace("/files/", "/lab/tree/", 1)
```

That is also how you learn `{hub_host}`, `{username}`, and `{server_id}` in the first place — there is no other reliable source for them. Make the call once, keep the prefix for the rest of the session, and append other paths to it.

A confirmed-working link on staging:

```
https://jupyterhub.staging.openbrainplatform.com/user/jdcourcol/77ae1e5c417cfad5ae6774cbd359afefce648cda71fde68a84a7da22a1149de/lab/workspaces/auto-Q/tree/home/jovyan/kv1_1-temperature-effects/plots/kv1_1_temperature_comparison.png
```

The extra `/lab/workspaces/auto-Q/` in that example is JupyterLab's own auto-created workspace — what the browser's address bar shows once Lab is open in more than one tab. `…/lab/tree/<path>` addresses the same file and Lab redirects to a workspace URL by itself, so use the plain `/lab/tree/` form and never invent a workspace name.

### Do not use `/hub/user-redirect/`

It looks attractive because it drops the username and server id, but it **does not reach the sandbox**: the sandbox is spawned as a JupyterHub *named* server (that hash is its id), while `user-redirect` resolves to the user's default server, which is a different — usually not running — thing. Build from `download_url` instead.

### Notes

- **The link is bound to the current sandbox instance.** `{server_id}` changes after `{obi}:kill-sandbox` or an idle shutdown, and links minted earlier stop working. Mint them fresh in the session you hand them out in; never copy one from an old transcript, and keep them out of `logbook.md`, which outlives the server (record the plain path there).
- **Link files under `/home/jovyan`** — the EFS-backed home where all work belongs (see [[obi-logbook]]). `/tmp` is addressable but vanishes with the container, so linking it is pointless.
- **Directories work too** — `…/lab/tree/home/jovyan/<topic>/plots/` opens the folder in the file browser. Prefer one directory link over ten file links.
- **The link opens for that user only**, when logged in. It is not shareable with a colleague.

### If the server is not running

The link does not break, it stalls: JupyterHub answers a request for a stopped server with `/hub/spawn-pending` or a "your server is starting / not running" page instead of the file. Once the server is up, Lab opens at the requested path.

So the rule is **mint the link in the turn you hand it over**. Any sandbox tool call — `{obi}:execute-python`, `{obi}:execute-shell`, `{obi}:get-sandbox-download-url` — starts the sandbox if it is down and gives back a URL carrying the live `{server_id}`; build the link from that, not from an id captured earlier in the transcript. If the user reports a "server not running" page, that is not a bad path: let the server start (or make any sandbox call), then re-issue the link.

The file itself is never at risk — `/home/jovyan` is EFS-backed and outlives every server, so a re-minted link with a new `{server_id}` resolves the same path.

---

## 3. Launched jobs → the workflows activity page

After starting a campaign or job, link the activity listing it will show up in, so the user can watch status without hunting through the UI:

```
https://{domain}/app/virtual-lab/{vlab_id}/{project_id}/workflows?tactivity={activity}&ttype={type}
```

- `{vlab_id}`, `{project_id}` — the virtual lab and project the job was launched into. These are the same values as the `virtual-lab-id` / `project-id` request headers, i.e. `$OBI_VLAB_ID` / `$OBI_PROJECT_ID` in the sandbox; see [[virtual-lab-manager-api]]. Read them from the environment rather than from memory — a tool call can be redirected to another project, and a link built from stale ids points at the wrong workspace.
- `{tactivity}` — the activity tab: `build`, `simulate`, `extract`, or `process`.
- `{ttype}` — the workflow's target entity type, from the table below.

**Always send both parameters.** With `ttype` alone the page keeps its default activity tab (`build`) and the listing will not show a simulate/extract/process job.

### Valid `tactivity` / `ttype` pairs

| `tactivity` | `ttype` | Job it lists |
|---|---|---|
| `build` | `ion_channel_modeling_campaign` | Ion channel fitting campaign |
| `build` | `memodel` | ME-model build |
| `build` | `single_neuron_synaptome` | Synaptome build |
| `build` | `em_synapse_mapping_campaign` | Electron-microscopy circuit build |
| `build` | `extracellular_recording_array_campaign` | Extracellular recording array build *(feature-flagged)* |
| `simulate` | `ion_channel_model_simulation` | Ion channel model simulation |
| `simulate` | `single_neuron_simulation` | Single neuron simulation (legacy) |
| `simulate` | `single_neuron_synaptome_simulation` | Synaptome simulation |
| `simulate` | `single_neuron_circuit_simulation` | Single-neuron circuit simulation |
| `simulate` | `me_model_circuit_simulation` | ME-model circuit simulation |
| `simulate` | `paired_neuron_circuit_simulation` | Paired-neuron circuit simulation |
| `simulate` | `small_microcircuit_simulation` | Small microcircuit simulation |
| `simulate` | `microcircuit_simulation` | Microcircuit simulation |
| `simulate` | `region_circuit_simulation` | Brain-region circuit simulation *(feature-flagged)* |
| `simulate` | `whole_brain_circuit_simulation` | Whole-brain circuit simulation |
| `extract` | `circuit_extraction_campaign` | Circuit extraction campaign *(activity is feature-flagged)* |
| `process` | `skeletonization_campaign` | EM mesh skeletonization campaign |

Types that exist in the frontend's catalogue but whose workflows are currently disabled — `me_model_circuit`, `paired_neuron_circuit`, `small_micro_circuit`, `micro_circuit`, `metabolism`, `ngv_unit`, `ngv_circuit`, `brain_region`, `brain_system`, `whole_brain` — have no jobs to list; don't build links to them.

> Source of truth in `core-web-app`, if a type is missing or a flag has changed: query keys in `src/ui/segments/workflows/elements/workflow-activity.tsx` (`tactivity` / `ttype`), the workflow registries in `src/ui/segments/workflows/config/activities/{build,simulate,extract,process}.ts`, and the type values in `src/api/entitycore/types/extended-entity-type.ts`.

### Also link the campaign entity

The activity page lists jobs by type; it does not deep-link one job. So pair the listing link with the campaign's own entity link:

> Launched the campaign ([`c4f9…`](https://staging.openbraininstitute.org/app/entity/c4f9…)) — follow progress on the [simulation activity page](https://staging.openbraininstitute.org/app/virtual-lab/{vlab}/{project}/workflows?tactivity=simulate&ttype=small_microcircuit_simulation).

---

## In the logbook

[[obi-logbook]] admits provenance IDs and output artifact paths as science, and excludes plumbing. Applied to links:

- **Entity links: yes.** Link the ids in the provenance table — they stay valid indefinitely and take a reader straight to the data.
- **Output files: the plain path, not the link.** JupyterLab URLs carry the current server id and die with it; the path stays valid for as long as the volume does.
- **Download/upload fetch links: never.** Those are short-lived, single-file credentials, not provenance. Workflow-activity links are session plumbing too — they belong in the reply, not the record.
