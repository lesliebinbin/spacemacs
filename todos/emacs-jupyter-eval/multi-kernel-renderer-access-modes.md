# Multi-kernel renderer and browser access modes

## Context

This work spans three related repositories and configuration boundaries:

| Area | Role |
| --- | --- |
| `.emacs.d` | Personal fork of upstream Spacemacs and parent repository for the `.spacemacs.d` submodule |
| `.spacemacs.d` | Personal Spacemacs configuration that installs, loads, and integrates `jupyter-eval` |
| `/home/lesliebinbin/codings/emacs-jupyter-eval` | Source repository for the kernel coordinator, Emacs adapter, broker, and browser renderer |

The upstream Spacemacs core and root `README.md` are not customization
surfaces. Integration changes belong in `.spacemacs.d`; product behavior
belongs in `emacs-jupyter-eval`.

## Components in scope

- The `jupyter-eval.el` coordinator and its lifecycle model.
- The Emacs `code-cells` adapter where integration behavior is affected.
- The Python input and output brokers that connect renderer sessions to
  Jupyter kernels.
- The React/Vite renderer, including routing, session discovery, live output,
  rich MIME rendering, and widgets.
- The `.spacemacs.d` package recipe and user configuration only where required
  to expose the resulting behavior.

The parent `.emacs.d` repository is context rather than a target for upstream
core changes.

## Current behavior

- A renderer instance serves one unparameterized root page at
  `http://127.0.0.1:5173/`.
- The kernel name and event URL are fixed through Vite environment variables
  when the renderer starts.
- Starting a kernel stops the previously managed kernel and replaces global
  coordinator process state.
- Each kernel execution uses a separate dynamically allocated event-broker
  port.
- Emacs opens the renderer with xwidget WebKit when available.
- Without xwidget WebKit, Emacs reports the URL but does not open an external
  browser.
- Any browser that reaches the current renderer origin can use the widget comm
  back-channel; there is no explicit read-only client mode.

## Desired behavior

- One long-lived renderer remains available on fixed port `5173`.
- The root path lists the currently running kernel sessions.
- Each running session has a dedicated subpath that shows the live evaluation
  feed currently shown at `/`.
- Session identity is unique even when multiple buffers use the same Jupyter
  kernelspec.
- Session labels make the kernel name and associated buffer understandable to
  the user.
- Multiple kernels can remain active concurrently without requiring a
  separate renderer port for each kernel.
- Internal event transports may retain distinct ports as long as users access
  every session through the single renderer origin.
- An xwidget session opened by Emacs is authorized for interactive widget
  input.
- An external browser can display the same live execution results in
  read-only mode.
- Read-only clients cannot send widget comm messages or otherwise mutate
  kernel state through the renderer.
- Interactive authorization is explicit and unguessable rather than inferred
  from a browser User-Agent.
- Widget controls in read-only views are visibly and functionally
  non-interactive.

## Out of scope

- Deriving additional semantics from Jupyter execution request IDs.
- Changing upstream Spacemacs core files.
- Adding notebook-style editing chrome to the renderer.
- Enabling kernel stdin prompts such as Python `input()`.

## Additional considerations

- A kernelspec name is not a sufficient route identity because several buffers
  can use the same kernelspec simultaneously.
- Read-only behavior must be enforced by the broker as well as represented in
  the UI; hiding controls alone is not an authorization boundary.
- The root session list must reflect starts, stops, failures, and stale
  sessions so it does not advertise dead kernels.
- A renderer restart or page refresh should recover the current session list
  and reconnect to active event streams.
- Session URLs should remain safe to open externally without exposing an
  interactive capability.
- Existing stream output, errors, rich MIME output, display updates, and
  ipywidget behavior must remain available in their appropriate access mode.
