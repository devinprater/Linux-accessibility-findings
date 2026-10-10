# Experiments

Code used to measure something, kept with the results it produced. The point of this directory
is that a finding in `research/` can point at a script here and the raw output it generated.

## How to add one

One directory per experiment, named for the question, for example
`inputcapture-touchscreen-on-gnome/`. Each directory carries:

- `README.md` — the question, the machine and versions it ran on, how to run it, and what the
  result was.
- The script or program.
- `results/` with the raw output, dated, exactly as produced, including the runs that failed.

## Rules

- Record the desktop, the compositor and its version, and the portal version. Touch and input
  behaviour differs between GNOME and KDE, and between portal releases.
- Keep the failures. A run that produced nothing is a result.
- Never edit a result file after the fact. Add a new run with a new date instead.

## Planned

- **InputCapture touchscreen on GNOME and KDE** — does the InputCapture portal deliver
  touchscreen events to a plain client, what do the events contain, and does the application
  underneath stop receiving touches while capture is active? This is the measurement that
  `research/touch-input-and-assistive-technology.md` says is missing.

Nothing here yet.
