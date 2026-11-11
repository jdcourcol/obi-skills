# obi-skills

Agent Skills for the Open Brain Institute (OBI) platform — reusable instructions that teach AI agents how to authenticate, link results, keep scientific logbooks, and drive OBI APIs (data integration, circuit simulation, single-cell / ion-channel modeling, skeletonization, visualization, and related workflows).

Skills follow the [Agent Skills](https://agentskills.io/) format and live under [`skills/`](./skills/). Install or symlink them into your agent’s skills directory (e.g. `.cursor/skills/`, `.claude/skills/`, `.kiro/skills/`), or point your tool at this folder.

| Skill | Purpose |
| --- | --- |
| [`build-synaptome`](./skills/build-synaptome/SKILL.md) | Place synaptic models on an ME-model → single-neuron synaptome circuit |
| [`chat-obi-sandbox-bridge`](./skills/chat-obi-sandbox-bridge/SKILL.md) | Claude Chat ↔ OBI sandbox file bridge |
| [`circuit-customization`](./skills/circuit-customization/SKILL.md) | Derive a customized circuit from a parent by uploading override files |
| [`circuit-extraction`](./skills/circuit-extraction/SKILL.md) | Extract a simulatable sub-circuit from a larger SONATA circuit |
| [`connectivity-matrix-extraction`](./skills/connectivity-matrix-extraction/SKILL.md) | Extract a ConnectomeUtilities connectivity matrix from a circuit (in-sandbox) |
| [`data-integration-api`](./skills/data-integration-api/SKILL.md) | Ingest morphologies, circuits, and metadata via obi-one |
| [`disease-modeling`](./skills/disease-modeling/SKILL.md) | End-to-end disease / condition modeling on a microcircuit |
| [`emodel-building`](./skills/emodel-building/SKILL.md) | Build e-models: e-feature extraction + BluePyEModel optimization |
| [`em-synapse-mapping`](./skills/em-synapse-mapping/SKILL.md) | Map EM-reconstructed afferent synapses onto morphologies → circuit |
| [`ephys-efeature-extraction`](./skills/ephys-efeature-extraction/SKILL.md) | Compute eFEL e-feature metrics from an intracellular recording |
| [`ion-channel-api`](./skills/ion-channel-api/SKILL.md) | Fit and simulate ion channel models |
| [`morphology-with-spines`](./skills/morphology-with-spines/SKILL.md) | Load & analyze spiny morphologies from a CellMorphology or a segmented-spines Circuit (entitysdk / bluepysnap + morph-spines + ConnectomeUtilities) |
| [`morphoviewer-standalone`](./skills/morphoviewer-standalone/SKILL.md) | Standalone HTML morphology / circuit viewers |
| [`ngv-metabolism`](./skills/ngv-metabolism/SKILL.md) | Run the NGV unit metabolism model and plot neuronal ATP |
| [`obi-circuit-simulation`](./skills/obi-circuit-simulation/SKILL.md) | SONATA circuit download, modify, register, and simulate |
| [`obi-links`](./skills/obi-links/SKILL.md) | Hyperlink entities, sandbox paths, and launched jobs |
| [`obi-logbook`](./skills/obi-logbook/SKILL.md) | Scientific logbook convention for OBI workflows |
| [`obi-research-log`](./skills/obi-research-log/SKILL.md) | Register a logbook as an EntityCore ResearchLog (on request) and keep it in sync with web edits |
| [`obi-sandbox-auth`](./skills/obi-sandbox-auth/SKILL.md) | Auth-manager token exchange and offline consent |
| [`parameter-scan`](./skills/parameter-scan/SKILL.md) | Shared parameter-scan model behind every obi-one scan-config workflow |
| [`single-cell-simulation-api`](./skills/single-cell-simulation-api/SKILL.md) | ME-model building and single-neuron simulation |
| [`skeletonization-api`](./skills/skeletonization-api/SKILL.md) | Skeletonization campaigns via obi-one / EntityCore |
| [`synapse-parameterization`](./skills/synapse-parameterization/SKILL.md) | Assign synaptic physiology to a circuit's connections |
| [`synaptome-simulation-api`](./skills/synaptome-simulation-api/SKILL.md) | Simulate a single-neuron synaptome (ME-model with synapses) circuit |
| [`virtual-lab-manager-api`](./skills/virtual-lab-manager-api/SKILL.md) | Virtual labs, projects, and credits |

Example install with the skills CLI (if you use it):

```bash
npx skills add openbraininstitute/obi-skills
```

# Claude plugin

The `open-brain-institute` plugin bundles all skills above with the production OBI MCP server, registered as `obi` (`https://cell-a.openbraininstitute.org/api/agent-ts/mcp`). (You may also see this server called `neuroagent`; it is the same server.) The plugin is distributed through this repository, which doubles as a plugin marketplace named `obi`.

## Install in Claude Code

```
/plugin marketplace add openbraininstitute/obi-skills
/plugin install open-brain-institute@obi
```

Or in one step: `/plugin install open-brain-institute --marketplace openbraininstitute/obi-skills`. Skills then appear as `/open-brain-institute:<skill>`, e.g. `/open-brain-institute:obi-logbook`.

## Install in Claude Desktop / Cowork

1. Open **Customize** in the sidebar and select **Plugins**.
2. Select **Add marketplace** and enter `openbraininstitute/obi-skills`.
3. Find **Open Brain Institute** in the list and click **Install**.
4. Open the installed plugin, go to its **Connectors** tab, and connect `obi`. Installing does not sign you in. Sign-in opens in your browser.

## Updating

Third-party marketplaces do not auto-update by default.

- **Claude Code:** run `/plugin`, open the **Marketplaces** tab, select `obi`, and choose **Update marketplace** (or **Enable auto-update**).
- **Claude Desktop / Cowork:** select **Check for updates** on the marketplace, or turn on **Sync automatically**.

## Maintainers

Whenever a skill or the MCP config changes, bump `version` in [`.claude-plugin/plugin.json`](./.claude-plugin/plugin.json) in the same PR; installed copies are cached by version. Keep the plugin name `open-brain-institute` so an update replaces the old version instead of adding a second plugin.

To test a branch before merging, run `claude --plugin-dir .` from a checkout, or `/plugin marketplace add openbraininstitute/obi-skills#<branch>`.

# Licence

Licensed under the Apache License, Version 2.0. See [LICENSE](./LICENSE).

# Acknowledgements

Copyright © 2025-2026 Open Brain Institute
