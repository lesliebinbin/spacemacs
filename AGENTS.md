# Spacemacs Fork Maintenance & Agent Routing

## Upstream Synchronization Policy

This repository (`.emacs.d`) is a personal fork of upstream Spacemacs that is periodically synchronized with official upstream branches:

- **Do NOT modify upstream `README.md`**: Keep `README.md` completely untouched to prevent merge conflicts during upstream pulls/rebases.
- **Use `README-additional.org`**: Document all fork-specific customizations, notes, or documentation additions in `README-additional.org` (in Emacs Org format) or inside `.spacemacs.d/`.
- **Use `todos/` for cross-repository tasks**: The parent repository owns
  lifecycle tracking for work spanning `.emacs.d`, `.spacemacs.d`, and
  external component repositories. Canonical task files live in project
  directories under `todos/`; status directories contain only symlinks.
- **Do NOT modify upstream core files directly**: User configuration, layers, and packages belong in the `.spacemacs.d` submodule.

## Agent Assets

Reusable agent assets for this configuration are maintained in the `.spacemacs.d` submodule under `.spacemacs.d/agents/`:

- Refer to `.spacemacs.d/AGENTS.md` and `.spacemacs.d/agents/README.md` for asset layout, skills, runbooks, and profiles.
- When fixing or maintaining unmaintained Emacs packages, use the `emacs-package-fork-patch` skill and runbook.
