---
name: obi-research-log
description: Registers a topic's sandbox logbook (see obi-logbook) as an EntityCore ResearchLog so the project's members can read and edit it under Reports, and keeps the two copies in sync afterwards — pulling edits made in the web app, merging them with the agent's own, and pushing the result back. Use ONLY when the user asks to register, publish, share, save to the platform, or push their logbook / research log / report — never on your own initiative — and, once a topic's logbook is registered (its front matter has a research_log_id), every time that logbook is about to be written to or the session ends. Also use when the user asks whether their logbook is already on the platform, or wants a registered research log's web edits brought back into the sandbox.
license: Apache-2.0
---

# OBI Research Log — registering and syncing the logbook

[[obi-logbook]] defines the logbook: where it lives (`{topic_dir}/logbook.md`) and what goes in it. This skill takes that file to EntityCore as a **ResearchLog** entity, where the members of the project can read and edit it in the web app (Reports), and keeps the sandbox copy and the platform copy from drifting apart.

> **Related skills:** [[obi-logbook]] for content rules (they apply unchanged — and matter more, since a registered log is visible to the whole project), [[obi-links]] for entity links, [[obi-sandbox-auth]] for credentials, [[virtual-lab-manager-api]] for which lab and project the log belongs to.

---

## When

| Situation | Do |
|---|---|
| The user asks to register / publish / share / push the logbook | Register it (§2), then sync (§3) |
| The logbook's front matter has **no** `research_log_id` and the user has not asked | **Nothing.** Never register on your own. You may mention once, at session end, that it can be registered. |
| The front matter **has** a `research_log_id` | The user already opted in for this topic: **sync before every write** to the logbook (pull, §3), and **push at milestones** — after results, after interpretation, at session end — without asking. |
| The user asks "is this on the platform?" | Check the front matter, then §2 step 2 — report, do not register |

**One research log per topic directory**, exactly as there is one logbook per topic directory. Never create a second ResearchLog for a topic that already has one.

---

## 1. What a ResearchLog is

- An EntityCore entity of type `research_log` with a **name**, a **description**, and the usual project scoping (private to the project unless published). No brain region, species, or license.
- Its text is an asset labelled `research_log_document` (`text/markdown`). Assets are **never overwritten**: every save is a *new* asset, and the web app shows the newest, chosen by the timestamp in its file name, **`logbook-<epoch-ms>.md`**. Always use that name — any other name may not be recognized as the newest.
- Figures go in an optional **directory** asset labelled `research_log_files`. The markdown references them by **relative** path (`![](figures/fig1.png)`).
- Inputs and outputs of the session are linked with **Derivations**, the ResearchLog always on the *generated* side:
  - `research_log_input` — `used` = an entity the session started from (circuit, morphology, recording…)
  - `research_log_output` — `used` = an entity the session produced (a registered circuit, a simulation result…)
- Only the log's creator or a project admin may delete an asset or a derivation. Anyone in the project may add them, and may edit the text in the web app — **which is why the sandbox copy is no longer the only master once the log is registered.**

### Which copy is the master

| | Master | The other copy |
|---|---|---|
| Before registration | `{topic_dir}/logbook.md` | — |
| After registration | the EntityCore ResearchLog (newest `research_log_document`) | `logbook.md` is a working copy, pulled before each write and pushed at milestones |

---

## 2. Registering (only on the user's request)

### State kept in the topic directory

Registration adds two fields to the logbook's front matter (so a later conversation resuming the topic knows the log exists):

```yaml
research_log_id: <uuid>
research_log_env: staging | production   # the id means nothing in the other environment
```

and a hidden sync directory, which never goes to EntityCore:

```
{topic_dir}/.research-log/
├── state.json   ← {"id", "env", "vlab_id", "project_id", "asset_id", "hash", "files_asset_id"}
└── base.md      ← the exact text last pushed or pulled (the merge base)
```

### Steps

1. **Front matter already has `research_log_id`** → the log is registered. Fetch it (§3 snippet). If it is gone or unreadable, **tell the user** and ask whether to register a new one — do not silently recreate it. If `research_log_env` differs from the current environment, stop and say so.
2. **No id** → look for an existing log the user may have made for this work, in the current project:
   - by name: `name__ilike` on the logbook title;
   - by input: research logs whose `research_log_input` uses the topic's main input entity (`generated_derivation__used_id=<id>&generated_derivation__derivation_type=research_log_input`).

   If a plausible match turns up, **show it to the user and ask** whether this logbook is that log. A wrong guess merges two unrelated logs.
3. **Nothing found, or the user says it is new** → create the ResearchLog in the topic's lab and project:
   - `name` = the logbook title (the `# <Title>` heading, without "— Session Log")
   - `description` = the front matter's `objective`
4. Write `research_log_id` and `research_log_env` into the front matter, create `.research-log/`, then **push** (§4) and **link** (§5).
5. Give the user the link to the log, per [[obi-links]]: `https://{domain}/app/entity/{research_log_id}`.

```python
# on the OBI sandbox — client setup shared by every snippet below
import os, json, hashlib, time, io
from entitysdk import Client, ProjectContext, models
from entitysdk.token_manager import TokenFromEnv

env = os.environ.get("OBI_ENVIRONMENT", "staging")
ctx = ProjectContext(virtual_lab_id=os.environ["OBI_VLAB_ID"], project_id=os.environ["OBI_PROJECT_ID"])
client = Client(token_manager=TokenFromEnv("OBI_ACCESS_TOKEN"), project_context=ctx, environment=env)

topic_dir  = "/home/jovyan/<topic>"
logbook    = f"{topic_dir}/logbook.md"
sync_dir   = f"{topic_dir}/.research-log"
```

```python
# step 2 — look for an existing log
title = "<logbook title>"
for rl in client.search_entity(entity_type=models.ResearchLog, query={"name__ilike": title}, limit=10):
    print(rl.id, rl.name, rl.creation_date)
```

```python
# step 3 — create
rl = client.register_entity(models.ResearchLog(name=title, description="<objective>"))
print(rl.id)
```

`OBI_ACCESS_TOKEN` expires in minutes — if a call returns 401, mint a fresh token as in [[obi-sandbox-auth]] and rebuild the client.

---

## 3. Pulling — before every write once registered

Find the newest `research_log_document` on the platform and compare it with what was last synced:

```python
def newest_document(client, rl_id):
    docs = [a for a in client.get_entity_assets(rl_id, entity_type=models.ResearchLog)
            if a.label == "research_log_document"]
    def stamp(a):  # logbook-<ms>.md; anything else sorts as oldest
        try: return int(a.path.rsplit("logbook-", 1)[1].split(".")[0])
        except (IndexError, ValueError): return -1
    return max(docs, key=stamp) if docs else None

def h(text): return hashlib.sha256(text.encode()).hexdigest()

state = json.load(open(f"{sync_dir}/state.json"))
base  = open(f"{sync_dir}/base.md").read()
local = open(logbook).read()
doc   = newest_document(client, state["id"])
remote = client.fetch_content(entity_id=state["id"], entity_type=models.ResearchLog,
                              asset_or_id=doc.id).decode() if doc else base
remote = remote.replace("](figures/", f"]({topic_dir}/plots/")   # undo §4's link rewrite
```

Then decide — **compare content, not only asset ids** (a browser autosave makes a new asset even when nothing changed):

| Remote vs base | Local vs base | Do |
|---|---|---|
| same | same | nothing |
| same | changed | nothing now; push at the next milestone |
| **changed** | same | **pull without asking**: overwrite `logbook.md` with `remote`, update `base.md` and `state.json` |
| **changed** | changed | **three-way merge** (below) |

```bash
# three-way merge on the sandbox; base.md is the common ancestor
cp logbook.md /tmp/local.md
git merge-file -p /tmp/local.md .research-log/base.md /tmp/remote.md > /tmp/merged.md; echo "conflicts: $?"
```

- **0 conflicts** → the merged text becomes `logbook.md`; push it (§4) right away so the web copy has the agent's lines too.
- **Conflicts** → do not push. Show the user the conflicting passages (both versions, side by side) and ask which to keep. The people editing in the web app are colleagues; never resolve their text away silently.

After any pull, tell the user in one line that web edits were brought in, and by whom if the asset's creator is known.

---

## 4. Pushing — at milestones once registered

Push only after a pull (§3) left `logbook.md` = what you intend to publish.

1. **Rewrite figure links** to the files folder. In `logbook.md`, a figure embedded as `![…]({topic_dir}/plots/fr.png)` becomes `![…](figures/fr.png)` in the pushed text, which the web app resolves against `research_log_files`. Only link targets are rewritten; plain paths in prose stay as they are. The sandbox copy keeps its paths — the rewrite happens on the text being uploaded. The same rewrite must be undone on pull (`figures/` → `{topic_dir}/plots/`) so the merge base stays comparable.
2. **Upload the text** as a new `research_log_document` named `logbook-<epoch-ms>.md`.
3. **Delete the previous document** you pushed (`state["asset_id"]`) — try it, and **ignore a refusal**: only the creator or a project admin may delete.
4. **Figures**, only if the set referenced by the log changed: upload a new `research_log_files` directory holding every referenced figure under `figures/`, then try to delete the previous one (`state["files_asset_id"]`). Directory assets cannot be appended to.
5. Update `base.md` (the text as it is in the sandbox, before the path rewrite) and `state.json`.

```python
import re
text = open(logbook).read()
published = text.replace(f"]({topic_dir}/plots/", "](figures/")   # link targets only
asset = client.upload_content(
    entity_id=state["id"], entity_type=models.ResearchLog,
    file_content=io.BytesIO(published.encode()), file_name=f"logbook-{int(time.time()*1000)}.md",
    file_content_type="text/markdown", asset_label="research_log_document")
if state.get("asset_id"):
    try: client.delete_asset(entity_id=state["id"], entity_type=models.ResearchLog, asset_id=state["asset_id"])
    except Exception: pass   # not ours to delete — fine
state.update(asset_id=str(asset.id), hash=h(text))
open(f"{sync_dir}/base.md", "w").write(text)
json.dump(state, open(f"{sync_dir}/state.json", "w"))
```

```python
figs = sorted(set(re.findall(r"\]\(" + re.escape(topic_dir) + r"/plots/([^)\s]+)\)", text)))
if figs:
    files = client.upload_directory(
        entity_id=state["id"], entity_type=models.ResearchLog, name="files",
        paths={f"figures/{f}": f"{topic_dir}/plots/{f}" for f in figs},
        label="research_log_files")
```

Do **not** push on every append: each push is a new asset and a new version in the web app. Milestones only.

---

## 5. Linking inputs and outputs

From the logbook's provenance tables ([[obi-logbook]] Structure):

| Logbook section | Derivation type |
|---|---|
| **Inputs & Provenance** rows | `research_log_input` |
| **Runs** result entity IDs, and any entity registered during the session | `research_log_output` |

Before adding, read the links that already exist and add only the missing ones — resumed sessions must not duplicate them:

```python
have = {(d.used.id, d.derivation_type) for d in client.get_entity_derivations(
            entity_id=state["id"], entity_type=models.ResearchLog, derivation_type="research_log_input")}
# …same for research_log_output; then for each missing (entity_id, type):
client.register_entity(models.Derivation(
    used=client.get_entity(entity_id=eid, entity_type=models.Entity),
    generated=client.get_entity(entity_id=state["id"], entity_type=models.ResearchLog),
    derivation_type=dtype))
```

Add links as entities appear; never remove one (removal is restricted to the creator or an admin, and an output linked by mistake should be raised with the user instead). A linked entity must be public, or private in the log's project.

---

## Live co-editing (when deployed)

A live co-editing server for research logs is being prototyped (`research-log-collab`). While a log is open in someone's browser, that server holds the text and the browser saves it. **A document uploaded directly to EntityCore while the log is open is not picked up, and the next browser save supersedes it** — the push in §4 can then be lost.

Until the server is deployed, §3–§4 are the method; the risk is real only when someone is editing at the same moment, so push at milestones rather than continuously. Once it is deployed, prefer its API for a registered log:

- `GET /research-log/<id>` — the live text, whether it is saved, and who has it open (replaces §3's fetch)
- `POST /research-log/<id>/edit` with `{"author": "<agent name>", "edits": [...]}`, each edit `{"old", "new"}` (quoted exactly once in the live text) or `{"append": "<markdown>", "under": "<heading>"}` — [[obi-logbook]]'s "append as you go" maps directly onto `append … under`. A 409 means read again and retry.

and keep §4's direct upload as the fallback when the server is unreachable, warning the user that anyone with the log open will overwrite it.

---

## Guardrails

- **Register only when the user asks.** Once registered, syncing is automatic for that topic.
- **One ResearchLog per topic directory.** Check the front matter and search before creating.
- **Pull before you write; push at milestones.**
- **Pull web edits without asking** when only the remote changed — say that you did.
- **Never resolve a merge conflict silently.** Show both sides and ask.
- **Never upload `.research-log/`, tokens, or anything [[obi-logbook]] keeps out of the logbook.** A registered log is visible to every member of the project, and to everyone if the project is published.
- **Match the environment.** A staging `research_log_id` is meaningless in production; links use the domain of the environment the log lives in.

## Not verified yet

Written from the EntityCore design (`docs/design/2026-10-research-log-entity.md`), `entitysdk` 0.19.x, and core-web-app#2062/#2070, without a run against a live environment. Check on a staging test log before relying on it:

- that the sandbox's installed `entitysdk` has `models.ResearchLog` and accepts the `research_log_document` / `research_log_files` labels and the two derivation types;
- that `register_entity(models.Derivation(...))` is the right way to create a derivation, versus `POST /derivation`;
- how the web app picks among several `research_log_files` directory assets, if the previous one could not be deleted.
