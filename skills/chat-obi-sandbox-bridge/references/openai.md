# File transfers in ChatGPT, Codex, and clients with local file tools

## Choose a supported delivery route

Inspect the current tool inventory before transferring a file. Do not assume a
shell, network access, a fixed upload/output directory, `bash_tool`,
`present_files`, or a live HTML artifact renderer is available.

1. Prefer a connector's native file/attachment or preview capability if it is
   actually exposed and supports this file and its authentication. Use its
   documented arguments; do not invent a tool or attachment API.
2. If the client exposes local execution, file access, and an allowed network
   transfer, download or upload using those capabilities as described below.
3. If neither is available, provide the authenticated user's JupyterLab link
   built by [obi-links](../../obi-links/SKILL.md). Explain that the file remains
   on OBI and has not been attached to the chat. For an input, ask the user to
   place it in the topic's OBI `inputs/` directory through JupyterLab and confirm
   its location, then verify it with an OBI tool. Continue independent work.

Do not recommend Claude's network-egress settings in ChatGPT or Codex. If a
transfer is blocked, follow the current host's supported network/approval flow
or use the JupyterLab route. Respect host media-display restrictions; do not
download media to bypass a blocked preview.

## OBI result → chat

1. Write and verify the output on OBI under `/home/jovyan/<topic>/plots/` or
   `results/`, and finalize `/home/jovyan/<topic>/logbook.md` there. Keep analysis
   code under `code/`. Use OBI's execution tools for this work.
2. Call `{obi}:get-sandbox-download-url` with the absolute path of one file.
   Fetch the bytes through its returned `fetch_url`. This ticket URL needs no
   Authorization header and expires after `expires_in_seconds` (currently 300).
   Keep `download_url` for constructing the user's JupyterLab link.
3. For a local transfer, choose the client output directory from the current
   environment's instructions. In a desktop workspace, use its deliverables
   directory; in a cloud runtime, use the location supported by that runtime's
   attachment tools. Create the directory and choose a destination filename.
   Do not assume `/mnt/user-data/outputs` or `/mnt/data` exists.
4. Download `fetch_url` through the available execution tool. Inspect the
   returned `curl_command` and verify its destination before use. For curl, use
   `--fail --silent --show-error` and write bytes directly to the chosen output
   path. Quote URLs and paths safely. Keep the ticket URL private and avoid
   verbose logs and shell tracing.
5. Verify HTTP success, a nonempty file, and the expected file type, so an HTML
   login/error page is not handed over as an image or data file. Present it with
   the client's file or preview capability. With local Markdown file delivery,
   use an absolute path; for cloud attachments, use only the file reference
   returned by the runtime. HTML can be delivered as a file even if it cannot
   run inline.
6. Include the fresh JupyterLab link to the canonical OBI file. Construct it
   from the returned download URL's path by replacing `/files/` with
   `/lab/tree/`; retain `home/jovyan`. Remove query/fragment credentials rather
   than exposing them. This link requires the user's OBI browser session and is
   not a public sharing link.

Repeat per file, including the finished logbook. For many outputs, create an
archive on OBI and transfer that single file, then also link the plots folder.

On expired transfer credentials, request a fresh URL and retry once. If sandbox
authentication itself fails, consult `obi-sandbox-auth`; a restart changes
server URLs and discards in-memory state, so preserve work on the persistent
volume before following its recovery procedure. Do not loop restarts for a
client network or permission error.

## User attachment → OBI input

1. Obtain the attachment's real local path or file handle from the available
   host tools. Never infer a path from its display name. If the host cannot
   expose the bytes to a supported transfer tool, use the JupyterLab input
   route above rather than putting base64 into the conversation.
2. Create `/home/jovyan/<topic>/inputs/` on OBI with `execute-python` or
   `execute-shell`. The Jupyter Contents API does not create missing parent
   directories. Check for an existing destination before writing so an input
   is not silently overwritten.
3. Request `{obi}:get-sandbox-upload-url` for the absolute destination path.
   The returned `upload_url` authorizes a PUT of the raw file bytes. It expires
   after `expires_in_seconds` (currently 300).
4. In the client execution environment, send the local file bytes directly to
   `upload_url`, using the returned `curl_command` as a template. For example,
   substitute the actual URL and local path in this command.

   ```bash
   curl --fail --silent --show-error -X PUT --data-binary @'<local-file>' '<upload_url>'
   ```

   Quote the URL and local path safely. The server performs the Jupyter
   base64/JSON conversion and supplies authorization internally. Keep the
   ticket URL private and avoid verbose logs and shell tracing.
5. Require HTTP success and `uploaded: true` in the response. Verify the file
   on OBI by comparing its byte size and, when practical, checksum before
   launching a workflow that depends on it. Preserve the input with the
   topic's provenance. On expiration, request a fresh ticket and retry once.

## Small results and interactive viewers

Return small scalar summaries and tables as normal conversation content. For
figures, use the client's supported native preview or image output; deliver the
underlying data and logbook as files or links. Do not require a Claude live
artifact. A standalone HTML morphology viewer remains a useful deliverable,
but describe it as an HTML file unless a supported preview actually renders
it; do not claim chat previews can load its CDN modules or run its JavaScript.
