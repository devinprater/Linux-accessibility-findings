# Patches

Patches and drafts for other projects, kept here until they have been proven, and offered
upstream afterwards as a small change with evidence attached.

## Why they live here first

Maintainers are rightly wary of a large patch that arrives with no evidence and no explanation
of how it was produced. A patch that has already been built, run and reported on locally is a
much easier thing to accept, and if it is rejected there is still a record of the work and its
results.

## How to add one

One directory per patch, named for the project and the change, for example
`gnome-control-center-missing-labels/`. Each directory carries:

- `README.md` — what the problem is, what the patch does, which upstream version it applies to,
  how it was built, how it was tested, and what the raw output was.
- The patch itself, as a `.patch` or `.diff` file, or the modified files if a patch does not
  apply cleanly yet.
- `evidence/` if there is raw output worth keeping, such as logs, test results or screen
  listings.

## Rules

- State the upstream version and commit the patch applies to.
- Say whether it was built and run, or only read. Do not describe an untested patch as working.
- Keep the change as small as the problem allows. A big refactor needs a discussion before it
  needs a patch.
- If a maintainer rejects it, leave the directory here and add their reason, so the next person
  does not repeat the work.

No patches yet.
