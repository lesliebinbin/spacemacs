# TODO — Fix origami `origami-fold-header-face` "Invalid face box" at the source

Status: **completed** (2026-09-07).
Created: 2026-09-04

## 1. Context

**Symptom:** in GUI Emacs, typing a completion prefix (e.g. `prin`) in
`gpu_memory_info.cu` with `toggle-debug-on-error` on shows:

```
Debugger entered--Lisp error: (error "Invalid face box" :line-width 1 :color unspecified)
  set-face-attribute(origami-fold-header-face ... :box (:line-width 1 :color unspecified) ...)
```

The company completion popup dies. Completion works fine in `emacs -nw`.

**Root cause:** upstream `gregsexton/origami.el` is effectively unmaintained —
last commit 2020-03-31 (`e558710`), installed copy is MELPA
`origami-20200331.1019`. It defines the face with colors **computed at load
time** and frozen into the face spec (`origami.el:59-62`):

```elisp
(defface origami-fold-header-face
  `((t (:box (:line-width 1 :color ,(face-attribute 'highlight :background))
             :background ,(face-attribute 'highlight :background))))
  "Face used to display fold headers.")
```

When origami loads before the theme applies, `highlight`'s background is
`unspecified`, which gets baked into the spec. Emacs 31 rejects
`:color unspecified` inside a `:box` when the spec is applied to any frame
created afterwards — the company tooltip is a posframe **child frame**, so
every popup creation re-applies specs on the new frame and aborts. The main
frame predates origami loading, and a terminal never creates child frames,
which is why only the GUI completion popup breaks.

Reproduced in batch on this machine:

```
emacs --batch -Q --eval '(defface zz-face (quote ((t (:box (:line-width 1 :color unspecified) :background unspecified)))) "x")'
```

Frozen package file:
`~/.emacs.d/elpa/31.1/develop/origami-20200331.1019/origami.el`

## 2. Temporary fix (applied — keep until §4 is done)

File: `.spacemacs.d/emacs-config/user-config.el`, end of
`dotspacemacs/user-config`. Re-registers the face with concrete colors once
the theme is loaded (verified working in batch):

```elisp
;; origami.el freezes `(face-attribute 'highlight :background)` into its
;; origami-fold-header-face defface at load time.  When origami loads
;; before the theme applies, that freezes `unspecified` into the :box spec,
;; and creating any later frame (e.g. the company completion popup via
;; posframe) fails with "Invalid face box".  Re-register the face with a
;; concrete spec (see origami.el `defface origami-fold-header-face`).
(let ((bg (or (face-background 'highlight) "grey70")))
  (custom-set-faces
   `(origami-fold-header-face ((t (:box (:line-width 1 :color ,bg)
                                        :background ,bg))))))
```

Note: the change in `.spacemacs.d` (a git submodule) is currently uncommitted.

## 3. Longer term: fork origami.el and fix it there

Upstream is dead (no commits since 2020-03-31, PRs unmerged, transfer to
emacsorphanage never happened — see melpa/melpa#6716), so no upstream fix
will ever arrive, and a fork will never drift from upstream either.

**Decision (open):** fix at the source, preferably by vendoring origami into
`.spacemacs.d/local/packages/origami/` (same pattern as the existing
`buffer-path-utils` local package); a public GitHub fork is the alternative
if we want it visible/shared. On completion, the temporary override in §2
should be removed (it becomes redundant) so there is a single source of
truth.

## 4. Steps to do it

1. `git clone https://github.com/gregsexton/origami.el` into
   `.spacemacs.d/local/packages/origami/` (or fork on GitHub first, then
   clone the fork); keep `origin` = the fork we own, add
   `git remote add upstream https://github.com/gregsexton/origami.el`.
2. Patch `origami.el` `defface origami-fold-header-face` (lines 59-62):
   drop the load-time backquote/`face-attribute` and use a static spec with
   light/dark variants, e.g.:

   ```elisp
   (defface origami-fold-header-face
     '((((class color) (min-colors 88) (background light))
        (:background "#cccccc" :box (:line-width 1 :color "#999999")))
       (((class color) (min-colors 88) (background dark))
        (:background "#333333" :box (:line-width 1 :color "#555555")))
       (t (:background "grey" :box (:line-width 1 :color "grey"))))
     "Face used to display fold headers.")
   ```

3. While we own the code, fix the other latent bugs in the same file:
   - `origami-fold-replacement-face` uses `:inherit 'font-lock-comment-face`
     — stray quote, should be `:inherit font-lock-comment-face` (~line 68-69).
   - deprecated `cl` require — Guix already carries a patch:
     https://issues.guix.gnu.org/61007
4. Make Spacemacs use the local package instead of the MELPA one (local
   `packages/` should shadow ELPA; verify no duplicate load).
5. Remove the temporary `custom-set-faces` override from
   `.spacemacs.d/emacs-config/user-config.el` (§2).
6. Restart Emacs and re-test: open a `.cu` file, `M-x toggle-debug-on-error`,
   type `prin`, confirm the completion tooltip appears with no error. Also
   re-test in `emacs -nw`.

## 5. Resolution Summary (2026-09-07)

- **Fork:** `git@github.com:lesliebinbin/origami.el.git`
- **Commit:** `8647d781834aa4e6cb918e46ac26c1684651803b`
  - Replaced load-time backquote `:box` calculation in `origami-fold-header-face` with static light/dark/default face spec.
  - Fixed stray quote on `:inherit font-lock-comment-face` in `origami-fold-replacement-face`.
  - Replaced deprecated `cl` with `cl-lib` (`cl-destructuring-bind`, `cl-remove-if`) and added `(require 's)`.
  - Updated deprecated `define-global-minor-mode` to `define-globalized-minor-mode`.
  - Verified compilation via `eldev` with zero errors.
- **Spacemacs Configuration:**
  - Added recipe pointing to commit `8647d781834aa4e6cb918e46ac26c1684651803b` under `dotspacemacs-additional-packages` in `.spacemacs.d/emacs-config/layers.el`.
  - Removed temporary `custom-set-faces` override from `.spacemacs.d/emacs-config/user-config.el`.
  - Installed and verified package loading via Quelpa into `elpa/31.1/develop/origami-20260907.94752/`.
