# Spacemacs Fork Maintenance & Agent Routing

## Upstream Synchronization Policy

This repository (`.emacs.d`) is a personal fork of upstream Spacemacs that is periodically synchronized with official upstream branches:

- **Do NOT modify upstream `README.md`**: Keep `README.md` completely untouched to prevent merge conflicts during upstream pulls/rebases.
- **Use `README-additional.org`**: Document all fork-specific customizations, notes, or documentation additions in `README-additional.org` (in Emacs Org format) or inside `.spacemacs.d/`.
- **Use `runbook/` for cross-repository features**: Keep each feature in one
  self-contained Org file under `runbook/<project>/`. Use Org TODO keywords
  inside that file for lifecycle state; do not maintain separate status
  directories or lifecycle symlinks.
- **Do NOT modify upstream core files directly**: User configuration, layers, and packages belong in the `.spacemacs.d` submodule.

## Component Topology and Delegation

Before executing a feature or maintenance task:

1. Scan the requested scope for distinct components, repositories, and
   repository-local `AGENTS.md` files.
2. Inspect the relevant agent assets in each component before proposing
   ownership.
3. If multiple components are involved, map their topology first: boundaries,
   dependencies, handoff order, reporting paths, and the root coordination
   point.
4. Present that topology to the user and ask whether they want a sub-agent
   approach before launching any sub-agents.
5. If the user approves delegation, assign each sub-agent one explicit
   component boundary and keep lifecycle and cross-component decisions with
   the root agent.
6. If the user declines delegation, or the scope is small enough to handle
   directly, keep the work with the root agent.

Feature runbooks describe the feature itself: context, requirements, decisions,
scope, lifecycle state, acceptance criteria, and outcomes. Keep reusable agent
selection, topology discovery, delegation, and reporting procedures in this
file rather than duplicating them in each feature runbook.

## Agent Assets

Reusable agent assets for this configuration are maintained in the `.spacemacs.d` submodule under `.spacemacs.d/agents/`:

- Refer to `.spacemacs.d/AGENTS.md` and `.spacemacs.d/agents/README.md` for asset layout, skills, runbooks, and profiles.
- When fixing or maintaining unmaintained Emacs packages, use the `emacs-package-fork-patch` skill and runbook.
