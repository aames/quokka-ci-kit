# The CI section for an adopter's AGENTS.md and CLAUDE.md

An agent working in your repository needs the same few facts a new contributor needs: run `make check` before a push, never edit `ci/kit/`, never push with `--no-verify`, never loosen a job to make it pass. The kit keeps that section as one text, [`kit/templates/agents/section.md`](../kit/templates/agents/section.md) (vendored at `ci/kit/templates/agents/section.md`), and this page holds no copy of it, because two copies drift.

**How it gets into your repository:**

1. **With `kit-init`** (from `v1.1.0`), nothing to do: it writes the section into `AGENTS.md` and `CLAUDE.md` with your adapter, check command, jobs and main branch filled in, creating either file or appending after one blank line. `--no-agent-files` skips it.
2. **Adopted by hand, or on an earlier release:** vendor `v1.1.0` or later, then run:

   ```sh
   python3 ci/kit/bin/kit-init --agent-files-only
   ```

   It reads your committed `ci/manifest` and writes the two files without committing them. *Done when* both files hold the section between its `quokka-ci-kit section begin` and `end` markers; commit them in a pull request of their own.
3. **On a kit older than `v1.1.0`:** read the template in the kit repository, copy it into both files, and replace its six `@@NAME@@` tokens by hand: `@@ADAPTER@@` (your verdict script), `@@CHECK@@` (your check command), `@@JOBS@@` (the jobs it runs), `@@MAIN@@`, `@@WORKFLOW@@` and `@@SIGNERS@@` (from `ci/manifest`).

`kit-init` refuses to add the section to a file that already holds its begin marker, so running it twice changes nothing.
