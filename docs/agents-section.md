# The CI section for an adopter's AGENTS.md and CLAUDE.md

`kit-init` will write this section into your repository's `AGENTS.md` and `CLAUDE.md` for you once it exists ([kit-init.md](kit-init.md)). Until then, append it by hand: copy everything from the `## CI (quokka-ci-kit)` heading to the end of this file onto the end of each of the two files, creating either if absent, and commit. If your adapter, image or check command live elsewhere than the example's, change those paths in your copy.

## CI (quokka-ci-kit)

This repository's CI skips a job when its exact content already passed, in CI or, signed, on a workstation. The rules for working here:

- `ci/manifest` decides each job's key: the paths its verdict depends on, its pins and its counts. It is data; never put a secret in it.
- `ci/ci-job.sh <job>` is each job's one verdict script, run the same way by CI and by `make check`. Never put a secret in it; settings come from the environment.
- `make check` runs the local jobs at `HEAD` inside the job image. It tests the commit, not the working copy, so commit before running it.
- The pre-push hook (`.githooks/pre-push`) runs any job not yet passed and signs a record of each pass with the person's key. Never run `ci/kit/bin/local-ci sign` or `push` yourself, and never touch a signing key.
- Never edit `ci/kit/`. It is vendored from a signed release; update it only with `ci/kit/bin/kit-update <tag>`, in a pull request of its own.
- To add a job: its lines in `ci/manifest`, its arm in `ci/ci-job.sh` ending in one counts line, its job in `.github/workflows/ci.yml`; commit, then run `bash ci/kit/bin/check-manifest`, `python3 ci/kit/bin/check-workflow`, `python3 ci/kit/bin/check-escaping-reads`, `make check` and `bash ci/kit/bin/check-adapter <job>`.
- Never push with `--no-verify`; it skips the hook and the job runs again in CI. The landing pull request that adopted the kit was the one exception.
- Never change branch protection or required checks; ask the person.
- The reference is `ci/kit/README.md`; the design is `https://github.com/aames/quokka-ci-kit/blob/main/DESIGN.md`.
