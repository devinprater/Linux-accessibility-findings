# Linux accessibility findings

Research, measured findings and patches about accessibility on the Linux desktop and on Linux
phones, staged here before anything goes upstream.

## Why this repo exists

Upstream maintainers are, quite reasonably, wary of contributions that arrive as a large
AI-written patch with no evidence behind it. So the work happens here first:

1. Write the question down, with sources.
2. Measure it, if it can be measured, and keep the raw output.
3. Only then offer it upstream, as a small change with evidence attached.

Nothing in here is finished upstream work. Everything is a finding, a draft or an experiment,
and each document says what was verified and what was not.

## Layout

- `research/` — written findings and surveys, with sources. Each document opens with the
  question it answers and ends with what was checked and what was not.
- `experiments/` — code and scripts used to measure something, kept with the raw results they
  produced. See `experiments/README.md`.
- `patches/` — patches and drafts for other projects, kept here while they are being proven.
  See `patches/README.md`.

## Documents so far

- `research/touch-input-and-assistive-technology.md` — does any Linux desktop give assistive
  technology access to touch input today? First pass, October 2026.
- `research/how-other-readers-do-touch.md` — how NVDA, Narrator, JAWS, TalkBack, BackTalk and
  VoiceOver implement touch, the six patterns they share, and three ways a Linux reader could do
  the same.
- `research/linux-accessibility-landscape.md` — the moving parts of Linux accessibility in 2026:
  AT-SPI, Newton, AccessKit, the two a11y D-Bus interfaces and the portals, with a list of what
  could be worked on.

## Conventions

- Every claim gets a link, and every link gets a date or a version where one exists.
- Say plainly which parts were verified by running or reading something, and which are second
  hand.
- Measurements go in as raw output, not as a summary of output.
- Research text is offered under CC BY 4.0. Patches follow the licence of the project they are
  written for.
