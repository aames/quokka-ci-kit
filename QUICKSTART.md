# From the template to green in 30 minutes

You have a Go or Node project (or both) on GitHub Actions, and you want CI that skips a job whose exact content already passed, in CI or signed on your own machine. This page starts from the example repository, which is a GitHub template, and ends with a pull request whose tested jobs were skipped on your own signed `make check`. Each step has its command, a time, and a "done when" line. The why behind every step is in [README.md](README.md) and [DESIGN.md](DESIGN.md); this page is only the how.

**You need:** Docker running, git 2.34 or later, an SSH key (`~/.ssh/id_ed25519.pub` below; use your own key's path where it differs), and admin rights on the new repository. The [GitHub CLI](https://cli.github.com/) (`gh`) saves two trips to the web page but is optional. Go, Node and Postgres come in the job image, not from your machine.

## 1. Make your repository from the template (2 minutes)

On [quokka-ci-kit-example](https://github.com/aames/quokka-ci-kit-example), click **Use this template**, then **Create a new repository**. Then:

```sh
git clone git@github.com:<you>/<your-repo>.git
cd <your-repo>
git switch -c adopt-kit
```

The template copies the files of `main` and nothing else: no records, no tags. GitHub may start a CI run on the new repository's first commit; it tests the example's code and nothing later depends on it.

*Done when* `ls ci/kit/bin/plan ci/manifest` lists both files and you are on `adopt-kit`.

## 2. Check that the vendored kit is the real release (2 minutes)

`ci/kit/` arrived with the template. Confirm it is release `v1.0.0-rc.1`, signed by the key published in [README.md](README.md#releases-and-the-release-signer):

```sh
cut -d' ' -f3- ci/KIT-SIGNERS | ssh-keygen -lf -
diff <(bash ci/kit/bin/kit-update --verify-only v1.0.0-rc.1 | head -n 4) <(head -n 4 ci/KIT-VERSION)
```

*Done when* the fingerprint reads `SHA256:Vwu8G5Eeo+oc6eRSJtjpGFVVMmUfMNrYpVMR0CfDjX8` and `diff` prints nothing. If either differs, stop. Keep `ci/KIT-SIGNERS` exactly as it is: it names who may sign a kit release, not who may vouch for your tests.

## 3. Sign your commits, and name yourself as the signer (3 minutes)

The key that signs your commits is the key that signs your local test records, so set it up once:

```sh
git config gpg.format ssh
git config user.signingkey ~/.ssh/id_ed25519.pub
git config commit.gpgsign true
gh ssh-key add ~/.ssh/id_ed25519.pub --type signing --title "commit signing"
printf '%s namespaces="git" %s\n' "$(git config user.email)" "$(cut -d' ' -f1,2 ~/.ssh/id_ed25519.pub)" > ci/allowed-signers
```

The `gh` line tells GitHub the key signs commits (or add it under **Settings, SSH and GPG keys, New SSH key, Signing key**). The last line replaces the example author's line in `ci/allowed-signers` with yours; CI reads that file from `main` only, so it trusts you once this branch merges.

*Done when* `cut -d' ' -f3- ci/allowed-signers | ssh-keygen -lf -` prints the same fingerprint as `ssh-keygen -lf ~/.ssh/id_ed25519.pub`, and the file holds that one line.

## 4. Rename the project (3 minutes)

Pick a project name: 2 to 16 characters, lower-case letters and digits, starting with a letter, no hyphen (`sightings`, say). Find every place the example's name appears:

```sh
git grep -n -i -e rottnest -e quokka-ci-kit-example -e 'quokka sightings' -- ':!ci/kit'
```

Never edit anything under `ci/kit/`: CI holds it byte-equal to the release, and its own README mentions the example. The grep lists these sites:

| Where | What it says | Becomes |
|---|---|---|
| `ci/manifest` line 6 | `project rottnest` | `project <name>`. Every derived name (image tags, cache volumes, the state directory, the key header) follows from this line |
| `ci/manifest` line 47, `ci/ci-job.sh` lines 15 and 65 | `ROTTNEST_TEST_PG_URL` | `<NAME>_TEST_PG_URL`, the same in all three places; your database tests read it |
| `ci/ci-job.sh` line 24 | `readonly PROJECT=rottnest` | `readonly PROJECT=<name>`, because the counts line must begin `<name>-ci-job v1 counts` |
| `.github/workflows/ci.yml` lines 127, 128, 135, 136, 176, 177 | `rottnest-ci-gomod`, `-gobuild`, `-npm` | `<name>-ci-gomod` and so on; the workflow fence requires `<project>-ci-<kind>` in each cache `path:` |
| `.github/workflows/ci.yml` line 1 | a comment | your project's name |
| `README.md` | the example's description | your README (step 5 says what to keep) |
| `api/`, `web/` | the example's service, its module path `github.com/aames/quokka-ci-kit-example/api`, and `rottnest-web` | your own code, in step 5 |

The `Makefile` and `ci/image/Dockerfile` carry no project name; they name jobs and paths, which step 5 handles.

*Done when* the grep prints only lines under `api/`, `web/` and `README.md`, which step 5 replaces.

## 5. Make it your shape (5 minutes)

The example has three local jobs: `api` (Go, with a throwaway Postgres), `web` (Node, TypeScript and vitest) and `kit` (the kit's own tests; always keep it). Bring your code in and remove the job you do not need.

**Both Go and Node.** Replace the contents of `api/` with your Go module and `web/` with your Node package. Your `go.mod` needs the line `toolchain go1.27.3` and your `web/.nvmrc` must read `24.22.1`, the versions `ci/image/Dockerfile` installs; for other versions see "What still takes thought" below. If your directories have other names, rename them here and in every place the second table below lists for that job.

**Go only.** Delete `web/` (`git rm -r web`) and every `web` and Node line:

| File | Remove |
|---|---|
| `ci/manifest` | `job web local`, `paths web ...`, `pin web node`, `count web ...`, `install web npm-ci web`, `cache web npm`, `pin-source node ...`, `probe node ...`, `escaping-tree web web`, `escaping-scan ts web/src`; the word `web/.nvmrc` from `image-path` and from `paths api`; the word `web` from `prune-keep-green` |
| `ci/ci-job.sh` | the function `arm_web` and the case line `web) arm_web ;;` |
| `ci/image/Dockerfile` | the three `NODE_` arguments, `web/.nvmrc` from the `COPY` line, the two `.nvmrc` check lines, `node_arch` and `node_sha` in the two `case` arms, the four Node download and unpack lines, `/tmp/node.tar.xz` from the `rm` line, `/home/ci/.npm` from the `install -d` line, `/usr/local/node/bin:` from `PATH`, and the two `npm_config_` lines with the trailing `\` on the `GOCACHE` line above them |
| `.github/workflows/ci.yml` | the whole `web:` job, and the `web` and `web_key` outputs of `checks` |
| `Makefile` | `web` from `JOBS ?= api web kit` |

**Node only.** Delete `api/` (`git rm -r api`) and every `api` and Go line, the mirror of the table above: in `ci/manifest`, the `api` lines including `service` and `cache api`, `pin-source go`, `probe go`, `escaping-tree api api`, `escaping-scan go api`, `api/go.mod` from `image-path` and `paths web`, and `api` from `prune-keep-green`; in `ci/ci-job.sh`, `arm_api` and its case line; in the Dockerfile, the three `GO_` arguments, `api/go.mod` from `COPY`, the two toolchain check lines, `go_sha`, the three Go download lines, `/tmp/go.tar.gz`, the three `/home/ci/go` directories and `/home/ci/.cache/go-build`, `/usr/local/go/bin:`, and `GOTOOLCHAIN`, `CGO_ENABLED`, `GOPATH`, `GOCACHE`; in the workflow, the `api:` job and its two outputs; in the `Makefile`, `api`. If your Node tests need Postgres, move the `service` line to `web` instead of deleting it.

**No database?** Delete the `service api ...` line and the `_TEST_PG_URL` check in `arm_api` (line 65).

**Your checks, not the example's.** Each arm in `ci/ci-job.sh` runs the example's checks (`gofmt`, `go vet`, `go test`; `tsc --noEmit`, `vitest run`) and ends with one counts line whose fields match the job's `count` line in the manifest. Change the commands to yours; keep the counts line, and keep every count above zero when tests ran.

**README.** Write your own, and copy into it, word for word, the residual paragraph from [README.md, How trust works](README.md#how-trust-works), with the two facts it names beside it: it says what a local record proves and what it does not.

*Done when* `git status` shows only your intended changes and step 4's grep prints nothing.

## 6. Commit and run the checks (5 minutes, most of it the first image build)

The checks read the commit, never your working copy, so commit first:

```sh
git add -A
git commit -m "Adopt quokka-ci-kit"
bash ci/kit/bin/check-manifest
make check
bash ci/kit/bin/check-adapter api
```

The first `make check` builds the kit's base image and your toolchain layer (a few minutes, once per image key), then runs each job in it. Run `check-adapter` once for each local job but `kit`. The two Python fences, `python3 ci/kit/bin/check-workflow` and `python3 ci/kit/bin/check-escaping-reads`, need PyYAML on your machine; if you lack it, CI runs both in `checks`.

*Done when* each command exits 0, and `git -c gpg.ssh.allowedSignersFile=ci/allowed-signers log --show-signature -1` reads `Good "git" signature for <your email>`, which proves the line in `ci/allowed-signers` is the key that signs your commits.

## 7. Land it (5 minutes, mostly waiting)

Push without the hook, so CI runs every job once and writes the first green records itself:

```sh
git push --no-verify -u origin adopt-kit
gh pr create --fill
```

In the run, `checks` plans: its summary table lists each job with its key, the record it found, the record's provenance, and the decision. Every decision is **run** this time, because no record exists for your new keys and `main`'s signers file still names the example's author. Each job that passes writes `refs/ci/green/<job>/<key>`.

*Done when* every job in the run is green.

## 8. Protect `main` (2 minutes)

In **Settings, Rules, Rulesets, New branch ruleset**, target the default branch and turn on: require a pull request before merging; require signed commits; require status checks to pass, adding `checks` and each job you kept (`api`, `web`, `kit`); block force pushes. A required job that the plan skips counts as passing, which is the point. Then merge the pull request.

*Done when* the ruleset reads back with those four rules and the pull request is merged. The merge's run on `main` shows every tested job **green** with provenance `ci`, and only `checks` ran.

## 9. Turn on the hook and push a real change (3 minutes)

On each machine you work from, once per clone:

```sh
git switch main && git pull
git config core.hooksPath .githooks
bash ci/kit/bin/check-prereqs
```

Then change something under `api/` or `web/` (a change only to docs moves no key, so nothing would run), commit it on a new branch, and push without `--no-verify`. The hook runs each job whose key moved, inside the same image CI uses, and on a pass signs and pushes a record. Look for lines like `api: signed refs/ci/local/api/<key>`. A failing job stops the push. Open the pull request.

*Done when* the pull request's plan summary shows `local: <your email>` in the provenance column beside each job you changed, and only `checks` ran.

```text
        .-"""-.          .---.
       / .===. \        ( 29:58 )
       \/ ^ ^ \/ _______/'---'
       ( \_o_/ )
  _____\/___\/________
```

Two seconds under, and the runner never woke up.

## On Mondays

Every Monday at 06:20 UTC (`weekly-cron` in `ci/manifest`, held equal to the workflow's `schedule` by the fence), CI ignores every local record and runs, on `main`'s tip, each job that CI itself has not passed at that exact content. After a week of local records that is usually every job: budget one full run a week, on purpose. The `prune` job lists records older than 30 days; a manual run with `task: weekly` and `delete` ticked deletes them. To move the day or time, change both lines together.

*Done when* the first Monday's run is green and its summary carries the sentence that the weekly run ignores every local record. Read it each week; a red there is a red `main`.

## What still takes thought

- **A different Go or Node version.** The pin lives where the tool reads it (`toolchain` in `api/go.mod`, `web/.nvmrc`) and again in `ci/image/Dockerfile` with each release archive's SHA-256 for amd64 and arm64; the build stops if the two disagree. Take Node's sums from `https://nodejs.org/dist/v<version>/SHASUMS256.txt` and Go's from `https://go.dev/dl/` (the `linux-amd64` and `linux-arm64` archives). Change both files in one commit; every local job runs again.
- **A different layout.** A directory name appears in the manifest (`paths`, `image-path`, `pin-source`, `install`, `escaping-tree`, `escaping-scan`), in the arm's `cd`, in the Dockerfile's `COPY`, and in the workflow's `hashFiles(...)` cache keys. A job's `paths` must cover everything its tests read; anything outside its own tree belongs on its `paths` line.
- **Escaping reads.** `check-escaping-reads` sees Go and TypeScript test files only. A helper your tests import that reads files at run time (migrations through a relative path, a fixture loader), or a path built rather than written, is yours to declare with an `escaping-read` line ([kit/README.md](kit/README.md) §3.4).
- **Job time bounds.** Each workflow job has a `timeout-minutes`; measure your suite's real time in CI and set about half again its worst run.

## Python, as far as the kit goes today

The kit has no Python adopter yet, so treat this as a sketch, not a proven path. What `manifest v1` offers: a pin read from a file (`pin-source python first-line .python-version`, the file holding just `3.13`), and a probe that reads the interpreter's major and minor version (`probe python python-minor`). What it lacks: an install kind other than `npm-ci`, a pip cache, and an escaping-read scanner for Python. So:

- **The image.** The kit's base is Debian 13, whose `python3` is 3.13 without `venv`. In `ci/image/Dockerfile`, as `USER root`, install `python3-venv` and fail the build unless the interpreter's minor version equals `.python-version`; list `.python-version` in `image-path` and in the job's `paths`. Another Python version means building it into the image by checksum, as the example does for Go.
- **The arm.** Your arm in `ci/ci-job.sh` makes a virtual environment in its scratch directory, installs your locked requirements (`pip install --require-hashes -r requirements.txt`), runs `pytest`, and prints `<name>-ci-job v1 counts <job> pytest_tests <n>`. The job container has network access but no pip cache, so every run downloads its packages.
- **The manifest and workflow.** `job py local`, `pin py python`, `count py pytest_tests`, no `cache` line, and a workflow job shaped like `web:` without its cache step, its outputs added to `checks`. Declare any test that reads outside its tree by hand, with `escaping-read`.

## When the template is not the right start

To adopt the kit in a repository you already have, with its history, follow [README.md](README.md#quick-start)'s pointer to the full procedure in [kit/README.md](kit/README.md) §7: the same files, written by you, and one extra pull request, because your `main` has no signers file until you add one.
