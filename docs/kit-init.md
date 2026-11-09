# `kit-init`: the generator of an adopter's files

```text
        .-"""-.        ______
       / .===. \      |______|
       \/ ^ ^ \/     _|______|
       ( \___/ )___/  |______|    six files, one init
    ____ooo___ooo_____|______|____
```

`kit-init` writes the files a repository needs to adopt quokka-ci-kit, from a handful of flags that say where your code lives. It shipped in `v1.1.0`. This document is for the kit's maintainers and contributors, and for anyone who wants to know exactly what the generator writes and why. It covers what the command does, what each generated file contains and where its values come from, how the templates are kept true by the kit's own tests, and how the generator is released. The quick route for an adopter is [QUICKSTART.md, Faster: kit-init](../QUICKSTART.md#faster-kit-init).

## 1. Purpose and scope

**What it changes.** Without the generator, an adopter writes six files by hand (the manifest, the test script, the job image's Dockerfile with each toolchain pinned by sha256 for both architectures, a workflow of about 260 lines, the hook shim and a Makefile target), plus the signers file, and lands them through the fourteen steps of the adoption table (`kit/README.md` §7). With `kit-init`, adoption is four acts:

1. vendor the kit (`kit-update <tag>`, unchanged, `kit/README.md` §5.3);
2. `python3 ci/kit/bin/kit-init --project NAME --go api --node web ...`;
3. `make check`;
4. push the branch and open the first pull request.

The generator writes complete files for the stacks it knows (Go, Node with vitest, Python with pytest) and an honest stub for anything else: a stub section in the test script that refuses with a sentence saying what to fill in, so a stub can never pass.

**Why a generator and not just a template repository.** A template repository (the quickstart's route) gives one shape. Every adopter's shape differs in its directories, job names, toolchain versions and services, and each of those appears in several files at once (a pin file is a `pin-source`, an `image-path`, a path in every local job's key, a `COPY` in the Dockerfile and a cross-check in the Dockerfile's `RUN`). Writing those consistently by hand is where adoption goes wrong; deriving them from one set of flags is what a generator is for.

**What it is not.** It is not a second source of rules. Everything it writes is held by the same checks an adopter's hand-written files are held by (`check-manifest`, `check-workflow`, `check-escaping-reads`, `check-adapter`), and the kit's tests keep every shape it can produce passing through them (§7.3). It runs once, at adoption. It never rewrites a file it wrote earlier.

**Deliberately out of scope**, each for a reason:

| Not generated | Why |
|---|---|
| `ci-only` and `content` jobs | each is project-specific by definition (a host assertion, an apply step); there is no template that would be honest |
| a plan extension (`kit/README.md` §4.4) | it decides jobs outside the scheme, which the generator cannot know about |
| `socket` and `history` jobs | each widens a job's reach (`kit/README.md` §3.3); that is a decision a person makes and states, never a default |
| `env` lines and `not-run-allowed` lines | an `env` line is a setting the generator cannot know; an allow-list entry exists only for a part of a test that has been shown to be impossible on a workstation |
| package managers other than npm, test runners other than vitest and pytest, languages other than Go, Node and Python, services other than Postgres | the manifest's closed kinds are `npm-ci` and `postgres` (`kit/README.md` §3.3); other stacks get a stub section and a stub image block |
| regenerating or upgrading generated files | the files are the adopter's own once written (`kit/README.md` §2.1); a later kit release says in its CHANGELOG what an adopter must change by hand |
| the trust paragraph (DESIGN §3.4) | it belongs in the adopter's own documentation, in their words around the copied paragraph; the generator prints it as a step |

## 2. The command

### 2.1 Synopsis

```
python3 ci/kit/bin/kit-init --project NAME
        [--go DIR[:JOB]]... [--node DIR[:JOB]]... [--python DIR[:JOB]]...
        [--postgres JOB[:MAJOR]]...
        [--signer auto|FILE] [--cron 'M H D M W']
        [--checksums FILE] [--no-commit] [--no-agent-files]

python3 ci/kit/bin/kit-init --agent-files-only
```

Run it at the top of a working copy, on a branch that is not the manifest's `main-branch`, after `kit-update` has written `ci/kit/`, `ci/KIT-VERSION` and `ci/KIT-SIGNERS`. The file is executable with a `#!/usr/bin/env python3` line, so `ci/kit/bin/kit-init` also works; the `python3` form is the documented one, as it is for the two Python checks.

### 2.2 The flags

| Flag | Takes | What it produces | Default |
|---|---|---|---|
| `--project NAME` | the project word | the manifest's `project` line and every name derived from it (`kit/README.md` §3.5): the counts prefix, the cache paths in the workflow, the service URL variable | required |
| `--go DIR[:JOB]` | a directory holding `go.mod`; repeatable | one `local` job: manifest lines (§3.1), a Go section in the test script (§3.2), the Go block of the image (§3.3), a W-7 job with the `gomod` and `gobuild` caches (§3.4) | none |
| `--node DIR[:JOB]` | a directory holding `package.json` and `.nvmrc` (or `.node-version`); repeatable | one `local` job: manifest lines with `install <job> npm-ci <dir>` when `package-lock.json` exists, a vitest section or a stub section, the Node block of the image, a W-7 job with the `npm` cache | none |
| `--python DIR[:JOB]` | a directory holding `.python-version`; repeatable | one `local` job: manifest lines, a pytest section or a stub section, the Python block of the image, a W-7 job with no cache step | none |
| `--postgres JOB[:MAJOR]` | a job one of the stack flags declared; repeatable, once per job | a `service <job> postgres` line with the image pinned by digest, the URL variable `<PROJECT>_TEST_PG_URL`, CI's two flags, and the section's refusal when the variable is unset | major `18`, the example's |
| `--signer auto\|FILE` | `auto`, or a file holding one SSH public key line | the first signer's allowed-signers line and its fingerprint, printed for the signers pull request and kept at `<git common dir>/<project>-ci/init/allowed-signers.next`; never written into the landing commit (§3.8) | none: the closing steps say how to make the line |
| `--cron 'M H D M W'` | five cron words | the manifest's `weekly-cron` and the workflow's one schedule, equal by construction (W-1) | `20 6 * * 1`, the example's |
| `--checksums FILE` | a file of `<64 hex>  <name>` lines | the toolchain checksums and the service digest, read from the file instead of fetched (§4.3) | fetched |
| `--no-commit` | nothing | the files are written and not committed; the checks then run on the working files (§5) | commit |
| `--no-agent-files` | nothing | neither `AGENTS.md` nor `CLAUDE.md` is written or appended to | both written |
| `--agent-files-only` | nothing; refuses every other flag | only the two agent sections, their values read from the committed manifest through `bin/keys dump HEAD`, for a project that adopted by hand or before this command existed; it writes them and does not commit them | off |

**Job names.** `JOB` defaults to the directory with `/` written as `-` (`services/api` gives `services-api`), and to the stack's name for `.` (`go`, `node`, `python`). It must be a manifest NAME (`kit/README.md` §3.1). The generator also declares two jobs of its own, the always-on `checks` and the kit job `kit`, and writes a workflow job `prune`, so `checks`, `kit` and `prune` are refused as stack job names, as are `image` and `base`, which the manifest itself refuses. Why fixed names for the two kit jobs: they are the example's, and every reader of the kit's documents meets them there; a project that wants different ones renames them in its own pull request later, and the checks hold the rename.

**Repeated stacks.** Two `--go` directories make two jobs in one image, and the image carries one Go. So every Go directory's `toolchain` line must name the same version; the first directory's `go.mod` is the one `pin-source go` line and the one image input, and each later job's section still reads its own `go.mod` at run time, so a later drift is refused by that section (§3.2). The same holds for Node and Python. Why: `pin-source` takes one line per tool (`kit/README.md` §3.4), and an image with two Go versions would be two images.

### 2.3 The refusals

Every refusal is one sentence ending `so nothing was written.`, printed before any file is touched, with exit 2. The generator renders every file in memory first, then checks every refusal, then writes; the paths it would write are checked for symbolic links after rendering and before writing, so a link refusal comes after the others have passed. A write that fails part way removes the files this run created, truncates each file it appended to back to the length in bytes that file had on disk before the run, and touches nothing else.

| Refusal | Sentence (the shape; the exact words may be polished) |
|---|---|
| not at the top of a git working copy | `kit-init runs at the top of a git working copy, so nothing was written.` |
| the kit is not vendored | `There is no ci/kit/lib/keys.sh or ci/KIT-VERSION; vendor the kit with kit-update first, so nothing was written.` |
| a file it would write exists | `ci/manifest already exists, so nothing was written.` (one sentence per such file: `ci/manifest`, `ci/ci-job.sh`, `ci/image/Dockerfile`, `.github/workflows/ci.yml`, `.githooks/pre-push`, `ci/allowed-signers`) |
| a file it appends to already holds what it would add | `The Makefile already has a check target, so nothing was written.`; `AGENTS.md already holds the quokka-ci-kit section, so nothing was written.` (likewise `CLAUDE.md`) |
| an unknown flag or stack | `--rust is not a stack kit-init knows (it knows --go, --node and --python), so nothing was written.` The closing sentence of the usage names the stub route: a stack it does not know is declared by hand after the run. |
| the project word | `<word> is not a project name the manifest accepts (a lower-case letter, then 1 to 15 lower-case letters or digits), so nothing was written.` The test is the manifest's own rule, and the rendered manifest goes through the parser before anything is written (§5.1), so the generator cannot be looser than the check |
| a job name out of shape, reserved or repeated | `<name> cannot name a job (<why>); give the directory a name with DIR:JOB, so nothing was written.` |
| a directory without its stack file | `api has no go.mod, so it is not a Go directory and nothing was written.` |
| a pin that is absent or not exact | `api/go.mod has no toolchain line, and the job image is pinned from it; add one such as "toolchain go1.27.3", so nothing was written.` For Node: `web/.nvmrc holds "lts/*", which is not an exact version such as 24.22.1, so nothing was written.` For Python: `api has no .python-version; pyproject.toml's requires-python is a range, not a pin, so nothing was written.` |
| repeated stacks that disagree | `api/go.mod pins go1.27.3 and worker/go.mod go1.27.2, and the job image carries one Go, so nothing was written.` |
| `--postgres` naming no stack job | `--postgres names db, which no stack flag declared, so nothing was written.` |
| no git identity | `git has no user.name or user.email here, and the adoption commit and the signer's principal need them, so nothing was written.` (with `--no-commit` and no `--signer` nothing needs an identity, so this refusal is not raised) |
| `--signer auto` with no usable key | `--signer auto needs gpg.format ssh and a user.signingkey whose public key can be read, so nothing was written.` |
| `--signer FILE` not one key | `<file> holds no single SSH public key line of a type the signers check knows, so nothing was written.` (the types are DESIGN §6.6's list) |
| a checksum or digest that cannot be had | `The sha256 of go1.27.3.linux-arm64.tar.gz could not be fetched from https://go.dev/dl/?mode=json&include=all; fetch it yourself and pass it with --checksums, so nothing was written.` (one sentence per missing value, each naming its URL, and the file shape of §4.3) |
| the current branch is the main branch | `kit-init commits the adoption on a branch for its own pull request, and this is main, so nothing was written.`, naming the main branch read from `origin/HEAD`, or `main` when that is unset (not raised with `--no-commit`) |
| HEAD is detached | `HEAD is detached, and kit-init commits on a branch for its own pull request, so nothing was written.` (not raised with `--no-commit`) |
| a path it would write or append to is, or lies under, a symbolic link | `CLAUDE.md is a symbolic link, and kit-init writes no file through one, so nothing was written.` (one sentence per link, naming the link's own path; a link inside the working copy is refused too) |
| a directory or file name outside the bounded shape | `api$(x) is not a directory name kit-init writes into a generated file (a directory is ., or letters, digits and . _ / - starting with a letter or digit, with no .. segment), so nothing was written.`; for a file name derived from a directory (a requirements file), the same with "file name" and "a name is" |
| the rendered manifest fails the manifest check | the check's own findings, then `The manifest kit-init would write does not pass check-manifest, so nothing was written.` A failure here is a defect in the generator, and the kit's tests exist to make it impossible (§7.3) |
| a Python below 3.9 | `kit-init needs python3 3.9 or later, and this is <version>, so nothing was written.` |
| a file it reads that is not UTF-8 (a stack file, a pin file, a requirements file, a file it appends to, or `ci/KIT-VERSION`, whose tag is read before anything is written) | `<path> is not UTF-8 text kit-init can read, so nothing was written.` |

### 2.4 What it never does

- **No push.** The landing push is the person's, after `make check` (§6).
- **No protection or settings change.** Branch protection, required checks and Actions settings stay a person's acts on the host.
- **No key generation, and no key in the landing commit.** It reads the existing signing key's public half for `--signer auto` and nothing else.
- **No network beyond the lookups of §4**, each to a named host, all skipped with `--checksums`.
- **Nothing outside the repository, and nothing through a symbolic link.** No target, and no directory above one, may be a link. Its scratch files live in `<git common dir>/<project>-ci/init/`, under the kit's state directory, and are removed at exit except for `allowed-signers.next`.
- **No edit of an existing line** in any file. It creates files, and appends to `Makefile`, `.gitignore`, `AGENTS.md` and `CLAUDE.md`; it never changes `ci/kit/`, `ci/KIT-VERSION` or `ci/KIT-SIGNERS`.
- **No run of `make check`.** Building the job image takes minutes and is the next thing the person does, in the order of §6.

### 2.5 Exit codes

`0`: written (and committed), every check passed. `1`: written, and a check run after writing found something, or the commit failed and one line says so; the findings are printed and the files stay for the person to fix. `2`: refused; nothing was written. `3`: written (and committed unless `--no-commit`), every check that ran passed, but a check did not run; the output says which and how to run it. When more than one applies, the first in the order 2, 1, 3 is the code: a finding is never hidden behind exit 3.

## 3. What it writes, file by file

The example adopter is the reference output: `kit-init --project rottnest --go api --node web --postgres api` run on the example's tree with its six files removed must reproduce them, equal line for line once comment lines are set aside, with the Postgres digest as the one value that may differ. Every rule below is derived from that output.

### 3.1 `ci/manifest`

Written from `kit/templates/manifest/` in the order of the example's manifest, each block from a census the generator can do without running anything.

| Directive | Value | Where it comes from |
|---|---|---|
| `manifest`, `key-epoch`, `local-epoch` | `v1`, `1`, `1` | fixed |
| `project` | `--project` | flag |
| `main-branch` | the remote's default branch as `refs/remotes/origin/HEAD` names it, otherwise `main` | git; the plan and the prune read `main` from `origin` |
| `workflow`, `adapter`, `image`, `signers` | `.github/workflows/ci.yml`, `ci/ci-job.sh`, `ci/image/Dockerfile`, `ci/allowed-signers` | fixed: the example's paths, all under `ci` except the workflow, so `ci` in each key covers them (rules 3, 5, 9) |
| `always-on`, `kit-job` | `checks`, `kit` | fixed (§2.2) |
| `check-command` | `make check` | fixed; the Makefile target of §3.6 |
| `weekly-cron` | `--cron` | flag; a minute of 0 is refused by rule 12 of the manifest check |
| `prune-days`, `lock-wait-minutes`, `refuse-drafts`, `user` | `30`, `30`, `yes`, `1001` | the kit's defaults, written out so the person can see them |
| `prune-keep-green` | `checks`, each stack job in flag order, `kit` | every declared job |
| `image-path` | `ci/image`, then each tool's pin file in flag order | the Dockerfile's build context is exactly these paths; a pin file is in it because the Dockerfile cross-checks it (§3.3) |
| `job` | `checks always-on`, each stack job `local`, `kit local` | flags |
| `paths <job>` | its directory, then every `image-path` entry outside `ci` that its directory does not already cover, then `ci` | rules 3 and 9: `ci` covers the kit, the manifest, the test script, the image and the signers file; the other jobs' pin files are image inputs. This reproduces the example's `paths api api web/.nvmrc ci` |
| `paths checks`, `paths kit` | `.`, `ci` | rules 2 and 3 |
| `pin <job> <tool>` | `go`, `node` or `python` | one per stack job that is not a stub; a stub gets no `pin`, `pin-source`, `cache` or `install` line, and its manifest comment says to add them with the section |
| `count <job> ...` | Go `go_packages go_tests`; Node `vitest_files vitest_tests`; Python `pytest_tests`; a stub `tests`; the kit job `kit_cases` | the section's counts line (§3.2); the kit job's line is fixed |
| `install <job> npm-ci <dir>` | for a Node job whose directory holds `package-lock.json` | the one install kind; without a lockfile the section is a stub (§3.2) |
| `service` | `service <job> postgres image postgres:<major>@sha256:<digest> url-env <PROJECT>_TEST_PG_URL flag fsync=off flag synchronous_commit=off` | `--postgres`; the digest from §4.2; the line is the example's |
| `cache` | Go `gomod gobuild`; Node `npm`; Python none; a stub none | the three kinds; Python has no cache kind in `manifest v1`, so its runs install cold (§3.2) |
| `pin-source` | `go toolchain-line <dir>/go.mod`; `node first-line <dir>/.nvmrc` (or `.node-version`); `python first-line <dir>/.python-version` | the three kinds; one line per tool, from the first directory of that stack (§2.2) |
| `probe` | `go first-line go env GOVERSION`; `node first-line node --version`; `python version-flag python3` | the example's two; for Python, `version-flag` reads `Python 3.12.7` as `3.12.7`, where `first-line` would read the whole line |
| `escaping-tree <job> <dir>` | each stack job | the escaping-read check's trees |
| `escaping-scan` | Go `go <dir>`; Node `ts <dir>/src` when that directory exists, otherwise `ts <dir>`; Python none | the two scanners; for Python the generator writes a comment line saying crossings are declared by hand, and prints the same |

The comment lines of the template say what each block is for, in the example's words, with the project's job names substituted.

### 3.2 `ci/ci-job.sh`

Assembled from `kit/templates/adapter/`: a head, one section per stack job, and a tail. The head and tail are the example's: `set -uo pipefail`, `readonly PROJECT=<project>`, the `err`, `refuse`, `failed`, `need` and `step` helpers, the scratch directory, and the `case` that dispatches each job to its section and refuses any other name with exit 2 (the contract of `kit/README.md` §4). File mode 0755, as the example's.

| Section | What it runs | Counts line | Refusals (exit 2) |
|---|---|---|---|
| Go | `gofmt -l`, `go vet ./...`, `go test -count=1 -v ./...` to a file read after its status | `go_packages`, `go_tests`, counted from the saved output | the toolchain absent or off the directory's own `toolchain` line; the directory missing; with `--postgres`, the URL variable unset |
| Node with vitest (`vitest` in `package.json`'s dependencies or devDependencies) | `tsc --noEmit` when `tsconfig.json` exists, then `vitest run` with the JSON reporter to a file | `vitest_files`, `vitest_tests`, read with `jq`, which the base image carries | node absent or off the pin file; `node_modules/.bin/vitest` absent (the install did not run); with `--postgres`, the URL variable unset |
| Python with pytest (`pytest` named in a requirements file carrying `--hash=` lines) | a venv in the scratch directory, `pip install --require-hashes -r <file>`, then `python -m pytest --junitxml=<file>` | `pytest_tests`: the tests the JUnit file reports minus failures, errors and skips, read by the venv's Python | python absent or off `.python-version`; with `--postgres`, the URL variable unset |
| Stub (any stack whose test runner the generator does not recognise: Node without vitest or without a lockfile, Python without a hashed requirements file) | nothing | none | always: `The <job> arm is a stub kit-init wrote; put its checks in ci/ci-job.sh, so <job> did not run.` |

**Why a stub refuses rather than passes.** A section that exited 0 with no checks would let a record vouch for a tree nothing tested. Exit 2 is the contract's "refused before any check", so `make check` stops red on the stub and the person sees the one sentence that says what to do. The stub's `count <job> tests` line is a placeholder the person renames when they write the section.

**Why the Python install sits in the test script.** `manifest v1` has one install kind, `npm-ci`, and an ordinary job's container runs on the bridge network, so the section can install. Requiring hashes keeps the result's inputs pinned; a requirements file without hashes gets a stub, and the closing steps say how to make one. A `pip` install kind and a `pip` cache kind would be a later MINOR release of the manifest (`kit/README.md` §3.8), not part of this one.

### 3.3 `ci/image/Dockerfile`

Assembled from `kit/templates/image/`. For the Go and Node shape it is the example's file; its parts:

1. `ARG CIKIT_BASE` then `FROM ${CIKIT_BASE}`, the one base `FROM` that rule 9 of the manifest check accepts.
2. Per tool, `ARG <TOOL>_VERSION=<pin>`, `ARG <TOOL>_SHA256_AMD64=<hex>`, `ARG <TOOL>_SHA256_ARM64=<hex>`; for Python also `ARG PBS_TAG=<release>` (§4.1).
3. `USER root`, `COPY <each pin-source file> /pins/`, `SHELL ["/bin/bash", "-o", "pipefail", "-c"]`.
4. One `RUN` that, for each tool: cross-checks the pin file against the `ARG` and stops the build with the sentence the example uses when they disagree; picks the checksum for `dpkg --print-architecture` and stops on any other architecture; downloads the release tarball from its official location and verifies it with `sha256sum -c`; extracts it under `/usr/local`; then creates the cache directories of the declared cache kinds as the job user: `gomod` gives `/home/ci/go/pkg/mod` and its parents, `gobuild` `/home/ci/.cache/go-build`, `npm` `/home/ci/.npm`.
5. One `ENV`: `PATH` with each tool's `bin` first, then `/usr/local/bin:/usr/bin:/bin`; for Go `GOTOOLCHAIN=local`, `CGO_ENABLED=0`, `GOPATH=/home/ci/go`, `GOCACHE=/home/ci/.cache/go-build`; for Node `npm_config_cache=/home/ci/.npm`, `npm_config_update_notifier=false`; for Python `PIP_DISABLE_PIP_VERSION_CHECK=1`, `PYTHONDONTWRITEBYTECODE=1`.
6. `USER ci`, `WORKDIR /home/ci`.

The sources, per tool:

| Tool | Tarball | Extracted to |
|---|---|---|
| Go | `https://go.dev/dl/go<V>.linux-<amd64\|arm64>.tar.gz` | `/usr/local/go` |
| Node | `https://nodejs.org/dist/v<V>/node-v<V>-linux-<x64\|arm64>.tar.xz` | `/usr/local/node`, with the archive's top directory and its file ownership dropped |
| Python | `https://github.com/astral-sh/python-build-standalone/releases/download/<PBS_TAG>/cpython-<V>+<PBS_TAG>-<x86_64\|aarch64>-unknown-linux-gnu-install_only_stripped.tar.gz` | `/usr/local/python` |

**Why both architectures every time.** The image is built on each machine for its own architecture and never pulled: a Mac builds `arm64`, the runner `amd64`, so a file with one checksum would work on one side only.

**Why the cross-check.** The pin lives in the file the tool reads (`go.mod`, `.nvmrc`, `.python-version`) and the Dockerfile names the same version with its checksum; the build stops when they disagree, so a pin bump is one reviewed edit of both files together.

**A shape with no stack** (only `--project`) gets steps 1 and 6 alone: a valid image file, the kit job as its only real job, and a closing line saying so.

**A stub stack** gets no image block: the person adds the toolchain along with the section.

### 3.4 `.github/workflows/ci.yml`

Assembled from `kit/templates/workflow/`, which reproduces the example's workflow:

- the triggers, the dispatch inputs `force`, `task` and `delete` and the workflow-level `permissions: contents: read`, with the schedule set to `--cron`;
- the always-on job `checks`: the draft refusal, the checkout, the plan step with its six variables, the manifest check, the two Python checks and the record step, and its `outputs` holding `sha`, `ref_group`, `main`, then one `<job>` and one `<job>_key` per declared job in manifest order;
- one job per stack job in W-7's exact shape: the checkout at the always-on job's `sha`, the cache-runtime action, the cache steps of its `cache` line (`gomod` keyed on `hashFiles('<dir>/go.sum')`, `gobuild` on the date stamp, `npm` on `hashFiles('<dir>/package-lock.json')`, as `kit/README.md` §5.5 gives them; `go.mod` when `go.sum` is absent), the image step, the test step and the record step;
- the kit job, its image step naming itself so that only the base is built;
- the `prune` job, on `main` only, a dry run on the schedule.

`timeout-minutes` are the example's (10 for `checks` and `prune`, 20 for a stack job, 30 for the kit job), with a comment saying to set each by measuring the job (DESIGN §11).

Every `uses:` names the action by the commit sha the example pins (`kit/README.md` §5.5; W-11). The template is the one place in the kit where those three shas live outside the documents, and moving them is a release of the kit (§9.1).

**Why from a template the workflow check already passes.** W-1 to W-12 (`kit/README.md` §2.3) are exact: the plan's six variables, the concurrency groups, the record step's shape. A generated workflow satisfies them by construction only if the template itself is tested against `check-workflow` for every shape the generator can produce, which §7.3 requires.

### 3.5 `.githooks/pre-push`

Copied byte for byte from `ci/kit/hooks/pre-push.shim`, mode 0755. It is not a template, so the shim has one source; rule 8 of the manifest check requires the two to be equal.

### 3.6 `Makefile`

From `kit/templates/Makefile.check`, the example's target:

```make
JOBS ?= <each stack job> kit

.PHONY: check
check:
	bash ci/kit/bin/local-ci run --commit HEAD $(JOBS)
```

When a `Makefile` exists, the block is appended after one blank line; a `Makefile` that already defines `check:` is refused (§2.3), because merging two `check` targets is a person's job, not the generator's.

### 3.7 `.gitignore`

Nothing for the kit itself: the kit writes nothing into a working copy, its state lives under the git common directory, so there is no kit line to add. Per stack, one line is appended when absent: `node_modules/` for a Node job, `__pycache__/` for a Python job. Why these two: each holds files a test run or an install writes into the working copy, and a stray commit of either would enter the job's archive and its key.

### 3.8 `ci/allowed-signers`

Written as one comment line, from `kit/templates/allowed-signers`:

```
# The keys whose local records CI trusts, one line each: <principal> namespaces="git" <key type> <key>. Each is added in a pull request of its own (DESIGN.md 6.6, 9).
```

**Why the landing commits no signer, even with `--signer`.** DESIGN §9 step 1 commits the signers file empty or holding only a comment, and step 4 adds the first signer in its own pull request with the fingerprint confirmed through another channel (DESIGN §6.6). The signer line is the one trust decision of adoption; putting it inside a landing of several hundred generated lines hides it from review. So `--signer` computes the line and prints it with `ssh-keygen -lf`'s fingerprint among the closing steps (§6), and keeps it in `<git common dir>/<project>-ci/init/allowed-signers.next` for the person to copy into the signers pull request. This is a deliberate narrowing of the obvious reading of the flag.

`--signer auto` reads `user.email` as the principal (DESIGN §6.6) and the public key from `user.signingkey`: a `key::` literal as given, a path to a `.pub` file as read, a path to a private key by its `.pub` beside it. A key held only behind `gpg.ssh.defaultKeyCommand` is refused, because the generator cannot read it without running that command.

### 3.9 `AGENTS.md` and `CLAUDE.md`

Agents that work in an adopter's repository (Codex reads `AGENTS.md`, Claude Code reads `CLAUDE.md`) need the same few facts a new contributor does, or they push with `--no-verify`, edit `ci/kit/`, or widen a key to make a red go away. So `kit-init` writes one short section into each file: created when the file is absent, appended after one blank line when it exists, refused when the file already holds the section's begin marker (§2.3). `--no-agent-files` skips both; `--agent-files-only` writes only these two, for a project that adopted without the generator.

**The text is one template**, `kit/templates/agents/section.md`, and it is the only copy. The generator substitutes its tokens (§7.2); the kit's guide for adopting with an agent (`docs/adopt-with-an-agent.md`) points an agent at `ci/kit/templates/agents/section.md` in the vendored tree, or `kit/templates/agents/section.md` in the kit repository, and holds no copy of its lines; `kit/README.md` cites it the same way, and so does this document. A guard in the kit repository's own workflow fails when either of those two documents repeats a body line of the template. Why: two copies of an instruction drift apart, and an agent that reads the stale one does the wrong thing with confidence.

The section's markers are HTML comments, so they do not render. They would let a later release replace the section mechanically; that refresh is not built in this version (§1).

### 3.10 Untouched

`ci/kit/`, `ci/KIT-VERSION` and `ci/KIT-SIGNERS` belong to the vendoring (`kit/README.md` §5.3), and the generator only reads them. Any other workflow under `.github/workflows/` is left alone; the workflow check reads every workflow for W-1's trigger ban, so an existing `pull_request_target` workflow shows up in the checks of §5, as it would by hand.

## 4. Checksums and digests

### 4.1 The lookups

The generator's only network acts, each over HTTPS through Python's `urllib`, or through the Docker client for the digest:

| Value | Source | Why this source |
|---|---|---|
| Go tarball sha256, both architectures | `https://go.dev/dl/?mode=json&include=all`, the `sha256` of the version's two `linux-amd64` and `linux-arm64` archive entries | the Go project's own published checksums |
| Node tarball sha256, both architectures | `https://nodejs.org/dist/v<V>/SHASUMS256.txt`, the `linux-x64.tar.xz` and `linux-arm64.tar.xz` lines | the Node project's own checksum file |
| Python tarball sha256 and its release tag | `https://api.github.com/repos/astral-sh/python-build-standalone/releases` (unauthenticated, the ten most recent releases, `per_page=10`) to find the newest release carrying `cpython-<V>+<tag>-...-install_only_stripped.tar.gz` for both architectures, then that release's `SHA256SUMS` | CPython publishes no Linux binary; this project's builds are the relocatable ones the Python tool ecosystem installs, with a checksum file per release (§11) |
| Postgres image digest | `docker buildx imagetools inspect postgres:<major>`, the index digest on its `Digest:` line | the multi-architecture index, so a Mac and the runner pull the same image name by one digest, as the base image does |

Nothing is computed by downloading a tarball and hashing it: a hash of whatever the generator happened to receive proves nothing about what the project released. The release tarballs themselves are fetched later by the image build, and verified there against these checksums (§3.3).

Why ten releases: a larger page timed out inside the generator's 30-second bound (each release has about 990 assets), and python-build-standalone only builds the newest patch of each CPython minor in each release, so a pin missing from the ten newest releases is an old patch a larger page would seldom rescue. Such a pin is refused with the URL and the `--checksums` route, which is the documented way round it.

### 4.2 What it prints

Each value fetched is printed with its source, one line each, so that the person reviewing the landing pull request can re-check any of them: `go1.27.3 linux-amd64 65efbc80... from https://go.dev/dl/?mode=json&include=all`.

### 4.3 The offline path

`--checksums FILE` replaces every lookup. The file holds lines in the shape the official checksum files use, `<64 hex>  <name>`, where `<name>` is the release file name (`go1.27.3.linux-arm64.tar.gz`, `node-v24.22.1-linux-x64.tar.xz`, `cpython-3.12.7+20241016-aarch64-unknown-linux-gnu-install_only_stripped.tar.gz`, with the Python release tag read from that name) or `postgres:<major>` for the index digest. Lines it does not need are ignored. A value the file lacks is refused with its URL (§2.3); there is no falling back to the network once the flag is given, so the offline run is fully offline. The kit's own tests use this path (§7.3).

## 5. The commit and the checks

### 5.1 Before writing

The rendered manifest goes to a scratch file and through `bash ci/kit/bin/check-manifest --file <scratch>`: the parser and the rules that need no tree. Any finding refuses the run (§2.3).

### 5.2 The commit

Unless `--no-commit`, the generator stages exactly the files it wrote or appended to, plus `ci/kit`, `ci/KIT-VERSION` and `ci/KIT-SIGNERS` when the vendoring left them uncommitted, and makes one commit of those paths alone (`git commit -- <paths>`), with the message `ci: adopt quokka-ci-kit <tag>`, the tag read from `ci/KIT-VERSION`. Git's own configuration decides whether the commit is signed. When `git add` or `git commit` exits non-zero, it prints `The files are written but the commit failed (<step> exited <n>: <git's first line>), so nothing is committed; commit them yourself, then run the checks below.`, runs the checks anyway, and exits 1. Why commit at all: `make check`, the manifest check and the two Python checks all read the commit, not the working copy, so an uncommitted adoption cannot be checked; and committing only its own paths keeps the person's other staged or unstaged work out of it.

### 5.3 After writing

Run in order, each one's exit code printed as it stands and never through a pipe:

1. `bash ci/kit/bin/check-manifest` at HEAD: all twelve rules, including rule 7's vendored tree and rule 8's shim. With `--no-commit`, `--file ci/manifest` and its own sentence for the rules that did not run.
2. `python3 ci/kit/bin/check-workflow` (with `--manifest-file ci/manifest` under `--no-commit`). When `import yaml` fails under `python3 -I`, the generator prints `NOT RUN check-workflow: this python3 (<interpreter>) has no PyYAML; install it and run python3 ci/kit/bin/check-workflow yourself; CI's always-on job runs it too.` rather than a pass, and the run exits 3 when nothing else is wrong.
3. `python3 ci/kit/bin/check-escaping-reads` (likewise `--manifest-file`).
4. `bash ci/kit/bin/check-prereqs`, so that a missing Docker or an old git shows up before `make check` is tried.

`check-adapter` is not run: its second half reads the log of a real passing run, which only exists after `make check`. It is a closing step (§6).

## 6. What it prints at the end

After the checks, the steps for the person, in order, one line each, numbered, with the project's values filled in. The order is the first pull request's (DESIGN §9; `kit/README.md` §7):

1. `make check` (the first run builds the base and job images and takes several minutes).
2. `bash ci/kit/bin/check-adapter <job>` for each stack job, after `make check` passes.
3. Fill in each stub section and its image block, then run steps 1 and 2 again (printed only when a stub was written).
4. Copy the trust paragraph of the kit's DESIGN.md §3.4 into your own documentation before you trust a local record.
5. Protect `main`: pull requests only, signed commits required, no force pushes.
6. Confirm that a skipped required check reports success, with a scratch pull request (DESIGN §4 item 7).
7. Require these checks on `main`: `checks`, each stack job, `kit`.
8. On each workstation: `git config core.hooksPath .githooks`, `git config gpg.format ssh`, `git config user.signingkey <your public key>`.
9. Merge `origin/main` into this branch, then push it with `git push --no-verify -u origin <branch>`; this once only, so that CI runs every job and writes the first green records.
10. Open the pull request with the sentence that its local records are refused until a signer is on `main`.
11. After it merges, add your signer in a pull request of its own: `<the line>`, fingerprint `<SHA256:...>` (from `--signer`; without it, the command that makes the line from your public key).
12. Read the summary of the first scheduled run on its day (`--cron`'s day and time, UTC).

The steps are numbered in sequence: when no stub was written, step 3 is not printed and the steps after it close up, because a gap in a numbered list reads as a missing step.

When `check-manifest` exited 0 and `check-escaping-reads` exited 2 (no scan root holds a test file), one step comes before step 1: `Write a first test in <the scan directories>, commit it, and run python3 ci/kit/bin/check-escaping-reads again; CI's checks job refuses this tree until you do.` The scan line is still written (§11).

Above the numbered steps, two lines appear when they apply: `kit-init took the main branch to be main, because origin/HEAD is not set here. If it is another branch, run git remote set-head origin -a, reset this branch to before the adoption commit and run kit-init again.` when `origin/HEAD` is unset, and, for each check that did not run, `<check> did not run, so this run proves nothing about it: install PyYAML for python3 and run python3 ci/kit/bin/<check> before you push.` They are facts about this run, to be read before the steps, not steps of the landing.

Then one line on what was declared by hand: for each Python job, that its escaping reads are a census by hand (`kit/README.md` §3.4).

## 7. The templates

### 7.1 Where they live

`kit/templates/` inside the vendored tree, so that every template is part of the `kit/` tree a release signs and `kit-update` verifies, and `ci/kit/templates/` in an adopter. There are 35 files:

```
kit/templates/
  manifest/head, manifest/tail                  the fixed lines and the grouped lines' frame
  manifest/go, manifest/node, manifest/python   one stack job's lines
  manifest/stub                                 a stub job's lines
  manifest/node-npm-ci                          the install line, for a Node job with a lockfile
  manifest/node-scan                            the ts escaping-scan line, for every Node job
  manifest/postgres                             the service line
  adapter/head.sh, adapter/tail.sh              the helpers; the dispatching case
  adapter/case.sh                               one case line per job, rendered into the tail
  adapter/go.sh, adapter/node-vitest.sh, adapter/python-pytest.sh, adapter/stub.sh
  adapter/node-tsc.sh                           the tsc step, when tsconfig.json exists
  adapter/postgres.sh                           the URL variable's refusal, for a job with a service
  image/head.Dockerfile, image/tail.Dockerfile  the base FROM; the job user
  image/run.Dockerfile                          the one RUN and ENV that every tool's parts fill
  image/go.Dockerfile, image/node.Dockerfile, image/python.Dockerfile
                                                one tool's parts, in @@@ sections (§7.2)
  workflow/head.yml                             the triggers and the always-on job
  workflow/output.yml                           one output pair per job, in @@@ sections
  workflow/local-job.yml                        one W-7 job per stack job
  workflow/cache-gomod.yml, workflow/cache-gobuild.yml, workflow/cache-npm.yml
  workflow/kit-job.yml, workflow/prune.yml
  Makefile.check
  allowed-signers
  agents/section.md
```

The shim is not here: it is copied from `kit/hooks/pre-push.shim` (§3.5).

### 7.2 Substitution

A closed set of `@@NAME@@` tokens, replaced by plain string substitution. There are 51 names, each used by some template and none outside the set:

- the project and its fixed names: `PROJECT`, `PROJECT_UPPER`, `MAIN`, `CRON`, `ADAPTER`, `WORKFLOW`, `SIGNERS`, `CHECK`;
- a job: `JOB`, `FN` (the job with `-` written `_`, for the section's function name), `DIR`, `PATHS`, `PIN_FILE`, `PIN_BASE`, `SCAN_DIR`, `LOCKFILE`, `REQUIREMENTS`, `NEED_BINS`, `TSC_STEP`, `SERVICE_CHECK`;
- lists over all jobs: `JOBS`, `KEEP_GREEN`, `IMAGE_PATHS`, `CASES`, `OUTPUTS`, `CACHES`, and the manifest's grouped lines `LINES_JOB`, `LINES_PATHS`, `LINES_PIN`, `LINES_COUNT`, `LINES_INSTALL`, `LINES_SERVICE`, `LINES_CACHE`, `LINES_PIN_SOURCE`, `LINES_PROBE`, `LINES_ESCAPING_TREE`, `LINES_ESCAPING_SCAN`;
- a tool and a service: `VERSION`, `SHA256_AMD64`, `SHA256_ARM64`, `PBS_TAG`, `DIGEST`;
- the image's assembled parts: `ARGS`, `PIN_FILES`, `CROSS_CHECKS`, `AMD64`, `ARM64`, `TOOL_NAMES`, `FETCH`, `FINISH`, `ENV`.

Three rules decide the output, each enforced by the generator:

1. **A token outside the set is a defect in the templates**, refused with exit 2 before anything is written; **a token left unreplaced** in any file the run would write is refused the same way.
2. **A line that consists of one token alone, whose value is empty, is dropped**, so that an absent part leaves no blank line behind.
3. **A part that repeats is rendered once per item.** Most such parts are a file of their own (a job's manifest lines, a section, a case line, a cache step). Two kinds of template hold several named parts in one file, each part opening with a line `@@@ <part>` that is read as data and never output: a tool's image block (`name`, `args`, `check`, `amd64`, `arm64`, `fetch`, `tmp`, `dirs`, `path`, `env-comment`, `env`), whose parts are spread across the one `RUN` and `ENV` of `image/run.Dockerfile`, so keeping one tool's parts in one file keeps that tool reviewable in one place; and `workflow/output.yml` (`decision`, `key`). A `@@@` line is a part header only in those four files.

No template language, no `eval`, no shell: the generator is standard-library Python. Why so plain: a template engine would be a dependency on every workstation, and logic inside templates is code the checks cannot see.

### 7.3 How they stay true

- **Every supported shape is generated and checked in the kit's own tests.** The K-init family in `kit/tests/run` (no Docker daemon; offline through `--checksums` and the existing `docker` shim) generates each shape of §10's AC-KI-30 into a fixture repository with a vendored `ci/kit` copy and requires: the generator's exit 0; `check-manifest` at HEAD (rules 1 to 12); `check-workflow`; `check-escaping-reads`; and the test script's unknown-job case run directly (`bash ci/ci-job.sh no-such-job` exits 2 with no counts line, the first half of `check-adapter`), because `check-adapter` itself needs the job image.
- **Two shapes end to end.** `kit/tests/image` has a family that generates the Go, Node and Postgres shape, builds its job image against the real checksums of a fixed recorded version, runs `local-ci run` for each job and then `check-adapter` for each stack job; and the same for the Python with pytest shape against a recorded python-build-standalone release.
- **The example as a golden output.** The fixture directory holds the example's six files for the `rottnest` shape; K-init compares its output with them, comment lines set aside, so a template that drifts from the reference adopter fails the test.
- **Plants.** `kit/tests/plants` has planted defects for the rules a single deleted line could remove: the existing-file refusal, the Dockerfile's pin cross-check, the shim copied from anywhere but the kit's file, an unreplaced token passing, the restore truncating by decoded length, the UTF-8 refusal, the offline branch, the pinned Python off the image's `PATH`, the symbolic-link refusal and the bounded names.
- **Fixtures no adopter's test runner collects.** The kit travels whole into every adopter at `ci/kit/`, so no file under `kit/` has a name a test runner or compiler collects by default from a repository's top (AC-KI-38): fixture sources carry `.fixture` after their real name, and the helpers that copy them into a scratch repository strip it.

## 8. Why the generator is Python

The kit's host scripts are bash 3.2 under a closed allow-list of commands (`kit/README.md` §2.2), and "no host tool parses any output". The generator has to fetch over HTTPS (`curl` is not on the list), read Go's JSON release index, read `package.json`, `go.mod` and the requirements files of each directory, and render templates with repeated parts. Doing that in bash 3.2 would mean either widening the allow-list for a one-off tool, which weakens a check every everyday host script lives under, or parsing JSON with `[[ =~ ]]`, which is exactly the fragile host parsing the rule exists to keep out.

Python is acceptable here because the kit already ships two Python tools a person runs on a workstation by hand, the workflow check and the escaping-read check, and they sit outside `check-host` by its own file list. `kit-init` joins them on the same terms. macOS provides `python3` through Apple's command line tools (3.9 at the oldest supported release), so the floor is Python 3.9 and the standard library: no PyYAML, since the generator writes YAML from templates and never parses it.

`check-host` is unchanged: its file list keeps its nine bash entries, and `kit-init` is not added, because it is not bash. Instead the generator carries two rules of its own, each proven in K-init:

1. **Its subprocesses are a closed set**, called with an argument vector, never through a shell: `git`, `docker` (the digest lookup only), `ssh-keygen` (the fingerprint only), and `bash` running the kit's own `check-manifest` and `check-prereqs`, plus `python3` (`sys.executable`) running the two Python checks and the PyYAML probe, always in isolated mode (`-I`), so that neither the current directory, `PYTHON*` variables nor the user's site-packages reach its path. A test reads the script's syntax tree and fails on any other first argument or on `shell=True`.
2. **Its syntax parses as Python 3.9**: a test runs `ast.parse(source, feature_version=(3, 9))`, so a newer construct cannot slip in on a contributor's newer Python.

## 9. Releases

### 9.1 The version

`kit-init` shipped as a MINOR release, `v1.1.0`: it added a command and templates, while the manifest format stayed `manifest v1`, the test script's contract, the key formula and the plan's sentences were unchanged, and nothing an adopter had written needed to change. Later moves of the workflow template's action shas, or of the default Postgres major, are PATCH releases: they change the generated output for new adopters and no interface.

### 9.2 Copies of the kit maintained elsewhere

A fork of the kit maintained separately does not receive the generator automatically, and nothing here depends on it doing so. A later port would need: the templates with the fork's own constants substituted (its image label, its default `kit-update` URL, its release tag message, its name in the agent section's heading and markers; `kit/README.md` §2.1); the fork's own example as the golden output; the K-init family and plants run against the fork's tree; and its own MINOR release. The generator's code carries no kit name outside one constant block for exactly this reason.

## 10. What the tests prove

Each criterion is checkable by reading a file or running a command. The ids are the ones the test cases cite.

**The command**

- **AC-KI-1** `kit/bin/kit-init` exists, mode 0755, first line `#!/usr/bin/env python3`, and imports only the standard library.
- **AC-KI-2** Its syntax parses under `ast.parse(..., feature_version=(3, 9))`; on a Python below 3.9 it exits 2 with its sentence.
- **AC-KI-3** Every subprocess call names `git`, `docker`, `ssh-keygen`, `bash` or `sys.executable` as its first argument, as a list, and none passes `shell=True` (a syntax-tree test).
- **AC-KI-4** Every refusal of §2.3 has a K-init case that triggers it, sees exit 2 and its sentence, and sees the working tree, the index and HEAD unchanged.
- **AC-KI-5** A run refused after rendering (a finding on the rendered manifest) writes nothing; a write failure removes only the files the run created.
- **AC-KI-6** With `--checksums`, no network is used: the K-init family runs with no network and the `curl` shim unreachable, and passes.
- **AC-KI-7** A value missing from `--checksums` is refused with its URL; there is no fallback to the network.
- **AC-KI-8** Nothing is written outside the working copy and `<git common dir>/<project>-ci/init/`; at exit that directory holds at most `allowed-signers.next`.
- **AC-KI-9** A write failure planted part way through the writes leaves every file the run created removed and every file it appended to byte-equal to its original, a Makefile with CRLF line endings among them, and the run exits 2 with its sentence.

**What it writes**

- **AC-KI-10** The manifest it writes passes `check-manifest` at HEAD, rules 1 to 12, for every shape of AC-KI-30.
- **AC-KI-11** `paths` for each stack job is its directory, then each other image input it does not cover, then `ci`; `image-path` is `ci/image` then each pin-source file in flag order.
- **AC-KI-12** The workflow passes `check-workflow` for every shape, and its schedule equals the manifest's `weekly-cron`.
- **AC-KI-13** The test script's unknown-job case exits 2 and prints no counts line for every shape; a stub section exits 2 with its sentence and no counts line.
- **AC-KI-14** `.githooks/pre-push` is byte-equal to `ci/kit/hooks/pre-push.shim`, mode 0755.
- **AC-KI-15** The Dockerfile holds exactly one `FROM ${CIKIT_BASE}`, both architecture checksums per tool, the cross-check of each pin file, and the cache directories of exactly the declared cache kinds.
- **AC-KI-16** `ci/allowed-signers` in the landing commit holds only its comment line, with or without `--signer`; with `--signer`, the printed line and fingerprint equal `allowed-signers.next` and `ssh-keygen -lf` of the key.
- **AC-KI-17** An existing `Makefile`, `.gitignore`, `AGENTS.md` or `CLAUDE.md` keeps every original byte; the addition follows after one blank line.
- **AC-KI-18** The agent section in both files is the rendered `kit/templates/agents/section.md` with no `@@` token left; `--no-agent-files` writes neither; `--agent-files-only` writes only these two, reading every value from `bin/keys dump HEAD`.
- **AC-KI-19** No token `@@...@@` survives in any written file.
- **AC-KI-20** On the example's tree with its six files removed, `kit-init --project rottnest --go api --node web --postgres api --checksums <the example's values>` writes files equal to the example's with comment lines set aside.

**The commit and the checks**

- **AC-KI-21** Without `--no-commit`, HEAD is one new commit whose message is `ci: adopt quokka-ci-kit <tag>` and whose changed paths are exactly the generator's files plus any uncommitted vendoring; other staged work stays staged and uncommitted.
- **AC-KI-22** After writing it runs the four checks of §5.3 in order and prints each exit code; a missing PyYAML prints the `NOT RUN check-workflow` line, never a pass, and the run exits 3 when nothing else is wrong.
- **AC-KI-23** The closing steps of §6 are printed in that order, one line each, with the project's job names.

**The templates and the documents**

- **AC-KI-24** Every template lives under `kit/templates/`; the generator reads nothing else but `kit/hooks/pre-push.shim`, `ci/KIT-VERSION`, `bin/keys` output and the adopter's own files.
- **AC-KI-25** `check-host kit/` passes unchanged, and its file list is unchanged.
- **AC-KI-26** `kit/README.md` names `kit-init` in §1's table, §2.2 and the prerequisites, and holds no copy of the agent section's body lines.
- **AC-KI-27** The kit repository's workflow fails when `docs/adopt-with-an-agent.md` or `kit/README.md` repeats a body line of `kit/templates/agents/section.md`, and passes when they cite its path; the guard has been seen to fail.
- **AC-KI-28** Each planted defect of §7.3 turns K-init red (the plant of the Python image block turns `initpy` red, K-init's end-to-end half), and no other family.
- **AC-KI-29** The end-to-end family in `kit/tests/image` generates the Go, Node and Postgres shape, `local-ci run` passes each job, and `check-adapter` passes each stack job.
- **AC-KI-30** K-init covers at least these shapes: Go alone; Node with vitest alone; Go, Node and Postgres (the example's); Python with pytest alone; Node without vitest (a stub); two Go directories with one toolchain; no stack at all; a stack directory of `.`.
- **AC-KI-31** The CHANGELOG entry for the generator's release has the usual shape and the release is MINOR.
- **AC-KI-32** `check-manifest` rule 6 finds nothing for an `escaping-tree` or `escaping-scan` path of `.`, and still finds a path absent at the commit.
- **AC-KI-33** `check-escaping-reads`, given a manifest with no `escaping-scan` line, prints `The manifest names no escaping-scan line, so no file was scanned and only the escaping-read lines were checked against the keys.`, exits 0 when every `escaping-read` line is keyed with `OK: no file was scanned; <n> escaping-read lines were checked, and every one is keyed.`, and exits 1 when one is not; given an `escaping-scan` root holding no test file, it still exits 2 with `the escaping-read fence found no test files, so it would prove nothing.`
- **AC-KI-34** Under `escaping-tree <job> .`, the check counts a file's depth as it would the same relative position under a named tree: `db/x_test.go` calling `filepath.Join("..", "..", "x")` is a finding.
- **AC-KI-35** A file `kit-init` reads that does not decode as UTF-8 (`ci/KIT-VERSION` among them) is refused with exit 2 and `<path> is not UTF-8 text kit-init can read, so nothing was written.`, and nothing is written.
- **AC-KI-36** The end-to-end family in `kit/tests/image` generates the Python with pytest shape against a recorded python-build-standalone release and its real checksums, and `local-ci run` passes the job with the pinned interpreter answering (not the base image's own `python3`), `pip install --require-hashes` succeeding inside the job, and a counts line whose `pytest_tests` equals the fixture's passing tests; `check-adapter` then passes the job. It runs in the kit's own workflow (amd64), and each release records one green run of AC-KI-29 and AC-KI-36 on an arm64 workstation as well.
- **AC-KI-37** Under a tree of `.`, `check-escaping-reads` accepts a keyed `escaping-read` line, scans a TypeScript file under the `test/` directory of a `ts` scan root of `.`, and does not call a tree of `.` "declared by hand" when a scan root lies inside it; K-init's `.` shape carries one declared, keyed read.
- **AC-KI-38** No file under `kit/` ends in `.go`, `.ts`, `.tsx`, `.mts`, `.cts`, `.js`, `.jsx`, `.mjs` or `.cjs`, or is named `test_*.py`, `*_test.py` or `conftest.py`; a K-init case checks this over the kit under test and fails on a planted file, and every K-init, `initgo` and `initpy` case passes on the renamed fixtures.
- **AC-KI-39** For a Go or Node directory with no test file, `kit-init` writes the `escaping-scan` line, exits 1, and prints the step of §6 naming that directory before `make check`.
- **AC-KI-40** A target that is a symbolic link, or lies under one, is refused with exit 2 and `<path> is a symbolic link, and kit-init writes no file through one, so nothing was written.`, one sentence per link, on the adoption route and on `--agent-files-only`, and the file the link names is byte-equal afterwards. A link inside the working copy is refused too. Plant: `init-symlink-targets`.
- **AC-KI-41** A stack directory outside the bounded shape (a command substitution; a quote) is refused with exit 2 and the sentence naming the shape, no `ci/ci-job.sh` is written, and nothing in the name runs. Plant: `init-bounded-values`.
- **AC-KI-42** A `yaml.py` at the repository's top is never imported: after a run, no file it would write exists and no `__pycache__` holds it.
- **AC-KI-43** Under a `python3` that cannot import PyYAML, the run writes and commits, prints the `NOT RUN check-workflow` line with its interpreter and the closing not-run line, and exits 3. Under one that can, the same run exits 0.
- **AC-KI-44** A commit refused by a pre-commit hook exits 1, prints the commit-failure line with `git commit`, its code and the hook's message, and leaves HEAD where it was.
- **AC-KI-45** A detached HEAD is refused with exit 2 and its sentence, and with `--no-commit` it is not refused. With `origin/HEAD` unset, the closing lines say the main branch was guessed and how to set it, and with it set they do not.

## 11. Decisions taken along the way, and known limitations

Decisions, each with its reason:

- **Python toolchains come from python-build-standalone**, verified against that project's per-release `SHA256SUMS`, and the generated Dockerfile's comment names the source. CPython publishes no Linux binary tarball, and these are the relocatable builds the Python tool ecosystem installs. The Python shape is proven end to end before every release that touches it (AC-KI-36), because the Python section has risks no check sees: the base image carries Debian's own `python3`, so the section must be shown to answer with the pinned interpreter; `pip install --require-hashes` must reach the index over the job's network; and the archive must extract where the `PATH` expects it.
- **The signer line never goes into the landing commit**, even with `--signer` (§3.8). Writing it there would save the second full CI run that DESIGN §9 step 4 costs, at the price of hiding the adoption's one trust decision inside a large generated pull request.
- **The default weekly minute is the example's**, `20 6 * * 1`. A minute derived from the project name would spread the load on shared runners, but a value the person can read in the example is easier to reason about than a derived one; revisit if adopters report queueing.
- **Both agent files are written by default** when neither exists, because a repository without them is exactly the one an agent most needs telling; `--no-agent-files` opts out.
- **A stub refuses rather than passes**, and a stack directory with no test file still gets its `escaping-scan` line, with a closing step telling the person to write a first test (§6). Leaving the line out would leave every test written later unscanned, and counting test files in the generator would be a second copy of the check's own rule, which would drift.
- **No `--main` flag.** When `origin/HEAD` is unset the generator takes `main`, says so, and prints the command that turns the guess into an answer (`git remote set-head origin -a`). A flag would widen the interface for a value git already holds, and the manifest's `main-branch` line is in the landing diff a reviewer reads.
- **A symbolic link anywhere in a target path is refused**, even one that stays inside the working copy. One test with no path arithmetic, and no file the generator writes has any reason to be a link. The rule exists because a committed link would have turned a maintainer's run into a write to any file they could write, with nothing showing in the commit.
- **Directory and file names are bounded, not quoted.** Every name the generator puts into a generated file is letters, digits and `. _ / -`. Quoting every token in every template would be many changes across three kinds of quoting (shell, Dockerfile, the workflow's expressions); the bound is one function, and real projects' names already fit it. The manifest's own path rule is unchanged, so a hand-written manifest is held exactly as before.
- **The generator's own Python runs isolated (`-I`)**, so a `yaml.py` at the top of the repository is never imported and the adoption guide's promise holds: no repository code runs before `make check`. The cost is one honest exit 3 on a machine whose PyYAML lives in the user site or on `PYTHONPATH`, where `check-workflow` cannot see it under `-I`; the remedy it prints, running the check yourself, works because that run is not isolated.
- **Exit 3 when a check did not run.** Before this, a run whose workflow check could not run exited 0, and an agent reading the exit code alone was told every check had passed.
- **The fixture sources carry `.fixture`**, so that a project whose pytest, vitest or root `tsconfig.json` discovers from the top never runs the kit's own fixtures as its tests. Measured before the fix: a scratch root holding `ci/kit` and one test of its own gave pytest `2 failed, 5 passed`, both failures the kit's.

Known limitations, left for later releases, each with the reason it waits:

| What | Why it waits |
|---|---|
| The python-build-standalone tag from the network is not checked against `^\d{8}$`, as the offline path checks it | the tag is bounded by git's refname rules and must match both the asset name and `SHA256SUMS`, and as an `ARG` value it cannot inject an instruction; the check would be for consistency, not to close a hole |
| Checksums are trusted on TLS alone (Go's index, Node's `SHASUMS256.txt` without its `.sig`, Python's `SHA256SUMS` from the same release as the archive); redirects are followed and reads are not capped | nothing fetched is executed, every value is checked as 64 hex and printed with its source for the reviewer, and `--checksums` exists for a person who verifies out of band. Cross-checking GitHub's asset `digest` is the next step |
| The generated `prune` job is outside the workflow check (W-8, W-11 and W-12 iterate the manifest's jobs only) | as generated it is correct (actions pinned by sha, `main` only, kit code at `main`'s sha); fencing it is a `check-workflow` change, which belongs in its own release with its own tests |
| `pip install --require-hashes` may not hash-check an sdist's build dependencies | needs verifying first. The fix (`--only-binary :all:`) would refuse projects that have no wheels, so it is a deliberate change to the generated Python section, not a fix |
| Hygiene: `--no-same-owner` on the Go and Python extracts; `curl --proto-redir =https`; the signer principal unvalidated; `--agent-files-only` raising a traceback rather than a refusal when `keys dump` lacks a directive | each comes after a check that already holds (the sha256 before the extract) or touches only the person's own value |
| The escaping-read check passes a manifest with no `escaping-scan` line (AC-KI-33) | a deliberate ruling, visible in the manifest diff, and it re-runs every job. A tighter form (refuse while any job pins `go` or `node`) is a check change for its own release |
| The `NOT RUN` sentence says "install it" where PyYAML is installed but unseen under `-I` | the exit code is honest and the remedy it prints works; rewording it to name isolated mode is a copy change for a later PATCH |
| The file-name half of the bounded-value refusal has no test case (a hashed `requirements-é.txt`) | the same function decides both sentences, and the `init-bounded-values` plant covers the directory case |
| No plants yet for the `-I` flag, exit 3, the commit-failure line and the detached-HEAD refusal | their cases were seen to fail against the generator before each fix; a plant guards against a later change to the case itself, and each is one line, for the next touch of `kit/tests/plants` |
