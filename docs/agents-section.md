# The CI section for an adopter's AGENTS.md and CLAUDE.md

An agent working in your repository needs the same few facts a new contributor does: run `make check` before pushing, never edit `ci/kit/`, never push with `--no-verify`, and never loosen a job to make it pass. The kit keeps that section as a single text, [`kit/templates/agents/section.md`](../kit/templates/agents/section.md) (vendored at `ci/kit/templates/agents/section.md`). This page deliberately holds no copy of it, because two copies would drift apart.

**How it gets into your repository:**

1. **With `kit-init`** (from `v1.1.0`), there is nothing to do. It writes the section into `AGENTS.md` and `CLAUDE.md` with your test script, check command, jobs and main branch filled in, creating either file if it is missing or appending after one blank line if it exists. `--no-agent-files` skips this.
2. **If you adopted by hand, or on an earlier release:** vendor `v1.1.0` or later, then run:

   ```sh
   python3 ci/kit/bin/kit-init --agent-files-only
   ```

   It reads your committed `ci/manifest` and writes the two files without committing them. *Done when* both files contain the section between its `quokka-ci-kit section begin` and `end` markers. Commit them in a pull request of their own.
3. **On a kit older than `v1.1.0`:** read the template in the kit repository, copy it into both files, and replace its six `@@NAME@@` tokens by hand: `@@ADAPTER@@` (your test script), `@@CHECK@@` (your check command), `@@JOBS@@` (the jobs it runs), `@@MAIN@@`, `@@WORKFLOW@@` and `@@SIGNERS@@` (all from `ci/manifest`).

`kit-init` refuses to add the section to a file that already contains its begin marker, so running it twice changes nothing.
