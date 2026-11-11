---
name: obi-logbook
description: Generic skill — the scientific logbook convention for every OBI (Open Brain Institute) scientific workflow. Defines where the logbook lives, when to write to it, and (most importantly) what belongs in it — the science, not the plumbing. Invoke this skill at the start of any OBI scientific session (circuit simulation, disease modeling, single-cell/ME model building, ion channel fitting, skeletonization, data integration, visualization) and keep appending to the logbook as the session progresses. Every OBI scientific skill defers to this one for logbook mechanics and content rules, so read it once per session rather than re-deriving a format.
license: Apache-2.0
---

# OBI Scientific Logbook

Every OBI scientific session keeps a logbook. It is the lab notebook for the work: a colleague who was not in the room should be able to read it and understand **what scientific question was asked, what was done to answer it, what came out, and whether to trust it** — without ever needing to know which API was called.

> **Related generic skills:** [[obi-sandbox-auth]] for credentials and consent flows, [[chat-obi-sandbox-bridge]] for getting files from the OBI sandbox to the user (Claude Chat / Claude Desktop). Neither of those belongs in the logbook's *content* — see "What never goes in" below. [[obi-links]] governs how ids, files, and jobs are turned into links; of those, only **entity links** belong in the logbook — see "Links in the logbook" below. [[obi-research-log]] registers the logbook on the platform as an EntityCore ResearchLog **when the user asks**, and keeps it in sync after that — see "On the platform" below.

---

## The one rule

**Log the science, not the plumbing.**

The logbook is written for a neuroscientist reading it six months later, not for a developer debugging a request. If a sentence would only make sense to someone looking at a terminal, it does not belong in the logbook.

### What goes in

| Category | Examples |
|---|---|
| **Question** | What is being investigated, and why this approach can answer it |
| **Model / data provenance** | Which circuit, morphology, e-model, recording, or mesh was used — name, brain region, species/strain, scale (neuron counts, cell types), and *why it was chosen over the alternatives* |
| **Scientific parameters** | Every biophysical value set or changed: quantity, baseline → new value, **units**, which cell type / compartment / synapse type it applies to, and the **citation** justifying it |
| **Protocol** | Stimulus type and amplitude, duration, temperature, holding potential, number of repetitions/seeds, what was recorded and where |
| **Comparisons** | What is the control, what is the perturbed condition, and what is held identical between them |
| **Results** | The actual numbers with units (firing rates in Hz, latencies in ms, voltages in mV, feature values), plus the figures produced and what each one shows |
| **Interpretation** | What the result means for the question — including when it *contradicts* the hypothesis |
| **Caveats & limitations** | Small N, unvalidated parameter, arbitrary choice, known model limitation, approximation taken |
| **Provenance IDs** | EntityCore IDs of the entities used and produced, in a compact table — these are scientific provenance, not plumbing |
| **Next steps** | What the result suggests trying next |

### What never goes in

Do not write any of these into the logbook:

- HTTP verbs, endpoint paths, status codes, request/response bodies, `curl` commands
- Authentication: tokens, token exchange, refresh, consent URLs, `virtual-lab-id` / `project-id` headers, anything from [[obi-sandbox-auth]] — **never write a token or any part of one into the logbook**
- Tool names, MCP connector names, sandbox lifecycle (kill/restart), polling loops, retries, timeouts
- Bugs, workarounds, and error messages encountered along the way — those are session chatter, and the platform-level ones already live in the relevant skill's Known Issues section
- Intermediate/scratch file paths and download plumbing (see [[chat-obi-sandbox-bridge]])

Two exceptions, both scientific in nature:
1. **Output artifacts** — the final paths of figures, result files, and analysis code are worth recording, since they are what a reader would go open.
2. **A technical fact that changes the scientific reading of a result** — e.g. "the simulation was cut to 500 ms, so late-phase adaptation is not captured". Write it as a scientific caveat, in scientific terms, not as an incident report.

> **Sanity check before appending:** would this line still make sense if the platform were replaced tomorrow by a completely different one? If yes, it is science — write it. If no, leave it out.

---

## Links in the logbook

[[obi-links]] governs how ids, files, and jobs become clickable for the user. Only part of that belongs in the permanent record:

| | In the logbook? |
|---|---|
| **Entity links** — `https://{domain}/app/entity/{id}` | **Yes.** Provenance IDs are science; make the id cells in the provenance table links so a reader can open the entity. Use the domain of the environment the work ran against. |
| **Output file paths** | **Yes, as plain paths** (`{topic_dir}/plots/comparison.png`). They stay valid as long as the volume does; a JupyterLab URL carries the current sandbox's server id and dies with it. Embed figures as markdown images, `![<what it shows>]({topic_dir}/plots/comparison.png)`, so that [[obi-research-log]] can carry them to the platform. |
| **Workflow activity links** | **No.** Session plumbing — hand them to the user in the reply instead. |
| **Download URLs, tokens, consent URLs** | **Never.** Credentials, not provenance — see "What never goes in" above. |

## The sandbox directory convention

This layout governs **all** OBI scientific work on the sandbox, not just the logbook. Every skill that writes a file follows it.

**One directory per topic, at the root of the persistent volume:**

```
/home/jovyan/<topic>/
```

`<topic>` is the scientific subject of the conversation, in kebab-case — `alzheimer-ca1`, `nav1.6-fitting`, `v1-microcircuit`, `thalamic-mesh-skeletons`. It names *what is being studied*, not which workflow or API is being used: everything a conversation produces lands under one topic, even when the work spans several skills. One conversation, one topic directory, one logbook.

`/home/jovyan` is network-mounted (EFS) and survives across sessions and conversations; everything else in the sandbox — and the whole client-side sandbox in Claude Chat — resets. Create and append files there via the OBI sandbox's Python execution, never in the client-side sandbox.

### If the topic directory already exists

Check before creating, and **never write blind into an existing topic directory** — another conversation's results and logbook may be in it.

- **Continuing that work** → reuse the directory and append to its existing `logbook.md`, stating explicitly that you are resuming it. Do not start a second logbook.
- **Unrelated work that happens to collide** → choose a more specific topic name (`alzheimer-ca1-hcn-sweep` rather than a second `alzheimer-ca1`). A name that collides is usually a name that was too vague anyway.

### Inside a topic directory

A fixed backbone, plus input directories named for what they actually hold:

```
/home/jovyan/<topic>/
├── logbook.md          ← always at the topic root
├── <inputs…>/          ← one dir per input kind, named for its contents
├── results/            ← raw outputs; one subdir per condition when comparing
│   ├── <condition-a>/
│   └── <condition-b>/
├── plots/              ← figures
└── code/               ← every script written, for reproducibility
```

`logbook.md`, `plots/`, `code/`, and `results/` are the backbone — always present, always named this way, so any skill and any later conversation can find them without asking. The input directories are workflow-specific: name them for their contents (`circuit-original/`, `circuit-disease/`, `meshes/`, `recordings/`, `morphologies/`). Where a workflow compares conditions, mirror those condition names under `results/`.

Set this up **before any scientific work starts**:

```python
# on the OBI sandbox
import os

topic_dir = "/home/jovyan/<topic>"                 # kebab-case scientific subject of this conversation

if os.path.exists(topic_dir):
    print("EXISTS — resume this topic, or pick a more specific name:")
    print(sorted(os.listdir(topic_dir)))
else:
    for sub in ["results", "plots", "code"]:       # + the input dirs this workflow needs,
        os.makedirs(f"{topic_dir}/{sub}")          # + results/<condition> per condition

logbook_path = f"{topic_dir}/logbook.md"
print(topic_dir)
```

**Use `topic_dir` as the root for every sandbox path for the rest of the conversation.** Build absolute paths from it — never scatter files in `/tmp`, `~`, or a bare `/home/jovyan/outputs`, which either do not survive or cannot be traced back to the work that produced them.

**If no OBI sandbox is in play** (e.g. a pure REST-API session driven from a local machine), keep the identical structure locally under `./obi-sessions/<topic>/` and tell the user where it is. The layout, format, and content rules are all the same.

---

## When to write

Append as you go. **Never reconstruct the logbook at the end of the session** — by then the reasoning behind each choice has been lost, and a session that fails partway leaves nothing behind.

| Moment | Append |
|---|---|
| Session start | Front-matter block + the scientific question / objective |
| A scientific choice is made | What was chosen, and the reason it was chosen over alternatives |
| A parameter is set or changed | Value, units, scope, and citation — before running anything with it |
| A run is launched | What is being run, under what protocol, and what it is expected to show |
| A run finishes | The numbers that came out, with units |
| A figure is produced | Its path and one line on what it shows |
| Interpretation forms | What the result means, including surprises and negative results |
| Session end | Results summary, caveats, next steps; flip `status` to `complete` |

**If the logbook is registered** (its front matter has a `research_log_id`), colleagues may have edited it in the web app: pull their edits before every write, and push at milestones, as in [[obi-research-log]].

Appending is a plain file append on the OBI sandbox:

```python
with open(logbook_path, "a") as f:
    f.write("\n## Circuit Selection\n"
            "- ...\n")
```

**Cite before you use.** Any biophysical parameter that came from the literature gets its reference recorded at the moment it is adopted. An unattributed parameter is a defect, not a shortcut.

---

## Structure

Front matter, then sections in the order the work happened. Sections that do not apply to a given workflow are simply omitted — this is a shape to fill in, not a form to complete.

```markdown
---
topic: <topic, e.g. nav1.6-fitting>
workflow: <what was done, e.g. ion-channel-fitting; list several if the topic spans skills>
objective: <one line — the scientific question>
date: <YYYY-MM-DD>
topic_dir: /home/jovyan/<topic>
status: in-progress | complete
research_log_id:    # set by obi-research-log when the user registers the logbook; empty until then
research_log_env:   # staging | production — the environment research_log_id lives in
---

# <Title> — Session Log

## Objective
<The scientific question, and why this workflow can answer it.>

## Inputs & Provenance
| Entity | Name | Key properties | EntityCore ID |
|---|---|---|---|
| <circuit / morphology / e-model / recording / mesh> | <name> | <region, species, scale…> | [`<uuid>`](https://<domain>/app/entity/<uuid>) |

Why these: <rationale for the selection over alternatives>

## Parameters
| Parameter | Baseline | Used | Units | Applies to | Reference |
|---|---|---|---|---|---|

## Protocol
<Stimulus, duration, recordings, conditions, replicates/seeds, what is held constant between conditions.>

## Runs
| Condition | What it tests | Result entity ID |
|---|---|---|
| <condition> | <what it tests> | [`<uuid>`](https://<domain>/app/entity/<uuid>) |

## Results
<Numbers with units. Figures: path + one line on what each shows.>

## Interpretation
<What this means for the objective. Include negative and surprising results.>

## Caveats
<Limits on how far the result can be pushed.>

## Outputs
- Figures: <paths>
- Data: <paths>
- Code: <topic_dir>/code/

## Next Steps
<What the result suggests doing next.>
```

---

## Presenting it

The logbook is a deliverable, not a byproduct — hand it to the user alongside the figures at the end of the session, and any time they ask what was done.

In Claude Chat / Claude Desktop, finalize the logbook on EFS first, then bridge the finished file over to the user using the pattern in [[chat-obi-sandbox-bridge]]. The copy that reaches the user is a **presentation snapshot**.

### On the platform

Until the user registers it, the canonical copy is `{topic_dir}/logbook.md`, which survives the conversation. **Register it only when the user asks** — to share it with the project, publish it, or see it under Reports — using [[obi-research-log]]. After that, the EntityCore ResearchLog is the canonical copy, since project members can edit it in the web app; `logbook.md` becomes the working copy that [[obi-research-log]] keeps in sync. At the end of an unregistered session you may mention once that the logbook can be registered.

---

## Guardrails

- **One logbook per topic directory.** Resuming earlier work means appending to that topic's existing logbook, not starting a fresh one — say so explicitly when resuming.
- **Write it during the session, not after.**
- **Never log secrets.** No tokens, no consent URLs, no auth headers — see [[obi-sandbox-auth]]. A registered logbook is readable by every member of the project.
- **Register only on request; sync once registered.** See [[obi-research-log]].
- **Record negative results.** A modification that changed nothing is a finding and belongs in the logbook.
- **Units are mandatory** on every numeric value.
- **Never invent a citation.** If a parameter has no source, record it explicitly as an assumption under Caveats.
