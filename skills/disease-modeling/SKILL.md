---
name: disease-modeling
description: Claude Chat / Claude Desktop specific — end-to-end autonomous workflow for modeling a disease or neurological condition (Alzheimer's, Parkinson's, epilepsy, etc.) using a small microcircuit from the Open Brain Institute (OBI) platform. Covers literature search, scientific parametrization, and orchestrating the modify → register → simulate → analyze → present pipeline, plus unprompted logbook maintenance. For all circuit/simulation platform mechanics, this skill defers to [[obi-circuit-simulation]] (how to find, stage, and simulate a circuit, and what the outputs look like) and [[data-integration-api]] (how to modify and register a circuit) — read those skills alongside this one. Invoke whenever the user asks to model any disease with a microcircuit, or to run an end-to-end disease-modeling workflow on OBI.
license: Apache-2.0
---

# Disease Modeling — End-to-End Skill

> **Scope: Claude Chat / Claude Desktop only.** This skill relies on Claude Chat's tool set — `bash_tool`, `create_file`, `present_files` — and the OBI MCP server. It should not be assumed to apply to other Claude surfaces.

> **Platform mechanics live in [[obi-circuit-simulation]] and [[data-integration-api]].** This skill is the scientific/process layer: what question to ask, what literature to gather, how to sequence the workflow. For *how* to actually find/download a circuit or configure/run/poll a simulation, read and follow [[obi-circuit-simulation]]; for *how* to modify and register a circuit, follow [[data-integration-api]] — do not duplicate that mechanical detail here.

> **Generic skills this one builds on — follow them, don't re-derive:**
> - [[obi-logbook]] — the logbook convention: where it lives, when to append, and what belongs in it (the science, never the plumbing). This skill only adds the disease-modeling-specific sections; everything else comes from there.
> - [[obi-sandbox-auth]] — obtaining OBI credentials. **A Neurodamus/CoreNeuron simulation of more than 20 neurons needs the offline-access consent flow before launch** — check this before configuring the campaign. Since this skill favours small circuits (low tens of neurons), a run can cross that line without you noticing; count the neurons rather than assuming "small" means exempt.
> - [[chat-obi-sandbox-bridge]] — getting plots, result files, and the finished logbook back to the user.
> - [[obi-links]] — every entity id, sandbox file, and launched campaign handed to the user goes out as a hyperlink, never as a bare UUID or path. See the Linking section below.

> **Library docs:** Use the **`context7`** MCP server whenever you need up-to-date API documentation for any library used in the sandbox (`entitysdk`, `obi-one`, `bluecellulab`, `bluepyopt`, etc.).

> **OBI server naming:** The OBI MCP server can be connected under any name (e.g. `OBI-local`, `OBI-staging`, `OBI-prod`). Throughout this skill, `{obi}` stands for whatever that connector is named in this conversation.

## Overview

When the user asks to model a disease or neurological condition (Alzheimer's, Parkinson's, epilepsy, or any other) with a microcircuit, execute the following workflow **autonomously and in order**, without waiting for the user to prompt each step. Maintain a logbook throughout.

Default workflow:
```
literature search → circuit selection → model modification → registration to EntityCore
→ baseline + disease experiments → analysis + plots → present results + logbook
```

This default is steered by dialogue with the user — if they specify a particular circuit, disease mechanism, or parameter, use that instead of searching for one.

---

## Prerequisites

Before starting, verify:

1. **OBI MCP server is connected** — `{obi}:execute-python` must be available. If not, tell the user to connect the OBI MCP server in their Claude Desktop settings.
2. **Network egress is enabled** in the Anthropic sandbox (needed to `curl` files from the OBI sandbox into the Anthropic sandbox). If a `curl` step fails with "Host not in allowlist", tell the user to go to **Settings → Capabilities → Allow network egress** and set the domain allowlist to **All domains**.

---

## Step 0 — Set up the topic directory and logbook (do this first, before anything else)

Follow the directory convention in [[obi-logbook]]: one directory per **topic** at the root of the persistent volume, named for the scientific subject in kebab-case — here, the disease and the circuit under study (`alzheimer-ca1`, `parkinson-stn`), not the workflow. Check whether it already exists before creating it: if it does, either resume that work by appending to its existing logbook, or pick a more specific topic name.

```python
# {obi}:execute-python
import os

topic_dir = "/home/jovyan/<topic>"    # e.g. /home/jovyan/alzheimer-ca1

if os.path.exists(topic_dir):
    print("EXISTS — resume this topic, or pick a more specific name:")
    print(sorted(os.listdir(topic_dir)))
else:
    for subdir in ["circuit-original", "circuit-disease",
                   "results/control", "results/disease", "plots", "code"]:
        os.makedirs(f"{topic_dir}/{subdir}")

print(f"Topic dir: {topic_dir}")
```

The input directories (`circuit-original/`, `circuit-disease/`) and the per-condition split under `results/` are this workflow's specialisation of the backbone; `logbook.md`, `results/`, `plots/`, and `code/` are fixed by [[obi-logbook]].

**Use `topic_dir` as the root for every OBI sandbox path for the rest of the conversation.**

Then create the logbook at `{topic_dir}/logbook.md` following [[obi-logbook]] — that skill owns the mechanics (EFS location, append-as-you-go discipline, and the rule that only science goes in, never HTTP/auth/tooling detail). Use the disease-modeling section set below in place of the generic one:

```markdown
---
topic: <TOPIC>
workflow: disease-modeling
disease: <DISEASE_OR_CONDITION>
objective: <the scientific question>
date: <TODAY>
topic_dir: /home/jovyan/<TOPIC>
status: in-progress
---

# <Disease> Modeling — Session Log

## Rationale            <!-- why this disease mechanism, why a circuit model can address it -->
## Literature Parameters
## Circuit Selection
## Model Modifications
## Experiment Configuration
## Results Summary
## Next Steps
```

Create and append to it via `{obi}:execute-python`, never via `bash_tool`/the Anthropic sandbox — the Anthropic sandbox only ever sees the logbook at the very end (Step 4), when it's bridged over for presentation. Never wait until the end to fill it in.

---

## Step 1 — Literature Search

Use Claude's **built-in** search tools — do **not** assume any external search MCP (e.g. Exa) is available:

- **Web search** — for broad queries (e.g. "Alzheimer's microcircuit ion channel changes", "Parkinson's dopaminergic burst firing model", "epilepsy interneuron loss hippocampus")
- **Web fetch** — to read a specific paper URL the user provides or you find
- **Research mode** — for a deep multi-source literature review (takes 5–10 min, use when high confidence in results is needed)

**Target parameter categories** (adapt to the disease at hand, what the user specified, and the circuit available):

| Category     | What to look for                                                                                                                                                                          |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ion channels | Conductance changes for specific channels (e.g. HCN reduction in Alzheimer's CA1, altered NaV kinetics in some epilepsies, KCa/Ca2+ handling changes in Parkinson's dopaminergic neurons) |
| Synaptic     | Receptor ratio or weight changes at specific synapse types (e.g. AMPA/NMDA ratio changes, altered release probability)                                                                    |
| Morphology   | Structural changes (e.g. dendritic spine density reduction)                                                                                                                               |
| Network      | Altered oscillations, connectivity, or excitatory/inhibitory balance                                                                                                                      |

**Guardrail:** For every parameter you use, record the source (authors, year, DOI or URL) in the logbook. Never use an unattributed parameter.

After searching, append findings to logbook:
```
## Literature Parameters
- <parameter>: <change> in <cell type/region> (Author et al. Year, doi:...)
```

---

## Step 2 — Circuit Selection, Modification, Registration, and Simulation

Follow [[obi-circuit-simulation]] for finding and simulating, and [[data-integration-api]] ("Modifying a circuit" and "Registering a (modified) circuit to EntityCore") for modifying and registering — they have the concrete recipes, request shapes, and known-bug list:

1. **Find and download a small microcircuit** from EntityCore. Favor circuits small enough to iterate quickly (single-digit to low-double-digit neuron counts) — small circuits are faster to run and easier to interpret; scale up later if the science calls for it.
2. **Modify the circuit** to apply the literature-derived parameter change(s) from Step 1 — e.g., for Alzheimer's, reducing HCN conductance in CA1 pyramidal cells; for another condition, whatever parameter your literature search identified as disease-relevant. Record the exact modification (parameter, original/new value, rationale, citation) in a `modifications.json` sidecar per [[data-integration-api]]'s pattern, and append it to the logbook.
3. **Register the modified circuit** to EntityCore via `register_circuit` (see [[data-integration-api]]; pass the original circuit as `parent`).
4. **Configure and run two simulation campaigns** — one for the unmodified (control) circuit, one for the modified (disease) circuit — using the same stimulus/protocol for both so the comparison is apples-to-apples. Prefer obi-one's `CircuitSimulationScanConfig` API over writing simulation logic from scratch.
5. **Download the results** (`spikes.h5`, `soma_voltage.h5`) for both campaigns.

Record circuit IDs, registered circuit ID, campaign IDs, and simulation IDs in the logbook as you go.

---

## Step 3 — Analyze and Plot

Compare control vs. disease conditions on the phenomena the literature predicted would change — for example, firing rate, spike timing, voltage traces, or whatever readout is relevant to the mechanism under study. There's no fixed required plot layout; produce whatever comparison (voltage traces, spike rasters, summary statistics) best shows the effect of the modification, given [[obi-circuit-simulation]]'s description of the output file formats.

A minimal illustrative example (not a template to follow exactly):

```python
# {obi}:execute-python
import h5py, numpy as np, matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

topic_dir = "<TOPIC_DIR>"
data = {}
for label in ["control", "disease"]:
    with h5py.File(f"{topic_dir}/results/{label}/spikes.h5") as f:
        population = next(iter(f["spikes"].keys()))  # don't hardcode a population name
        spike_times = f[f"spikes/{population}/timestamps"][:]
    with h5py.File(f"{topic_dir}/results/{label}/soma_voltage.h5") as f:
        population = next(iter(f["report"].keys()))
        voltage = f[f"report/{population}/data"][:]
        t_start, t_stop, dt = f[f"report/{population}/mapping/time"][:]
    data[label] = {"spike_times": spike_times, "voltage": voltage,
                   "time": np.linspace(t_start, t_stop, voltage.shape[0])}

fig, ax = plt.subplots(figsize=(8, 4))
for label, color in [("control", "steelblue"), ("disease", "firebrick")]:
    ax.plot(data[label]["time"], data[label]["voltage"][:, 0], color=color, label=label, lw=0.8)
ax.set_xlabel("Time (ms)"); ax.set_ylabel("mV"); ax.legend()
plt.savefig(f"{topic_dir}/plots/comparison.png", dpi=150)
```

Save all analysis code to `{topic_dir}/code/` for reproducibility.

---

## Step 4 — Bridge Plots to User and Present Logbook

For each plot, bridge from the OBI sandbox to the Anthropic sandbox using the pattern from [[chat-obi-sandbox-bridge]]:

```
# 1. Get signed URL — use the actual topic_dir path
{obi}:get-sandbox-download-url(path="<TOPIC_DIR>/plots/comparison.png")
# -> { url: "...", token: "...", curl_command: "..." }
```

```bash
# 2. bash_tool — pull into Anthropic sandbox
curl -sS -H "Authorization: Bearer <token>" "<url>" \
     -o /mnt/user-data/outputs/comparison.png
```

```
# 3. present_files
present_files(filepaths=["/mnt/user-data/outputs/comparison.png"])
```

Repeat for every plot. Then finalize the logbook **on EFS** (its canonical location — see Step 0), and only then bridge the finished file over for presentation:

```python
# {obi}:execute-python — finalize logbook, still on EFS at topic_dir
with open(logbook_path, "a") as f:
    f.write("""
## Results Summary
- Control: ...
- Disease: ...
- Interpretation: ...

## Next Steps
- ...
""")
```

```
# {obi}:get-sandbox-download-url — bridge the finished logbook, same pattern as plots
{obi}:get-sandbox-download-url(path="<TOPIC_DIR>/logbook.md")
```

```bash
# bash_tool
curl -sS -H "Authorization: Bearer <token>" "<url>" \
     -o /mnt/user-data/outputs/logbook.md
```

```
present_files(filepaths=[
    "/mnt/user-data/outputs/comparison.png",
    "/mnt/user-data/outputs/logbook.md"
])
```

The copy in `/mnt/user-data/outputs` is a presentation-only snapshot — it's fine that it resets between conversations, since the canonical, persistent copy always lives on EFS at `{topic_dir}/logbook.md` and survives independently.

---

## Guardrails

- **Always cite literature** for every biophysical parameter used. Never apply a change without a reference.
- **Log the science, not the plumbing** — per [[obi-logbook]]. The logbook records the mechanism, parameters, protocol, numbers, and interpretation; never endpoints, tokens, sandbox restarts, or workarounds.
- **Simulations run on OBI, not in the Anthropic sandbox.** The Anthropic sandbox is for light glue work (curl, file bridging) only — the logbook itself is written and appended to on EFS (`{topic_dir}/logbook.md`) via `{obi}:execute-python`, and only bridged to the Anthropic sandbox as a final snapshot for presentation (Step 4).
- **Save all code** you write to `{topic_dir}/code/` for reproducibility.
- **One file per `get-sandbox-download-url` call.** For multiple plots, call it once per file or `tar` them first.
- **Tokens are short-lived** — fetch the download URL immediately before curling; do not cache across turns.
- **If curl fails** with "Host not in allowlist" — tell the user to enable **Settings → Capabilities → Allow network egress → All domains** in the Anthropic sandbox.
- **Split long operations into separate tool calls** — e.g. copy and modify a circuit in separate `execute-python` calls so a transient sandbox timeout doesn't lose both steps.
- **If `execute-python` or `execute-shell` are unresponsive or timing out**, call `{obi}:kill-sandbox` (no arguments). This stops the stuck server; the next `execute-python` call will cold-start a fresh one automatically.
- **No fixed plot layout is required** — pick whichever comparison best demonstrates the effect under study; don't feel bound to reproduce any specific figure template.
- For platform-level pitfalls (broken `neuron_set` field, missing `delay` field, correct polling params, etc.), see [[obi-circuit-simulation]]'s Known Issues section.

---

## Logbook Template

Mechanics, discipline, and the science-not-plumbing rule all come from [[obi-logbook]]. What follows is the disease-modeling specialisation of that template — the sections to fill and what belongs in each.

```markdown
---
topic: <TOPIC>
workflow: disease-modeling
disease: <DISEASE_OR_CONDITION>
objective: <the scientific question>
date: <DATE>
topic_dir: /home/jovyan/<TOPIC>
circuit_id: <ENTITYCORE_CIRCUIT_ID>
registered_circuit_id: <NEW_ENTITYCORE_ID>
status: <in-progress | complete>
---

# <Disease> Modeling — Session Log

## Rationale
<The disease mechanism under study, why a circuit model can address it, and what a positive result would look like.>

## Literature Parameters
| Parameter | Baseline | Change | Units | Cell type / region | Reference |
|-----------|----------|--------|-------|--------------------|-----------|
| <parameter> | <value> | <×factor> | <units> | <where it applies> | <Author et al. Year, doi:...> |

## Circuit Selection
- Name / brain region / species / neuron count / cell types
- Why this circuit fits the mechanism and the scale needed
- ID: <uuid>

## Model Modifications
- <parameter>: <original> → <modified> (<factor>), applied to <cell type / compartment>
- Biological reasoning + citation for each

## Experiment Configuration
- Stimulus, amplitude, duration, dt, temperature, recorded variables, seeds
- What is held identical between control and disease conditions
- Control simulation ID / disease simulation ID

## Results Summary
<Numbers with units for both conditions, the difference, what each figure shows, and what it means for the hypothesis — including if it does not support it.>

## Caveats
<Circuit size, missing afferents, unvalidated parameter, simulation duration — anything limiting how far the result can be pushed.>

## Next Steps
<What the result suggests trying next.>
```

---

## Linking

**Link everything.** Per [[obi-links]], every entity, file, and campaign mentioned in a reply goes out as a hyperlink — a bare UUID or a bare path is a defect, not a style choice. `{domain}` is `staging.openbraininstitute.org` on staging, `www.openbraininstitute.org` on production, and must match the environment the work ran against.

In the Step 4 presentation message, the three things worth linking are:

- **The entities** — `https://{domain}/app/entity/{id}` for the source circuit, the registered modified circuit, and the simulation campaigns for both the control and the perturbed condition. Link them at first mention in the summary, and link the id cells in the logbook's provenance table.
- **The launched campaigns** — `https://{domain}/app/virtual-lab/{vlab_id}/{project_id}/workflows?tactivity=simulate&ttype={simulation_type}`, with `{simulation_type}` matching the circuit's scale (`small_microcircuit_simulation` for the small circuits this skill favours; see [[obi-links]] for the full table). Both parameters are required.
- **The sandbox files** — plots, result files, and `logbook.md`: take the `download_url` already fetched for the bridging step and swap `/files/` for `/lab/tree/`, keeping the rest of the path (`home/jovyan` included). Give the link *alongside* the bridged file, not instead of it — the artifact is what the user reads, the link is how they get back to the original.

The logbook itself keeps entity links and plain output paths; JupyterLab URLs, download URLs, and tokens stay out of it.
