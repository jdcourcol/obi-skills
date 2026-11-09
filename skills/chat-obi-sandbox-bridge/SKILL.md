---
name: chat-obi-sandbox-bridge
description: Claude Chat (claude.ai) specific — how to route work between the OBI (Open Brain Institute) compute sandbox and the Anthropic sandbox used by Claude Chat. Relies on Claude Chat's own tools (bash_tool, create_file, present_files) and is custom-built for that sandbox/artifact/egress model — do not assume it transfers to other Claude surfaces. Use whenever a task involves heavy computation, scientific/simulation workloads, large data processing, or any OBI tool (execute-python, execute-shell, get-sandbox-download-url, get-sandbox-upload-url) AND the result needs to reach the user as a file/artifact/plot/download, or the user has uploaded a file that OBI-side compute needs to read. Prefer OBI over running compute directly in bash_tool when an OBI connector is available. Also consult any time a file produced on OBI needs to be shown to the user or turned into an artifact, or a user upload needs to reach OBI.
license: Apache-2.0
---

# Claude Chat × OBI (Open Brain Institute) Sandbox Bridge

> **Scope: Claude Chat (claude.ai) only.** This skill is written specifically against Claude Chat's tool set — `bash_tool`, `create_file`, `present_files`, and its particular sandbox/artifact/egress model. It's custom-built for this environment and shouldn't be assumed to apply anywhere else. If a version for another environment is needed later, it should get its own name/prefix rather than extending this one.

Two separate, non-connected sandboxes are in play. Knowing which one to use, and how to bridge them, is the whole point of this skill.

> **Related generic skills:** [[obi-links]] — a bridged file should also reach the user as a **link** into JupyterLab, built from the `download_url` this skill already fetches (not the same field as the fetchable `fetch_url` — see "Link the file, not just bridge it" below). [[obi-logbook]] for the topic-directory convention every path here assumes.

> **A note on naming:** the OBI (Open Brain Institute) MCP server can be connected under any name the user chooses (e.g. `OBI-local`, `OBI-staging`, `OBI-prod`, `neuroagent` — the server's former and still occasionally used name — or something else entirely) — check the actual tool list in this conversation for the exact prefix in use. The tool *suffixes* are fixed regardless of server name: `execute-python`, `execute-shell`, `get-sandbox-download-url`, `get-sandbox-upload-url`, etc. Throughout this skill, `{obi}` stands in for whatever that connector is named here — e.g. `{obi}:execute-python` means "the `execute-python` tool on the connected OBI server, whatever it's called."

## The two sandboxes

| | **Anthropic sandbox** (`bash_tool`, `create_file`, `str_replace`, `view`) | **OBI sandbox** (`{obi}:execute-python`, `{obi}:execute-shell`) |
|---|---|---|
| What it's for | Light file ops, assembling artifacts, final delivery | Heavy compute, scientific/simulation workloads (obi-one, Ultraliser, bluecellulab, entitysdk, etc.) |
| Persistence | Per-conversation, resets after | `/home/jovyan` persists across sessions — it's a network-mounted volume (EFS-backed), confirmed via `df -h` inside the OBI sandbox. Everything else (`/`, `/tmp`) is ephemeral overlay storage that resets like the Anthropic sandbox. Always write outputs you care about under `/home/jovyan/<topic>/`, per the directory convention in [[obi-logbook]]. |
| Internet egress | None by default (allowlist only) | N/A — has its own preinstalled scientific stack |
| Where the user sees output | `/mnt/user-data/outputs`, surfaced via `present_files` | Nowhere directly — files stay on OBI's filesystem until pulled over |
| Where user uploads land | `/mnt/user-data/uploads/` | N/A — nothing lands here automatically |
| Connects to the other sandbox? | **No.** No route from one sandbox's filesystem to the other's. | **No.** Same constraint, other direction. |

**Rule of thumb:** if the task is computationally heavy, scientific, or needs OBI's preinstalled libraries — do it on OBI, not in `bash_tool`. Only use the Anthropic sandbox for lightweight glue work (downloading a result file, assembling it into an artifact, presenting it, or pushing a user upload over to OBI).

## The bridge: getting a file from OBI to the user

There is no direct sandbox-to-sandbox transfer. The only supported path is:

**OBI sandbox → short-lived fetch link → `bash_tool` curl → Anthropic sandbox disk → `present_files`**

This keeps large binary data off the model's context window entirely (it never appears as text in the conversation — only a small JSON with a URL does). The link is self-contained (no `Authorization` header to carry around) and expires in minutes, scoped to exactly this one file — nothing more is ever placed in the model's context.

### Step-by-step

1. **Run the compute on OBI.**
   Use `{obi}:execute-python` or `{obi}:execute-shell` to do the actual work. Write output file(s) into the conversation's topic directory, following the layout in [[obi-logbook]] — figures to `/home/jovyan/<topic>/plots/`, data to `/home/jovyan/<topic>/results/`. Don't invent ad-hoc output paths; the bridge needs an absolute path anyway, and the convention gives you one that persists and stays traceable.

2. **Get a fetch link for each output file.**
   Call `{obi}:get-sandbox-download-url` with that absolute path. It returns a `fetch_url` and a ready-to-use `curl_command` — no `Authorization` header needed. It also returns `download_url`, which is **not** fetchable on its own; that field only exists to build the JupyterLab link in "Link the file, not just bridge it" below. One call per file — it does not accept a directory or glob. The fetch link expires after `expires_in_seconds` (currently 300); if too much time passes before step 3, just call the tool again for a fresh one.

3. **Download it into the Anthropic sandbox via `bash_tool`.**
   Run the returned `curl_command` inside `bash_tool`, writing straight to `/mnt/user-data/outputs/`, e.g.:
   ```bash
   curl -sS "<fetch_url>" -o /mnt/user-data/outputs/result.png
   ```
   Do this for every file you need to hand back. This is a normal `bash_tool` call — no special handling needed, and since it downloads straight to disk, the file bytes never pass through the model's context.

4. **Present the file(s).**
   Call `present_files` with the path(s) now sitting in `/mnt/user-data/outputs/`. If the extension is one of `.md .html .jsx .svg .mermaid .pdf`, it will also render as a live artifact; anything else shows as a plain downloadable file card.

### Minimal example

```python
# 1. Compute on OBI
{obi}:execute-python(code="""
import matplotlib.pyplot as plt
plt.plot([1,2,3],[4,1,2])
plt.savefig('/home/jovyan/<topic>/plots/plot.png')
""")
```
```
# 2. Get a fetch link for that one file
{obi}:get-sandbox-download-url(path="/home/jovyan/<topic>/plots/plot.png")
# -> { download_url: "...", fetch_url: "...", curl_command: "curl '...' -o plot.png", expires_in_seconds: 300 }
```
```bash
# 3. Pull it into the Anthropic sandbox (bash_tool)
curl -sS "<fetch_url>" -o /mnt/user-data/outputs/plot.png
```
```
# 4. Hand it to the user
present_files(filepaths=["/mnt/user-data/outputs/plot.png"])
```

## The bridge: getting a file from the user to OBI

Files the user uploads through the Claude Chat interface land in the **Anthropic sandbox** at `/mnt/user-data/uploads/` — they never automatically reach the OBI sandbox. If OBI-side compute needs to read one (e.g. hashing, image processing, feeding it into an OBI pipeline), it has to be pushed across explicitly using `{obi}:get-sandbox-upload-url`.

There is no direct sandbox-to-sandbox transfer here either. The only supported path is:

**User upload → `/mnt/user-data/uploads/` → `bash_tool` PUT → short-lived upload link → OBI sandbox filesystem**

### Step-by-step

1. **Pick the destination and confirm the directory exists on OBI first.**
   Send uploads into the conversation's topic directory, per the layout in [[obi-logbook]] — an input directory named for what the file is (`inputs/`, `recordings/`, `meshes/`, `source/`), not a scratch path. A file the user handed over is provenance: it needs to sit with the work that consumed it.

   The upload is written via Jupyter's Contents API under the hood, which does **not** create parent directories — it fails with `[Errno 2] No such file or directory` if the target folder is missing. Run `{obi}:execute-shell(command="mkdir -p /home/jovyan/<topic>/inputs")` before requesting the upload link, or the attempt will fail.

2. **Get an upload link for the destination path.**
   Call `{obi}:get-sandbox-upload-url` with the absolute destination path (e.g. `/home/jovyan/<topic>/inputs/sample-image.jpg`). It returns an `upload_url` and a ready-to-use `curl_command` — PUT the raw file bytes to it directly, no JSON envelope and no `Authorization` header required. It expires after `expires_in_seconds` (currently 300); re-request it if too much time passes before step 3.

3. **PUT the raw file bytes via `bash_tool`.**
   ```bash
   curl -sS -X PUT --data-binary @/mnt/user-data/uploads/sample-image.jpg "<upload_url>"
   ```
   A `200` response with `"uploaded":true` confirms success.

4. **Verify / use the file on OBI.**
   Confirm with `{obi}:execute-shell(command="ls -la <path>")` or go straight into the compute step (`md5sum`, `execute-python`, etc.).

### Minimal example

```bash
# 1. Ensure destination dir exists
{obi}:execute-shell(command="mkdir -p /home/jovyan/<topic>/inputs")
```
```python
# 2. Get an upload link
{obi}:get-sandbox-upload-url(path="/home/jovyan/<topic>/inputs/sample-image.jpg")
# -> { upload_url: "...", curl_command: "curl -X PUT --data-binary @'<local-file>' '...'", expires_in_seconds: 300 }
```
```bash
# 3. PUT it (bash_tool)
curl -sS -X PUT --data-binary @/mnt/user-data/uploads/sample-image.jpg "<upload_url>"
```

## The logbook is one of the files to bridge

Scientific sessions keep a logbook on the OBI sandbox's EFS volume (see [[obi-logbook]]). It is a deliverable, not scratch: at the end of a session, bridge `/home/jovyan/<topic>/logbook.md` over with the same 4-step pattern as the plots, and include it in the `present_files` call. Because it is `.md`, it renders as a live artifact. The canonical copy stays on EFS — what reaches the user is a presentation snapshot.

## Link the file, not just bridge it

Bridging puts a *copy* of the file in the conversation. The original stays on the OBI sandbox, where the user can open it in JupyterLab — so **always give the link too**, in the same message as the artifact. Per [[obi-links]], the `download_url` field `get-sandbox-download-url` returns (not the fetchable `fetch_url`) *is* the link, one replacement away:

```python
lab_url = download_url.replace("/files/", "/lab/tree/", 1)
```

```
download_url:  https://{hub_host}/user/{username}/{server_id}/files/home/jovyan/<topic>/plots/comparison.png
lab_url:       https://{hub_host}/user/{username}/{server_id}/lab/tree/home/jovyan/<topic>/plots/comparison.png
```

The full path is kept, `home/jovyan` included — the Jupyter root is `/`. Keep the prefix for the session and append other paths to it; directory links work the same way (`…/lab/tree/home/jovyan/<topic>/plots/`) and are the compact way to hand over a folder of figures instead of bridging ten files.

Two things to respect: `{server_id}` belongs to the current sandbox, so a link dies after `kill-sandbox` or an idle shutdown — mint it fresh rather than reusing one from an earlier turn. And `/hub/user-redirect/…` is **not** a shortcut here: it resolves to the user's default server, not the named server the sandbox runs as.

## Key constraints to keep in mind

- **One file per `get-sandbox-download-url`/`get-sandbox-upload-url` call.** For multiple files, loop: get-link → curl → repeat, or `tar`/`zip` them first (download side: zip on OBI, pull the archive; upload side: push the archive, then unzip on OBI).
- **Fetch/upload links are short-lived and scoped to exactly one file in one direction** — request one, use it right away, and don't cache or reuse it across turns. If it expires before you get to the `curl`, call the tool again for a fresh link rather than retrying the stale one.
- **If a fetch/upload link returns 401 (expired/invalid) or `execute-python` returns a 401/token-expired error**, just retry: calling `get-sandbox-download-url`/`get-sandbox-upload-url` again mints a fresh link, and any sandbox tool call re-starts the sandbox if it had gone idle. `{obi}:kill-sandbox` is for when the sandbox itself is stuck or unresponsive, not for an expired link.
- **Anthropic sandbox has no general internet egress by default.** The `curl` calls above will fail with `"Host not in allowlist"` unless network egress is enabled. **This is a one-time setup step the user must do themselves**, at **Settings → Capabilities → "Allow network egress"**. Turn the toggle on, then set the **Domain allowlist** dropdown to **"All domains"** (there is no way to allowlist a single specific host — it's an all-or-nothing dropdown, not a URL list). If a `curl` fails with an allowlist error, tell the user to enable this setting rather than retrying blindly.
- **Never treat OBI tool output as a place to dump large data as text.** If a call returns a big blob directly (not via the download-url/upload-url mechanism), that content passes through the model's context — prefer writing results to a file on OBI and using the download-url/upload-url path instead, even for moderately-sized data.
