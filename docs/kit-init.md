# `kit-init`: the generator of an adopter's files

```text
        .-"""-.        ______
       / .===. \      |______|
       \/ ^ ^ \/     _|______|
       ( \___/ )___/  |______|    six files, one init
    ____ooo___ooo_____|______|____
```

Status: specification, not yet built. It is written for the kit's maintainers and contributors. It names what `kit-init` does, what it writes and why, how its templates stay true, and how it is released. Where it cites the kit, the line numbers are those of the `v1.0.0-rc.1` tree (`kit/` tree `16d76d5c`, commit `4083f42`); where it cites the example adopter, they are those of [`quokka-ci-kit-example`](https://github.com/aames/quokka-ci-kit-example) at `8ad9623`, written `example:<path>:<line>`.

## 1. Purpose and scope

**What it changes.** Today an adopter writes six files by hand (the manifest, the adapter, the job image's Dockerfile with each toolchain pinned by sha256 per architecture, a workflow of about 260 lines, the hook shim and a Makefile target), plus the signers file, and lands them through the fourteen steps of the adoption table (`kit/README.md:612-631`). With `kit-init` adoption becomes four acts:

1. vendor the kit (`kit-update <tag>`, unchanged, `kit/README.md:359-373`);
2. `python3 ci/kit/bin/kit-init --project NAME --go api --node web ...`;
3. `make check`;
4. push the branch and open the landing pull request.

The generator writes complete files for the stacks it knows (Go, Node with vitest, Python with pytest) and an honest stub for anything else: a stub arm that refuses with a sentence saying what to fill in, so a stub can never pass.

**Why a generator and not only a template repository.** A template repository (the quickstart) gives one shape. Every adopter's shape differs in its directories, job names, toolchain versions and services, and each of those appears in several files at once (a pin file is a `pin-source`, an `image-path`, a path in every local job's key, a `COPY` in the Dockerfile and a cross-check in the Dockerfile's `RUN`). Writing those consistently by hand is where adoption goes wrong; deriving them from one set of flags is what a generator is for.

**What it is not.** It is not a second source of law. Everything it writes is held by the same lints an adopter's hand-written files are held by (`check-manifest`, `check-workflow`, `check-escaping-reads`, `check-adapter`), and the kit's tests hold every shape it can produce green through them (§7.3). It runs once, at adoption. It never rewrites a file it wrote earlier.

**Deliberately out of scope**, each for a reason:

| Not generated | Why |
|---|---|
| `ci-only` and `content` jobs | each is project-specific by definition (a host assertion, an apply step); there is no template that would be honest |
| a plan extension (`kit/README.md:330-332`) | it decides jobs outside the scheme, which the generator cannot know |
| `socket` and `history` jobs | each widens a job's reach (`kit/README.md:175-176`); that is a decision a person makes and states, never a default |
| `env` lines and `not-run-allowed` lines | an `env` line is a setting the generator cannot know; an allow-list entry exists only for a leg proven impossible on a workstation (`kit/README.md:606-609`) |
| package managers other than npm, test runners other than vitest and pytest, languages other than Go, Node and Python, services other than Postgres | the manifest's closed kinds are `npm-ci` and `postgres` (`kit/README.md:173-174`); other stacks get a stub arm and stub image block |
| regenerating or upgrading generated files | the files are the adopter's own once written (`kit/README.md:37-44`); a later kit release says in its CHANGELOG what an adopter must change by hand |
| the residual paragraph (DESIGN §3.4) | it belongs in the adopter's own documentation, in their words around the copied paragraph; the generator prints it as a step |

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

Run at the top of a working copy, on a branch that is not the manifest's `main-branch`, after `kit-update` has written `ci/kit/`, `ci/KIT-VERSION` and `ci/KIT-SIGNERS`. The file is executable with a `#!/usr/bin/env python3` line, so `ci/kit/bin/kit-init` also works; the `python3` form is the documented one, as it is for the two Python fences (`kit/README.md:25-26`).

### 2.2 The flags

| Flag | Takes | What it produces | Default |
|---|---|---|---|
| `--project NAME` | the project word | the manifest's `project` line and every name derived from it (`kit/README.md:191-208`): the counts prefix, the cache paths in the workflow, the service URL variable | required |
| `--go DIR[:JOB]` | a directory holding `go.mod`; repeatable | one `local` job: manifest lines (§3.1), a Go arm (§3.2), the Go block of the image (§3.3), a W-7 job with the `gomod` and `gobuild` caches (§3.4) | none |
| `--node DIR[:JOB]` | a directory holding `package.json` and `.nvmrc` (or `.node-version`); repeatable | one `local` job: manifest lines with `install <job> npm-ci <dir>` when `package-lock.json` exists, a vitest arm or a stub arm, the Node block of the image, a W-7 job with the `npm` cache | none |
| `--python DIR[:JOB]` | a directory holding `.python-version`; repeatable | one `local` job: manifest lines, a pytest arm or a stub arm, the Python block of the image, a W-7 job with no cache step | none |
| `--postgres JOB[:MAJOR]` | a job one of the stack flags declared; repeatable, once per job | a `service <job> postgres` line with the image pinned by digest, the URL variable `<PROJECT>_TEST_PG_URL`, CI's two flags, and the arm's refusal when the variable is unset | major `18`, the example's (`example:ci/manifest:47`) |
| `--signer auto\|FILE` | `auto`, or a file holding one SSH public key line | the first signer's allowed-signers line and its fingerprint, printed for the signers pull request and kept at `<git common dir>/<project>-ci/init/allowed-signers.next`; never written into the landing commit (§3.8) | none: the closing steps say how to make the line |
| `--cron 'M H D M W'` | five cron words | the manifest's `weekly-cron` and the workflow's one schedule, equal by construction (W-1, `kit/README.md:83`) | `20 6 * * 1`, the example's (`example:ci/manifest:16`) |
| `--checksums FILE` | a file of `<64 hex>  <name>` lines | the toolchain checksums and the service digest, read from the file instead of fetched (§4.3) | fetched |
| `--no-commit` | nothing | the files are written and not committed; the lints then run on the working files (§5) | commit |
| `--no-agent-files` | nothing | neither `AGENTS.md` nor `CLAUDE.md` is written or appended to | both written |
| `--agent-files-only` | nothing; refuses every other flag | only the two agent sections, their values read from the committed manifest through `bin/keys dump HEAD` (`kit/README.md:13`), for a project that adopted by hand or before this command existed | off |

**Job names.** `JOB` defaults to the directory with `/` written as `-` (`services/api` gives `services-api`), and to the stack's name for `.` (`go`, `node`, `python`). It must be a manifest NAME (`kit/README.md:135`). The generator also declares two jobs of its own, the always-on `checks` and the kit job `kit`, and writes a workflow job `prune`, so `checks`, `kit` and `prune` are refused as stack job names, as are `image` and `base`, which the manifest itself refuses (`kit/README.md:137`). Why fixed names for the two kit jobs: they are the example's (`example:ci/manifest:11,13`) and every reader of the kit's documents meets them there; a project that wants others renames them in its own pull request later, and the lints hold the rename.

**Repeated stacks.** Two `--go` directories make two jobs in one image, and the image carries one Go. So every Go directory's `toolchain` line must name the same version; the first directory's `go.mod` is the one `pin-source go` line and the one image input, and each later job's arm still reads its own `go.mod` at run time, so a later drift is refused by that arm (§3.2). The same holds for Node and Python. Why: `pin-source` takes one line per tool (`kit/README.md:185`), and an image with two Go versions would be two images.

### 2.3 The refusals

Every refusal is one sentence ending `so nothing was written.`, printed before any file is touched, with exit 2. The generator renders every file in memory first, then checks every refusal, then writes; a write that fails part way removes the files this run created and nothing else.

| Refusal | Sentence (the shape; the build may polish the words) |
|---|---|
| not at the top of a git working copy | `kit-init runs at the top of a git working copy, so nothing was written.` |
| the kit is not vendored | `There is no ci/kit/lib/keys.sh or ci/KIT-VERSION; vendor the kit with kit-update first, so nothing was written.` |
| a file it would write exists | `ci/manifest already exists, so nothing was written.` (one sentence per such file: `ci/manifest`, `ci/ci-job.sh`, `ci/image/Dockerfile`, `.github/workflows/ci.yml`, `.githooks/pre-push`, `ci/allowed-signers`) |
| a file it appends to already holds what it would add | `The Makefile already has a check target, so nothing was written.`; `AGENTS.md already holds the quokka-ci-kit section, so nothing was written.` (likewise `CLAUDE.md`) |
| an unknown flag or stack | `--rust is not a stack kit-init knows (it knows --go, --node and --python), so nothing was written.` The closing sentence of the usage names the stub route: a stack it does not know is declared by hand after the run. |
| the project word | `<word> is not a project name the manifest accepts (a lower-case letter, then 1 to 15 lower-case letters or digits), so nothing was written.` The test is the manifest's own rule (`kit/README.md:144`), and the rendered manifest goes through the parser before anything is written (§5.1), so the generator cannot hold a looser rule than the lint |
| a job name out of shape, reserved or repeated | `<name> cannot name a job (<why>); give the directory a name with DIR:JOB, so nothing was written.` |
| a directory without its stack file | `api has no go.mod, so it is not a Go directory and nothing was written.` |
| a pin that is absent or not exact | `api/go.mod has no toolchain line, and the job image is pinned from it; add one such as "toolchain go1.27.3", so nothing was written.` For Node: `web/.nvmrc holds "lts/*", which is not an exact version such as 24.22.1, so nothing was written.` For Python: `api has no .python-version; pyproject.toml's requires-python is a range, not a pin, so nothing was written.` |
| repeated stacks that disagree | `api/go.mod pins go1.27.3 and worker/go.mod go1.27.2, and the job image carries one Go, so nothing was written.` |
| `--postgres` naming no stack job | `--postgres names db, which no stack flag declared, so nothing was written.` |
| no git identity | `git has no user.name or user.email here, and the adoption commit and the signer's principal need them, so nothing was written.` (with `--no-commit` and no `--signer` nothing needs an identity, so this refusal is not raised) |
| `--signer auto` with no usable key | `--signer auto needs gpg.format ssh and a user.signingkey whose public key can be read, so nothing was written.` |
| `--signer FILE` not one key | `<file> holds no single SSH public key line of a type the signers check knows, so nothing was written.` (the types are DESIGN §6.6's list) |
| a checksum or digest that cannot be had | `The sha256 of go1.27.3.linux-arm64.tar.gz could not be fetched from https://go.dev/dl/?mode=json&include=all; fetch it yourself and pass it with --checksums, so nothing was written.` (one sentence per missing value, each naming its URL, and the file shape of §4.3) |
| the current branch is the main branch | `kit-init commits the adoption on a branch for its own pull request, and this is main, so nothing was written.` (not raised with `--no-commit`) |
| the rendered manifest fails the lint | the lint's own findings, then `The manifest kit-init would write does not pass check-manifest, so nothing was written.` A failure here is a defect of the generator, and the kit's tests exist to make it impossible (§7.3) |
| a Python below 3.9 | `kit-init needs python3 3.9 or later, and this is <version>, so nothing was written.` |

### 2.4 What it never does

- **No push.** The landing push is the person's, after `make check` (§6).
- **No protection or settings change.** Branch protection, required checks and Actions settings stay a person's acts on the host (`kit/README.md:618,625`).
- **No key generation, and no key in the landing commit.** It reads the existing signing key's public half for `--signer auto` and nothing else.
- **No network beyond the lookups of §4**, each to a named host, all skipped with `--checksums`.
- **Nothing outside the repository.** Its scratch files live in `<git common dir>/<project>-ci/init/`, under the kit's state directory (`kit/README.md:123`), and are removed at exit but for `allowed-signers.next`.
- **No edit of an existing line** in any file. It creates files, and appends to `Makefile`, `.gitignore`, `AGENTS.md` and `CLAUDE.md`; it never changes `ci/kit/`, `ci/KIT-VERSION` or `ci/KIT-SIGNERS`.
- **No run of `make check`.** Building the job image takes minutes and is the next act the person takes, in the order of §6.

### 2.5 Exit codes

`0`: written (and committed), every lint held. `1`: written, and a lint after writing found something; the findings are printed and the files stay for the person to fix. `2`: refused; nothing was written.

## 3. What it writes, file by file

The example adopter is the reference output: `kit-init --project rottnest --go api --node web --postgres api` run on the example's tree with its six files removed must reproduce them, equal line for line once comment lines are set aside, with the Postgres digest as the one value that may differ (AC-KI-20). Every rule below is derived from that output.

### 3.1 `ci/manifest`

Written from `kit/templates/manifest/` in the order of the example (`example:ci/manifest:1-61`), each block from a census the generator can do without running anything.

| Directive | Value | Where it comes from |
|---|---|---|
| `manifest`, `key-epoch`, `local-epoch` | `v1`, `1`, `1` | fixed (`example:ci/manifest:5,7,8`) |
| `project` | `--project` | flag |
| `main-branch` | the remote's default branch as `refs/remotes/origin/HEAD` names it, else `main` | git; the plan and the prune read main from `origin` (`kit/README.md:22`) |
| `workflow`, `adapter`, `image`, `signers` | `.github/workflows/ci.yml`, `ci/ci-job.sh`, `ci/image/Dockerfile`, `ci/allowed-signers` | fixed: the example's paths (`example:ci/manifest:10,12,14,21`), all under `ci` but the workflow, so `ci` in each key covers them (rules 3, 5, 9; `kit/README.md:278`) |
| `always-on`, `kit-job` | `checks`, `kit` | fixed (§2.2) |
| `check-command` | `make check` | fixed; the Makefile target of §3.6 |
| `weekly-cron` | `--cron` | flag; the minute 0 is refused by the lint's rule 12 (`kit/bin/check-manifest:435-436`) |
| `prune-days`, `lock-wait-minutes`, `refuse-drafts`, `user` | `30`, `30`, `yes`, `1001` | the kit's defaults, written out (`kit/README.md:156-162`) so the person reads them |
| `prune-keep-green` | `checks`, each stack job in flag order, `kit` | every declared job (`example:ci/manifest:18`) |
| `image-path` | `ci/image`, then each tool's pin file in flag order | the Dockerfile's build context is exactly these paths (`kit/README.md:161`); a pin file is in it because the Dockerfile cross-checks it (§3.3) |
| `job` | `checks always-on`, each stack job `local`, `kit local` | flags |
| `paths <job>` | its directory, then every `image-path` entry outside `ci` that its directory does not already cover, then `ci` | rules 3 and 9 (`kit/bin/check-manifest:150-158,259-264`): `ci` covers the kit, the manifest, the adapter, the image and the signers file; the other jobs' pin files are image inputs. This reproduces `paths api api web/.nvmrc ci` (`example:ci/manifest:37`) |
| `paths checks`, `paths kit` | `.`, `ci` | rules 2 and 3 (`kit/bin/check-manifest:134,162-165`) |
| `pin <job> <tool>` | `go`, `node` or `python` | one per stack job |
| `count <job> ...` | Go `go_packages go_tests`; Node `vitest_files vitest_tests`; Python `pytest_tests`; a stub `tests`; the kit job `kit_cases` | the arm's counts line (§3.2); the kit job's line is fixed (`kit/README.md:172`) |
| `install <job> npm-ci <dir>` | for a Node job whose directory holds `package-lock.json` | the one install kind (`kit/README.md:173`); without a lockfile the arm is a stub (§3.2) |
| `service` | `service <job> postgres image postgres:<major>@sha256:<digest> url-env <PROJECT>_TEST_PG_URL flag fsync=off flag synchronous_commit=off` | `--postgres`; the digest from §4.2; the line is the example's (`example:ci/manifest:47`) |
| `cache` | Go `gomod gobuild`; Node `npm`; Python none | the three kinds (`kit/README.md:177`); Python has no cache kind in `manifest v1`, so its runs install cold (§3.2) |
| `pin-source` | `go toolchain-line <dir>/go.mod`; `node first-line <dir>/.nvmrc` (or `.node-version`); `python first-line <dir>/.python-version` | the three kinds (`kit/README.md:185`); one line per tool, from the first directory of that stack (§2.2) |
| `probe` | `go first-line go env GOVERSION`; `node first-line node --version`; `python version-flag python3` | the example's two (`example:ci/manifest:53-54`); for Python, `version-flag` reads `Python 3.12.7` as `3.12.7` (`kit/README.md:186`) where `first-line` would read the whole line |
| `escaping-tree <job> <dir>` | each stack job | the fence's trees (`kit/README.md:179`) |
| `escaping-scan` | Go `go <dir>`; Node `ts <dir>/src` when that directory exists, else `ts <dir>`; Python none | the two scanners (`kit/README.md:188`); for Python the generator writes a comment line saying crossings are declared by hand, and prints the same |

The comment lines of the template say what each block is for, in the example's words (`example:ci/manifest:1-4,25-26,32-35,56-57`) with the project's job names substituted.

### 3.2 `ci/ci-job.sh`

Assembled from `kit/templates/adapter/`: a head, one arm per stack job, and a tail. The head and tail are the example's (`example:ci/ci-job.sh:1-56` and `:95-99`): `set -uo pipefail`, `readonly PROJECT=<project>`, the `err`, `refuse`, `failed`, `need` and `step` helpers, the scratch directory, and the `case` that dispatches each job to its arm and refuses any other name with exit 2 (the contract of `kit/README.md:303-320`). File mode 0755, as the example's.

| Arm | What it runs | Counts line | Refusals (exit 2) |
|---|---|---|---|
| Go | `gofmt -l`, `go vet ./...`, `go test -count=1 -v ./...` to a file read after its status (`example:ci/ci-job.sh:58-76`) | `go_packages`, `go_tests`, counted from the saved output | the toolchain absent or off the directory's own `toolchain` line; the directory missing; with `--postgres`, the URL variable unset |
| Node with vitest (`vitest` in `package.json`'s dependencies or devDependencies) | `tsc --noEmit` when `tsconfig.json` exists, then `vitest run` with the JSON reporter to a file (`example:ci/ci-job.sh:78-93`) | `vitest_files`, `vitest_tests`, read with `jq`, which the base carries (`kit/image/base.Dockerfile:60`) | node absent or off the pin file; `node_modules/.bin/vitest` absent (the install did not run); with `--postgres`, the URL variable unset |
| Python with pytest (`pytest` named in a requirements file carrying `--hash=` lines) | a venv in the scratch directory, `pip install --require-hashes -r <file>`, then `python -m pytest --junitxml=<file>` | `pytest_tests`: the tests the JUnit file reports minus failures, errors and skips, read by the venv's Python | python absent or off `.python-version`; with `--postgres`, the URL variable unset |
| Stub (any stack whose runner the generator does not recognise: Node without vitest or without a lockfile, Python without a hashed requirements file) | nothing | none | always: `The <job> arm is a stub kit-init wrote; put its checks in ci/ci-job.sh, so <job> did not run.` |

**Why a stub refuses rather than passes.** An arm that exited 0 with no checks would let a record vouch for a tree nothing tested. Exit 2 is the contract's "refused before any check" (`kit/README.md:308`), so `make check` stops red on the stub and the person sees the one sentence that says what to do. The stub's `count <job> tests` line is a placeholder the person renames when they write the arm.

**Why the Python install sits in the arm.** `manifest v1` has one install kind, `npm-ci` (`kit/bin/check-manifest:404-405`), and an ordinary job's container runs on the bridge network (`kit/bin/container:644`), so the arm can install. Requiring hashes keeps the verdict's inputs pinned; a requirements file without hashes gets a stub, and the closing steps say how to make one. A `pip` install kind and a `pip` cache kind would be a later MINOR release of the manifest (`kit/README.md:299`), not part of this one.

### 3.3 `ci/image/Dockerfile`

Assembled from `kit/templates/image/`. For the Go and Node shape it is the example's file (`example:ci/image/Dockerfile:14-58`); its parts:

1. `ARG CIKIT_BASE` then `FROM ${CIKIT_BASE}`, the one base `FROM` rule 9 takes (`kit/bin/check-manifest:312-315,347`).
2. Per tool, `ARG <TOOL>_VERSION=<pin>`, `ARG <TOOL>_SHA256_AMD64=<hex>`, `ARG <TOOL>_SHA256_ARM64=<hex>`; for Python also `ARG PBS_TAG=<release>` (§4.1).
3. `USER root`, `COPY <each pin-source file> /pins/`, `SHELL ["/bin/bash", "-o", "pipefail", "-c"]`.
4. One `RUN` that, for each tool: cross-checks the pin file against the `ARG` and stops the build with the sentence the example uses when they disagree (`example:ci/image/Dockerfile:28-31`); picks the checksum for `dpkg --print-architecture` and stops on any other architecture (`:32-37`); downloads the release tarball from its official location, verifies it with `sha256sum -c` (`:38-44`); extracts it under `/usr/local`; then creates the cache directories of the declared cache kinds as the job user (`:46`): `gomod` gives `/home/ci/go/pkg/mod` and its parents, `gobuild` `/home/ci/.cache/go-build`, `npm` `/home/ci/.npm` (`kit/README.md:177`).
5. One `ENV`: `PATH` with each tool's `bin` first, then `/usr/local/bin:/usr/bin:/bin`; for Go `GOTOOLCHAIN=local`, `CGO_ENABLED=0`, `GOPATH=/home/ci/go`, `GOCACHE=/home/ci/.cache/go-build`; for Node `npm_config_cache=/home/ci/.npm`, `npm_config_update_notifier=false`; for Python `PIP_DISABLE_PIP_VERSION_CHECK=1`, `PYTHONDONTWRITEBYTECODE=1` (`example:ci/image/Dockerfile:50-56`).
6. `USER ci`, `WORKDIR /home/ci`.

The sources, per tool:

| Tool | Tarball | Extracted to |
|---|---|---|
| Go | `https://go.dev/dl/go<V>.linux-<amd64\|arm64>.tar.gz` | `/usr/local/go` |
| Node | `https://nodejs.org/dist/v<V>/node-v<V>-linux-<x64\|arm64>.tar.xz` | `/usr/local/node`, the archive's top directory and its file ownership dropped (`example:ci/image/Dockerfile:44`) |
| Python | `https://github.com/astral-sh/python-build-standalone/releases/download/<PBS_TAG>/cpython-<V>+<PBS_TAG>-<x86_64\|aarch64>-unknown-linux-gnu-install_only_stripped.tar.gz` | `/usr/local/python` |

**Why both architectures every time.** The image is built on each machine for its own architecture and never pulled (`kit/README.md:16`; `example:ci/image/Dockerfile:6-7`): a Mac builds `arm64`, the runner `amd64`, so a file with one checksum works on one side only.

**Why the cross-check.** The pin lives in the file the tool reads (`go.mod`, `.nvmrc`, `.python-version`) and the Dockerfile names the same version with its checksum; the build stops when they disagree, so a pin bump is one reviewed edit of both files together (`example:ci/image/Dockerfile:9-12`).

**A shape with no stack** (only `--project`) gets steps 1 and 6 alone: a valid image file, the kit job its only real job, and a closing line saying so.

**A stub stack** gets no image block: the person adds the toolchain with the arm.

### 3.4 `.github/workflows/ci.yml`

Assembled from `kit/templates/workflow/`, which reproduces the example's workflow (`example:.github/workflows/ci.yml:13-259`):

- the triggers, the dispatch inputs `force`, `task` and `delete` and the workflow-level `permissions: contents: read` (`:15-40`), with the schedule set to `--cron`;
- the always-on job `checks` (`:43-102`): the draft refusal, the checkout, the plan step with its six variables, the manifest lint, the two fences and the record step, and its `outputs` holding `sha`, `ref_group`, `main`, then one `<job>` and one `<job>_key` per declared job in manifest order;
- one job per stack job in W-7's exact shape (`:104-151`): the checkout at the always-on job's `sha`, the cache-runtime action, the cache steps of its `cache` line (`gomod` keyed on `hashFiles('<dir>/go.sum')`, `gobuild` on the date stamp, `npm` on `hashFiles('<dir>/package-lock.json')`, as `kit/README.md:400-406` gives them; `go.mod` when `go.sum` is absent), the image step, the verdict step and the record step;
- the kit job (`:194-228`), its image step naming itself so only the base is built;
- the `prune` job (`:233-259`), on main only, dry on the schedule.

`timeout-minutes` are the example's (10 for `checks` and `prune`, 20 for a stack job, 30 for the kit job), with a comment saying to set each by measuring the job (DESIGN §11).

Every `uses:` names the action by the commit sha the example pins (`kit/README.md:392-398`; W-11). The template is the one place in the kit those three shas live outside the documents, and moving them is a release of the kit (§9.1).

**Why from a template the fence already holds green.** W-1 to W-12 (`kit/README.md:77-111`) are exact: the plan's six variables, the concurrency groups, the record step's shape. A generated workflow satisfies them by construction only if the template is itself proven against `check-workflow` for every shape the generator can produce, which §7.3 requires.

### 3.5 `.githooks/pre-push`

Copied byte for byte from `ci/kit/hooks/pre-push.shim`, mode 0755 (`kit/README.md:113-121`; the example's blob equals the kit's, both `9d9e0553`). It is not a template, so the shim has one source; the lint's rule 8 holds the two equal (`kit/bin/check-manifest:249-256`).

### 3.6 `Makefile`

From `kit/templates/Makefile.check`, the example's target (`example:Makefile:1-9`):

```make
JOBS ?= <each stack job> kit

.PHONY: check
check:
	bash ci/kit/bin/local-ci run --commit HEAD $(JOBS)
```

When a `Makefile` exists, the block is appended after one blank line; a `Makefile` that already defines `check:` is refused (§2.3), because two `check` targets would be a person's merge to make, not the generator's.

### 3.7 `.gitignore`

Nothing for the kit itself: the kit writes nothing into a working copy, its state lives under the git common directory (`kit/README.md:123`), so there is no kit line to add. Per stack, one line appended when absent: `node_modules/` for a Node job, `__pycache__/` for a Python job. Why these two: each holds files a test run or an install writes into the working copy, and a stray commit of either would enter the job's archive and its key.

### 3.8 `ci/allowed-signers`

Written as one comment line, from `kit/templates/allowed-signers`:

```
# The keys whose local records CI trusts, one line each: <principal> namespaces="git" <key type> <key>. Each is added in a pull request of its own (DESIGN.md 6.6, 9).
```

**Why the landing commits no signer, even with `--signer`.** DESIGN §9 step 1 commits the signers file empty or holding only a comment, and step 4 adds the first signer in its own pull request with the fingerprint confirmed out of band (DESIGN §6.6). The signer line is the one trust decision of adoption; putting it inside a landing of several hundred generated lines hides it from review. So `--signer` computes the line and prints it with `ssh-keygen -lf`'s fingerprint among the closing steps (§6), and keeps it in `<git common dir>/<project>-ci/init/allowed-signers.next` for the person to copy into the signers pull request. This is a deliberate narrowing of the obvious reading of the flag; §12 puts it as a question with this answer as the default.

`--signer auto` reads `user.email` as the principal (DESIGN §6.6) and the public key from `user.signingkey`: a `key::` literal as given, a path to a `.pub` file as read, a path to a private key by its `.pub` beside it. A key held only behind `gpg.ssh.defaultKeyCommand` is refused, because the generator cannot read it without running that command.

### 3.9 `AGENTS.md` and `CLAUDE.md`

Agents that work in an adopter's repository (Codex reads `AGENTS.md`, Claude Code reads `CLAUDE.md`) need the same few facts a new contributor needs, or they push with `--no-verify`, edit `ci/kit/`, or widen a key to make a red go away. So `kit-init` writes one short section into each file: created when absent, appended after one blank line when present, refused when the file already holds the section's begin marker (§2.3). `--no-agent-files` skips both; `--agent-files-only` writes only these two, for a project that adopted without the generator.

**The text is one template**, `kit/templates/agents/section.md`, and it is the only copy. The generator substitutes its tokens (§7.2); the kit's later guide for adopting with an agent (`docs/adopt-with-an-agent.md`) points an agent at `ci/kit/templates/agents/section.md` in the vendored tree, or `kit/templates/agents/section.md` in the kit repository, and holds no copy of its lines; the kit README cites it the same way. A guard in the kit repository's own workflow fails when either document repeats a body line of the template (AC-KI-27). Why: two copies of an instruction drift, and an agent that reads the stale one does the wrong thing confidently.

The template, exactly:

```markdown
<!-- quokka-ci-kit section begin: written by kit-init from ci/kit/templates/agents/section.md -->
## quokka-ci-kit

CI in this repository uses quokka-ci-kit, vendored at `ci/kit/`. How it works here:

- `ci/manifest` lists each CI job and the paths its key is a hash of. A job is skipped when a trusted record of its key exists, and runs otherwise.
- `@@ADAPTER@@` is each job's one verdict script, with one arm per job. CI and `@@CHECK@@` run the same script in the same job image.
- Run `@@CHECK@@` before you push. It tests the HEAD commit, not uncommitted changes, so commit first. It runs @@JOBS@@.
- The pre-push hook signs a record of each job that passed and pushes it beside the branch. Each clone needs `git config core.hooksPath .githooks` once.
- Never edit anything under `ci/kit/`. Update the kit with `bash ci/kit/bin/kit-update <tag>` from a checkout of @@MAIN@@, in a pull request of its own.
- To add a job, add its lines to `ci/manifest`, its arm to `@@ADAPTER@@` and its job to `@@WORKFLOW@@`. Commit, then run `bash ci/kit/bin/check-manifest`, `python3 ci/kit/bin/check-workflow`, `python3 ci/kit/bin/check-escaping-reads` and `@@CHECK@@`, then `bash ci/kit/bin/check-adapter <job>` once it passes.
- Never push with `--no-verify`. The only exception was the pull request that added the kit.
- Never put a secret in `ci/manifest` or `@@ADAPTER@@`. Both are committed, and the manifest refuses names that look like secrets.
- Change `@@SIGNERS@@` or `ci/KIT-SIGNERS` only when a person asks you to. Each is a trust file and changes in a reviewed pull request of its own.
- A red `@@CHECK@@` is a failing job. Fix the cause; never narrow a job's paths, change a pin or edit a count to make it pass.
<!-- quokka-ci-kit section end -->
```

The markers are HTML comments, so they do not render. They let a later release replace the section mechanically; that refresh is not built in this version (§1).

### 3.10 Untouched

`ci/kit/`, `ci/KIT-VERSION` and `ci/KIT-SIGNERS` are the vendoring's (`kit/README.md:359-375`) and the generator reads them only. Any other workflow under `.github/workflows/` is left alone; the fence reads every workflow for W-1's trigger ban (`kit/README.md:79`), so an existing `pull_request_target` workflow shows up in the lints of §5, as it would by hand.

## 4. Checksums and digests

### 4.1 The lookups

The generator's only network acts, each over HTTPS by Python's `urllib`, or through the Docker client for the digest:

| Value | Source | Why this source |
|---|---|---|
| Go tarball sha256, both architectures | `https://go.dev/dl/?mode=json&include=all`, the `sha256` of the two `linux-amd64` and `linux-arm64` archive entries of the version | the Go project's own published checksums |
| Node tarball sha256, both architectures | `https://nodejs.org/dist/v<V>/SHASUMS256.txt`, the `linux-x64.tar.xz` and `linux-arm64.tar.xz` lines | the Node project's own checksum file |
| Python tarball sha256 and its release tag | `https://api.github.com/repos/astral-sh/python-build-standalone/releases` (unauthenticated, the most recent page) to find the newest release carrying `cpython-<V>+<tag>-...-install_only_stripped.tar.gz` for both architectures, then that release's `SHA256SUMS` | CPython publishes no Linux binary; this project's builds are the relocatable ones the Python tool ecosystem installs, with a checksum file per release. §12 asks whether the kit accepts it |
| Postgres image digest | `docker buildx imagetools inspect postgres:<major>`, the index digest on its `Digest:` line | the multi-architecture index, so a Mac and the runner pull the same image name by one digest, as the base does (`kit/image/base.Dockerfile:10-12`) |

Nothing is computed by downloading a tarball and hashing it: a hash of what the generator happened to receive proves nothing about what the project released.

The release tarballs themselves are fetched later by the image build, and verified there against these checksums (§3.3).

### 4.2 What it prints

Each value fetched is printed with its source, one line each, so the person reviewing the landing pull request can re-check any of them: `go1.27.3 linux-amd64 65efbc80... from https://go.dev/dl/?mode=json&include=all`.

### 4.3 The offline path

`--checksums FILE` replaces every lookup. The file holds lines in the shape the official checksum files use, `<64 hex>  <name>`, where `<name>` is the release file name (`go1.27.3.linux-arm64.tar.gz`, `node-v24.22.1-linux-x64.tar.xz`, `cpython-3.12.7+20241016-aarch64-unknown-linux-gnu-install_only_stripped.tar.gz`, the Python release tag read from that name) or `postgres:<major>` for the index digest. Lines it does not need are ignored. A value the file lacks is refused with its URL (§2.3); there is no fall back to the network once the flag is given, so the offline run is fully offline. The kit's own tests use this path (§7.3).

## 5. The commit and the lints

### 5.1 Before writing

The rendered manifest goes to a scratch file and through `bash ci/kit/bin/check-manifest --file <scratch>`: the parser and the rules that need no tree (`kit/bin/check-manifest:8-14,451`). Any finding refuses the run (§2.3).

### 5.2 The commit

Unless `--no-commit`, the generator stages exactly the files it wrote or appended to, plus `ci/kit`, `ci/KIT-VERSION` and `ci/KIT-SIGNERS` when the vendoring left them uncommitted, and makes one commit of those paths alone (`git commit -- <paths>`), with the message `ci: adopt quokka-ci-kit <tag>`, the tag read from `ci/KIT-VERSION`. Git's own configuration decides whether the commit is signed. Why commit: `make check`, the lint and the fences all read the commit, not the working copy (`kit/README.md:282,623`), so an uncommitted adoption cannot be checked; and the commit leaves the person's other staged or unstaged work out of it.

### 5.3 After writing

Run in order, each one's exit code printed as it stands and never through a pipe:

1. `bash ci/kit/bin/check-manifest` at HEAD: all twelve rules, including rule 7's vendored tree and rule 8's shim (`kit/bin/check-manifest:234-256`). With `--no-commit`, `--file ci/manifest` and its own sentence for the rules that did not run.
2. `python3 ci/kit/bin/check-workflow` (with `--manifest-file ci/manifest` under `--no-commit`; `kit/bin/check-workflow:65-69`). When `import yaml` fails, the generator prints `NOT RUN check-workflow: this python3 has no PyYAML; CI's always-on job runs it.` rather than a pass (`kit/bin/check-workflow:85-89`).
3. `python3 ci/kit/bin/check-escaping-reads` (likewise `--manifest-file`).
4. `bash ci/kit/bin/check-prereqs` (`kit/README.md:15`), so a missing Docker or an old git shows before `make check` is tried.

`check-adapter` is not run: its leg (b) reads the log of a real passing run (`kit/bin/check-adapter:11-16`), which exists only after `make check`. It is a closing step (§6).

## 6. What it prints at the end

After the lints, the human steps in order, one line each, numbered, with the project's values filled in. The order is the landing's (DESIGN §9; `kit/README.md:618-631`):

1. `make check` (the first run builds the base and job images and takes several minutes).
2. `bash ci/kit/bin/check-adapter <job>` for each stack job, after `make check` passes.
3. Fill in each stub arm and its image block, then run steps 1 and 2 again (printed only when a stub was written).
4. Copy the residual paragraph of the kit's DESIGN.md §3.4 into your own documentation before you trust a local record.
5. Protect main: pull requests only, signed commits required, no force pushes.
6. Confirm a skipped required check reports success, with a scratch pull request (DESIGN §4 item 7).
7. Require these checks on main: `checks`, each stack job, `kit`.
8. On each workstation: `git config core.hooksPath .githooks`, `git config gpg.format ssh`, `git config user.signingkey <your public key>`.
9. Merge `origin/main` into this branch, then push it with `git push --no-verify -u origin <branch>`; this once only, so CI runs every job and writes the first green records.
10. Open the pull request with the sentence that its local records are refused until a signer is on main.
11. After it merges, add your signer in a pull request of its own: `<the line>`, fingerprint `<SHA256:...>` (from `--signer`; without it, the command that makes the line from your public key).
12. Read the summary of the first scheduled run on its day (`--cron`'s day and time, UTC).

Then one line on what was declared by hand: for each Python job, that its escaping reads are a census by hand (`kit/README.md:188`).

## 7. The templates

### 7.1 Where they live

`kit/templates/` inside the vendored tree, so every template is part of the `kit/` tree a release signs and `kit-update` verifies (`kit/README.md:347-357`), and `ci/kit/templates/` in an adopter. One file per output file and per stack part:

```
kit/templates/
  manifest/head, manifest/go, manifest/node, manifest/python, manifest/postgres, manifest/tail
  adapter/head.sh, adapter/go.sh, adapter/node-vitest.sh, adapter/python-pytest.sh, adapter/stub.sh, adapter/tail.sh
  image/head.Dockerfile, image/go.Dockerfile, image/node.Dockerfile, image/python.Dockerfile, image/tail.Dockerfile
  workflow/head.yml, workflow/local-job.yml, workflow/cache-gomod.yml, workflow/cache-gobuild.yml,
    workflow/cache-npm.yml, workflow/kit-job.yml, workflow/prune.yml
  Makefile.check
  allowed-signers
  agents/section.md
```

The shim is not here: it is copied from `kit/hooks/pre-push.shim` (§3.5).

### 7.2 Substitution

A closed set of `@@NAME@@` tokens (`@@PROJECT@@`, `@@PROJECT_UPPER@@`, `@@JOB@@`, `@@DIR@@`, `@@JOBS@@`, `@@CRON@@`, `@@MAIN@@`, `@@ADAPTER@@`, `@@WORKFLOW@@`, `@@CHECK@@`, `@@SIGNERS@@`, and per tool `@@VERSION@@`, `@@SHA256_AMD64@@`, `@@SHA256_ARM64@@`), replaced by plain string substitution; a template part that repeats (an output line per job, a step per cache) is a separate file rendered once per item. No template language, no `eval`, no shell: the generator is standard-library Python and a token left unreplaced after rendering is a defect it refuses on (exit 2). Why so plain: a template engine would be a dependency on every workstation, and logic inside templates is code the lints cannot see.

### 7.3 How they stay true

- **Every supported shape is generated and linted in the kit's own tests.** A new family, K-init, in `kit/tests/run` (no Docker daemon, offline through `--checksums` and the existing `docker` shim, `kit/tests/fixture/shims/docker`) generates each shape of AC-KI-30 into a fixture repository with a vendored `ci/kit` copy and holds it to: the generator's exit 0; `check-manifest` at HEAD (rules 1 to 12); `check-workflow`; `check-escaping-reads`; and the adapter's unknown-job leg run directly (`bash ci/ci-job.sh no-such-job` exits 2 with no counts line, `check-adapter`'s leg (a), `kit/bin/check-adapter:8-10`), because `check-adapter` itself needs the job image.
- **One shape end to end.** `kit/tests/image` gains a family that generates the Go, Node and Postgres shape, builds its job image against the real checksums of a fixed recorded version, runs `local-ci run` for each job and then `check-adapter` for each stack job.
- **The example as a golden output.** The fixture directory holds the example's six files for the `rottnest` shape; K-init compares its output with them, comment lines set aside, so a template that drifts from the reference adopter turns red.
- **Plants.** `kit/tests/plants` gains planted defects for the rules a single deleted line could remove: the existing-file refusal, the Dockerfile's pin cross-check, the shim copied from anywhere but the kit's file, an unreplaced token passing.

## 8. The host rule

**Ruling: `kit-init` is a Python 3 script, standard library only, run once by hand; it is not a host bash entry point, and `check-host`'s file list and allow-list do not change.**

Why not bash 3.2 under the allow-list. The host bash scripts may call only the commands of `kit/bin/check-host:38-39` (git, docker, mkdir, mv, rm, mktemp, cat, date, uname, sleep, kill, tail, head, tr, wc, shasum, sha256sum, ssh-keygen), and "no host tool parses any output" (`kit/README.md:71`). The generator must fetch over HTTPS (`curl` is not on the list), read Go's JSON release index, read `package.json`, `go.mod` and the requirements files of each directory, and render templates with repeated parts. Doing that in bash 3.2 would either widen the allow-list for a one-time tool, which weakens a fence every everyday host script lives under, or parse JSON with `[[ =~ ]]`, which is the class of fragile host parsing the rule exists to keep out.

Why Python is acceptable here. The kit already ships two Python tools a person runs on a workstation by hand, the workflow fence and the escaping-read fence (`kit/README.md:25-26,73`), and they sit outside `check-host` by its own file list (`kit/bin/check-host:40-41`). `kit-init` joins them on the same terms. macOS provides `python3` through Apple's command line tools (3.9 at the oldest supported release), so the floor is Python 3.9 and the standard library: no PyYAML, since the generator writes YAML from templates and never parses it.

What this means for `check-host`: no code change. `HOST_FILES` (`kit/bin/check-host:40-41`) keeps its nine entries; `kit-init` is not added, because it is not bash. Instead the generator carries two rules of its own, each proven in K-init:

1. **Its subprocesses are a closed set**, called by argument vector, never through a shell: `git`, `docker` (the digest lookup only), `ssh-keygen` (the fingerprint only), and `bash` running the kit's own `check-manifest` and `check-prereqs`, plus `python3` (`sys.executable`) running the two fences. A test reads the script's syntax tree and fails on any other first argument or on `shell=True`.
2. **Its syntax parses as Python 3.9**: a test runs `ast.parse(source, feature_version=(3, 9))`, so a newer construct cannot slip in on a contributor's newer Python.

The README's §2.2 gains one sentence naming `kit-init` beside the two fences, and the prerequisite list of `kit/README.md:618` gains "`python3` 3.9 or later, to run `kit-init` once".

## 9. The release

### 9.1 The version

**A MINOR release.** It adds a command and templates; the manifest format stays `manifest v1`, the adapter contract, the key formula and the plan's sentences are unchanged, and nothing an adopter wrote needs to change (`kit/README.md:345`; `CONTRIBUTING.md:81-85`). Today's tag is `v1.0.0-rc.1` and no final `v1.0.0` exists. A final is tagged on the same `kit/` tree as the last candidate once an adopter ran it green (`CONTRIBUTING.md:79`), which the example has done. So the order is: `v1.0.0` on rc.1's tree, unchanged; then this work as `v1.1.0-rc.1`; then `v1.1.0` on the same tree once an adoption through `kit-init` has run green end to end. Putting `kit-init` into a `v1.0.0-rc.2` instead would make every candidate adopter re-test the 1.0 trust code for a change that does not touch it.

Later moves of the workflow template's action shas, or of the default Postgres major, are PATCH releases: they change generated output for new adopters and no interface.

### 9.2 The CHANGELOG entry

```markdown
## v1.1.0-rc.1 (unreleased)

- New: `kit/bin/kit-init`, the generator of an adopter's files (docs/kit-init.md):
  the manifest, the adapter, the job image's Dockerfile, the workflow, the hook
  shim, the Makefile's check target, the signers file and an agent section in
  AGENTS.md and CLAUDE.md, for Go, Node with vitest and Python with pytest,
  with a refusing stub arm for anything else. Python 3.9 or later, standard
  library only; run once, by hand.
- New: `kit/templates/`, the files it renders, signed with the release.
- Unchanged: the manifest format (`manifest v1`), the adapter contract, the key
  formula, the plan's sentences, `check-host`'s file list and allow-list.
- The proofs: the K-init family in `kit/tests/run` (every supported shape
  generated offline and held to the lint and both fences), one shape end to end
  in `kit/tests/image`, and planted defects in `kit/tests/plants`.
- Moves keys: yes (every adapter-run key moves with `ci/kit`).
- The `kit/` tree this tag names: `<written by the release pull request>`.
- Adopter's actions: none. A project that adopted by hand may add the agent
  section with `python3 ci/kit/bin/kit-init --agent-files-only`.
```

### 9.3 Copies of the kit maintained elsewhere

A fork of the kit maintained separately does not receive this automatically, and nothing here depends on it doing so. A later port would need: the templates with the fork's own constants substituted (its image label, its default `kit-update` URL, its release tag message, its name in the agent section's heading and markers; `kit/README.md:48`); the fork's own example as the golden output; the K-init family and plants run against the fork's tree; and its own MINOR release. The generator's code carries no kit name outside one constant block for exactly this reason.

## 10. Acceptance criteria

Each is checkable by reading a file or running a command.

**The command**

- **AC-KI-1** `kit/bin/kit-init` exists, mode 0755, first line `#!/usr/bin/env python3`, and imports only the standard library.
- **AC-KI-2** Its syntax parses under `ast.parse(..., feature_version=(3, 9))`; on a Python below 3.9 it exits 2 with its sentence.
- **AC-KI-3** Every subprocess call names `git`, `docker`, `ssh-keygen`, `bash` or `sys.executable` as its first argument, by list, and none passes `shell=True` (a syntax-tree test).
- **AC-KI-4** Every refusal of §2.3 has a K-init case that triggers it, sees exit 2 and its sentence, and sees the working tree, the index and HEAD unchanged.
- **AC-KI-5** A run refused after rendering (a lint finding on the rendered manifest) writes nothing; a write failure removes only files the run created.
- **AC-KI-6** With `--checksums`, no network is used: the K-init family runs with no network and the `curl` shim unreachable, and passes.
- **AC-KI-7** A value missing from `--checksums` is refused with its URL; there is no fallback to the network.
- **AC-KI-8** Nothing is written outside the working copy and `<git common dir>/<project>-ci/init/`; at exit that directory holds at most `allowed-signers.next`.

**What it writes**

- **AC-KI-10** The manifest it writes passes `check-manifest` at HEAD, rules 1 to 12, for every shape of AC-KI-30.
- **AC-KI-11** `paths` for each stack job is its directory, then each other image input it does not cover, then `ci`; `image-path` is `ci/image` then each pin-source file in flag order.
- **AC-KI-12** The workflow passes `check-workflow` for every shape, and its schedule equals the manifest's `weekly-cron`.
- **AC-KI-13** The adapter's unknown-job arm exits 2 and prints no counts line for every shape; a stub arm exits 2 with its sentence and no counts line.
- **AC-KI-14** `.githooks/pre-push` is byte-equal to `ci/kit/hooks/pre-push.shim`, mode 0755.
- **AC-KI-15** The Dockerfile holds exactly one `FROM ${CIKIT_BASE}`, both architecture checksums per tool, the cross-check of each pin file, and the cache directories of exactly the declared cache kinds.
- **AC-KI-16** `ci/allowed-signers` in the landing commit holds only its comment line, with or without `--signer`; with `--signer`, the printed line and fingerprint equal `allowed-signers.next` and `ssh-keygen -lf` of the key.
- **AC-KI-17** An existing `Makefile`, `.gitignore`, `AGENTS.md` or `CLAUDE.md` keeps every original byte; the addition follows after one blank line.
- **AC-KI-18** The agent section in both files is the rendered `kit/templates/agents/section.md` with no `@@` token left; `--no-agent-files` writes neither; `--agent-files-only` writes only these two, reading every value from `bin/keys dump HEAD`.
- **AC-KI-19** No token `@@...@@` survives in any written file.
- **AC-KI-20** On the example's tree with its six files removed, `kit-init --project rottnest --go api --node web --postgres api --checksums <the example's values>` writes files equal to the example's with comment lines set aside.

**The commit and the lints**

- **AC-KI-21** Without `--no-commit`, HEAD is one new commit whose message is `ci: adopt quokka-ci-kit <tag>` and whose changed paths are exactly the generator's files plus any uncommitted vendoring; other staged work stays staged and uncommitted.
- **AC-KI-22** After writing it runs the four checks of §5.3 in order and prints each exit code; a missing PyYAML prints the `NOT RUN check-workflow` line, never a pass.
- **AC-KI-23** The closing steps of §6 are printed in that order, one line each, with the project's job names.

**The templates and the documents**

- **AC-KI-24** Every template lives under `kit/templates/`; the generator reads nothing else but `kit/hooks/pre-push.shim` and the adopter's own files.
- **AC-KI-25** `check-host kit/` passes unchanged, and its file list is unchanged.
- **AC-KI-26** `kit/README.md` names `kit-init` in §1's table, §2.2 and the prerequisites, and holds no copy of the agent section's body lines.
- **AC-KI-27** The kit repository's workflow fails when `docs/adopt-with-an-agent.md` or `kit/README.md` repeats a body line of `kit/templates/agents/section.md`, and passes when they cite its path; the guard is proven red once.
- **AC-KI-28** Each planted defect of §7.3 turns K-init red, and only K-init.
- **AC-KI-29** The end-to-end family in `kit/tests/image` generates the Go, Node and Postgres shape, `local-ci run` passes each job, and `check-adapter` passes each stack job.
- **AC-KI-30** K-init covers at least these shapes: Go alone; Node with vitest alone; Go, Node and Postgres (the example's); Python with pytest alone; Node without vitest (a stub); two Go directories with one toolchain; no stack at all; a stack directory of `.`.
- **AC-KI-31** The CHANGELOG entry has §9.2's shape and the release is MINOR.

## 11. Build plan

Three pieces of work, in order. Each names who builds it, the files it may touch, the criteria it meets and an estimate in agent tokens from the measured medians of builds of this size.

**1. The generator and its templates.** A build engineer. Files: `kit/bin/kit-init` (new), `kit/templates/**` (new), `kit/README.md` (§1 table row, §2.2 sentence, prerequisites, §7 gaining the generator route with steps 4 to 6 folded into it), `CHANGELOG.md` (the unreleased entry). Meets AC-KI-1 to AC-KI-3, AC-KI-5, AC-KI-7, AC-KI-8, AC-KI-10 to AC-KI-19, AC-KI-21 to AC-KI-26, AC-KI-31, each by the build's own run on the example's tree and on one scratch repository per stack. Estimate about 500k (range 400k to 600k): roughly 700 to 900 lines of Python and 600 lines of templates, three stacks, the lookups and the offline path.

**2. The proofs.** A test engineer, tests only. Files: `kit/tests/fam/init.sh` (new), its entry in `kit/tests/run`'s family list (`kit/tests/run:26`), `kit/tests/fixture/init/**` (the shapes' inputs, the golden example files, the checksums file), a family in `kit/tests/image`, plants in `kit/tests/plants`, and the kit workflow's guard for AC-KI-27 (`.github/workflows/ci.yml`, a step in `guards`). Meets AC-KI-4, AC-KI-6, AC-KI-20, AC-KI-27 to AC-KI-30, each criterion seen red once. Estimate about 220k (range 200k to 300k).

**3. The documents.** A documentation pass. Files: `docs/adopt-with-an-agent.md` (new; it cites the agent template, never copies it), the root `README.md`'s quickstart pointer to `kit-init`, `docs/kit-init.md` amended to what was built. Meets AC-KI-26 and AC-KI-27's passing half. Estimate about 100k.

Total about 820k, range 700k to 1.0M. A security read is worth asking for once the first piece lands, scoped to the generator's network lookups, its subprocess set and the signer handling.

## 12. Open questions

Each has the answer this specification takes when nobody rules otherwise.

1. **The Python toolchain's source.** CPython publishes no Linux binary tarball. Accept python-build-standalone's builds, verified against that project's per-release `SHA256SUMS` (§4.1)? Or leave Python's image block as a stub the person fills in, with only the arm and the manifest lines generated? **Default: accept python-build-standalone**, named in the generated Dockerfile's comment so the reviewer sees the source.
2. **The signer in the landing.** `--signer` could write the signer's line straight into `ci/allowed-signers`, saving the second full CI run DESIGN §9 step 4 costs. That puts the adoption's one trust decision inside a large generated pull request. **Default: never in the landing**; the line is printed and kept for its own pull request (§3.8).
3. **The release order.** Tag `v1.0.0` on rc.1's tree first and ship this as `v1.1.0-rc.1`, or fold it into a `v1.0.0-rc.2`? **Default: `v1.0.0` first, then `v1.1.0-rc.1`** (§9.1).
4. **The default weekly minute.** Every generated project would run at `20 6 * * 1` unless it passes `--cron`. A minute derived from the project name would spread the load on the shared runners and keep each project stable. **Default: the example's `20 6 * * 1`**, because a value the person can read in the example is easier to reason about than a derived one; revisit if adopters report queueing.
5. **The agent files by default.** Write both `AGENTS.md` and `CLAUDE.md` when neither exists, or only append to those already present? **Default: write both**, because a repository without them is the one an agent most needs told, and `--no-agent-files` opts out.

One inconsistency found while writing this, for the documentation pass: the cache-runtime action's trailing comment reads `# v3` in `kit/README.md:397` and `# v4.0.0` in `example:.github/workflows/ci.yml:123,172,213` for the same sha `04d248b8`. The template follows the example's file, since the repository's file is the one that runs (`kit/README.md:278`); the README line should be corrected to whichever release that sha is.
