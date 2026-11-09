# Green CI in 30 minutes, or faster with kit-init

You have a project on GitHub Actions, and you want CI that skips a job when the files it depends on have already passed, either in CI or in a signed run on your own machine. This page gives two routes, each ending with a pull request whose test jobs were skipped on the strength of your own signed `make check`. **Faster: kit-init** is for a repository you already have. **The template route** (steps 1 to 9) is for a new repository made from the example. Each step has its commands, a rough time, and a "done when" line. The reasons behind the steps are in [README.md](README.md) and [DESIGN.md](DESIGN.md); this page is only the how.

## Faster: kit-init

For a repository you already have, with its tools pinned where each tool reads its version (a `toolchain` line in `go.mod`, an exact version in `.nvmrc`, an exact version in `.python-version`). `kit-init` is available from `v1.1.0`. You need everything the template route needs (see below), plus `python3` 3.9 or later. The names `sightings`, `api` and `web` are examples; use your own project's name and directories.

### A. Get the kit and check your machine (2 minutes)

```sh
git clone https://github.com/aames/quokka-ci-kit.git ../quokka-ci-kit
bash ../quokka-ci-kit/kit/bin/check-prereqs
git config gpg.format ssh
git config user.signingkey ~/.ssh/id_ed25519.pub
git config commit.gpgsign true
```

*Done when* `check-prereqs` exits 0. The last three lines set git up to sign your commits with your SSH key, which is also the key that will sign your local test records.

### B. Vendor the kit (3 minutes)

On `main`, with a clean working tree, put the release signer's line from [README.md](README.md#releases-and-the-release-signer) into `ci/KIT-SIGNERS` (create the `ci` directory if it doesn't exist), then:

```sh
bash ../quokka-ci-kit/kit/bin/kit-update v1.2.0
git switch -c adopt-kit
```

*Done when* `kit-update` has printed the fingerprint `SHA256:Vwu8G5Eeo+oc6eRSJtjpGFVVMmUfMNrYpVMR0CfDjX8` for that line and `ls ci/kit/bin/kit-init` lists the file. If the fingerprint differs, stop.

### C. Write the files (1 minute)

```sh
python3 ci/kit/bin/kit-init --project sightings --go api --node web --postgres api --signer auto
```

Give one flag for each directory whose tests a job should run: `--go` for a directory with a `go.mod`, `--node` for one with a `package.json`, `--python` for one with a `.python-version`, and `--postgres <job>` for a job that needs a throwaway database. `kit-init` writes the manifest, the test script, the job image's Dockerfile, the workflow, the hook, the `check` target and the CI section for agents, commits them as one commit on `adopt-kit`, runs the checks and prints the steps that remain. If a file it would write already exists, or a version pin is missing, it says which and writes nothing.

*Done when* `kit-init` exited 0, each check it printed reads `exit 0`, and `git status` prints nothing. Exit 1 means a check found a problem or the commit failed, and the output says which. Exit 2 means it refused and wrote nothing; its message gives the reason. Exit 3 means one of the checks didn't run, usually `check-workflow` on a machine whose `python3` has no PyYAML; run the command it printed before you push.

### D. Run the jobs (5 minutes, most of it the first image build)

```sh
make check
bash ci/kit/bin/check-adapter api
```

Run `check-adapter` once for each job `kit-init` created, except `kit`.

*Done when* each command exits 0.

### E. Land it, then add your signer (10 minutes, mostly waiting)

Follow the numbered steps `kit-init` printed, in order: copy the trust paragraph into your own documentation, protect `main`, merge `origin/main` into `adopt-kit`, push it once with `git push --no-verify -u origin adopt-kit`, open the pull request, and merge it when every job is green. Then add your signer in a pull request of its own:

```sh
git switch main && git pull && git switch -c add-signer
cat "$(git rev-parse --git-common-dir)/sightings-ci/init/allowed-signers.next" >> ci/allowed-signers
git commit -am "ci: add my signer"
git push -u origin add-signer
```

*Done when*, before that push, `git -c gpg.ssh.allowedSignersFile=ci/allowed-signers log --show-signature -1` reports `Good "git" signature for <your email>`, and the pull request is merged. From here on, every push goes through the hook, as step 9 below shows.

**Why your signer line comes second here, and first in the template route.** With `kit-init`, your `main` has no signers file yet. The first pull request commits `ci/allowed-signers` with only a comment line in it, and `kit-init` keeps your line aside in `<git common dir>/<project>-ci/init/allowed-signers.next`. That way the one trust decision in the whole adoption is reviewed on its own, in a small pull request, rather than buried in a large generated one. In the template route, your new repository's `main` already carries the example author's line, so step 3 replaces it with yours in the first pull request: `main` stops trusting someone else's key the moment that merges.

## The template route

**You need:** Docker running, git 2.34 or later, an SSH key (`~/.ssh/id_ed25519.pub` below; use your own key's path if it differs), and admin rights on the new repository. The [GitHub CLI](https://cli.github.com/) (`gh`) saves two trips to the web page but is optional. Go, Node and Postgres come in the job image, not from your machine.

## 1. Make your repository from the template (2 minutes)

On [quokka-ci-kit-example](https://github.com/aames/quokka-ci-kit-example), click **Use this template**, then **Create a new repository**. Then:

```sh
git clone git@github.com:<you>/<your-repo>.git
cd <your-repo>
git switch -c adopt-kit
```

The template copies the files on `main` and nothing else: no records, no tags. GitHub may start a CI run on the new repository's first commit; it tests the example's code, and nothing later depends on it.

*Done when* `ls ci/kit/bin/plan ci/manifest` lists both files and you are on `adopt-kit`.

## 2. Check that the vendored kit is the real release (2 minutes)

`ci/kit/` came with the template. Confirm that it is release `v1.2.0`, signed by the key published in [README.md](README.md#releases-and-the-release-signer):

```sh
cut -d' ' -f3- ci/KIT-SIGNERS | ssh-keygen -lf -
diff <(bash ci/kit/bin/kit-update --verify-only v1.2.0 | head -n 4) <(head -n 4 ci/KIT-VERSION)
```

*Done when* the fingerprint reads `SHA256:Vwu8G5Eeo+oc6eRSJtjpGFVVMmUfMNrYpVMR0CfDjX8` and `diff` prints nothing. If either differs, stop. Leave `ci/KIT-SIGNERS` exactly as it is: it says who may sign a kit release, not who may vouch for your tests.

## 3. Sign your commits, and name yourself as the signer (3 minutes)

The key that signs your commits is also the key that signs your local test records, so set it up once:

```sh
git config gpg.format ssh
git config user.signingkey ~/.ssh/id_ed25519.pub
git config commit.gpgsign true
gh ssh-key add ~/.ssh/id_ed25519.pub --type signing --title "commit signing"
printf '%s namespaces="git" %s\n' "$(git config user.email)" "$(cut -d' ' -f1,2 ~/.ssh/id_ed25519.pub)" > ci/allowed-signers
```

The `gh` line tells GitHub that the key is for signing commits (or add it yourself under **Settings, SSH and GPG keys, New SSH key, Signing key**). The last line replaces the example author's line in `ci/allowed-signers` with yours. CI only reads that file from `main`, so it will trust you once this branch merges. Your line goes into this first pull request because the template's `main` already trusts someone, the example's author, and leaving that line in place until a later pull request would keep their key trusted on your `main` in the meantime. With `kit-init` the file starts empty, so there your line comes in a pull request of its own (Faster: kit-init, step E).

*Done when* `cut -d' ' -f3- ci/allowed-signers | ssh-keygen -lf -` prints the same fingerprint as `ssh-keygen -lf ~/.ssh/id_ed25519.pub`, and the file holds that one line.

## 4. Rename the project (3 minutes)

Pick a project name: 2 to 16 characters, lower-case letters and digits, starting with a letter, no hyphen (`sightings`, say). Find every place the example's name appears:

```sh
git grep -n -i -e rottnest -e quokka-ci-kit-example -e 'quokka sightings' -- ':!ci/kit'
```

Never edit anything under `ci/kit/`. CI checks that it is byte-for-byte identical to the release, and its own README mentions the example. The grep lists these places:

| Where | What it says | Becomes |
|---|---|---|
| `ci/manifest` line 6 | `project rottnest` | `project <name>`. Every derived name (image tags, cache volumes, the state directory, the key header) follows from this line |
| `ci/manifest` line 47, `ci/ci-job.sh` lines 15 and 65 | `ROTTNEST_TEST_PG_URL` | `<NAME>_TEST_PG_URL`, the same in all three places; your database tests read it |
| `ci/ci-job.sh` line 24 | `readonly PROJECT=rottnest` | `readonly PROJECT=<name>`, because the counts line must begin `<name>-ci-job v1 counts` |
| `.github/workflows/ci.yml` lines 127, 128, 135, 136, 176, 177 | `rottnest-ci-gomod`, `-gobuild`, `-npm` | `<name>-ci-gomod` and so on; the workflow check requires `<project>-ci-<kind>` in each cache `path:` |
| `.github/workflows/ci.yml` line 1 | a comment | your project's name |
| `README.md` | the example's description | your own README (step 5 says what to keep) |
| `api/`, `web/` | the example's service, its module path `github.com/aames/quokka-ci-kit-example/api`, and `rottnest-web` | your own code, in step 5 |

The `Makefile` and `ci/image/Dockerfile` contain no project name; they refer to jobs and paths, which step 5 deals with.

*Done when* the grep prints only lines under `api/`, `web/` and `README.md`, which step 5 replaces.

## 5. Make it your shape (5 minutes)

The example has three local jobs: `api` (Go, with a throwaway Postgres), `web` (Node, TypeScript and vitest) and `kit` (the kit's own tests; always keep this one). Bring your own code in and remove the job you don't need.

**Both Go and Node.** Replace the contents of `api/` with your Go module and `web/` with your Node package. Your `go.mod` needs the line `toolchain go1.27.3` and your `web/.nvmrc` must read `24.22.1`, because those are the versions `ci/image/Dockerfile` installs; for other versions see "What still takes thought" below. If your directories have other names, rename them here and in every place the second table below lists for that job.

**Go only.** Delete `web/` (`git rm -r web`) and every `web` and Node line:

| File | Remove |
|---|---|
| `ci/manifest` | `job web local`, `paths web ...`, `pin web node`, `count web ...`, `install web npm-ci web`, `cache web npm`, `pin-source node ...`, `probe node ...`, `escaping-tree web web`, `escaping-scan ts web/src`; the word `web/.nvmrc` from `image-path` and from `paths api`; the word `web` from `prune-keep-green` |
| `ci/ci-job.sh` | the function `arm_web` and the case line `web) arm_web ;;` |
| `ci/image/Dockerfile` | the three `NODE_` arguments, `web/.nvmrc` from the `COPY` line, the two `.nvmrc` check lines, `node_arch` and `node_sha` in the two `case` arms, the four Node download and unpack lines, `/tmp/node.tar.xz` from the `rm` line, `/home/ci/.npm` from the `install -d` line, `/usr/local/node/bin:` from `PATH`, and the two `npm_config_` lines with the trailing `\` on the `GOCACHE` line above them |
| `.github/workflows/ci.yml` | the whole `web:` job, and the `web` and `web_key` outputs of `checks` |
| `Makefile` | `web` from `JOBS ?= api web kit` |

**Node only.** Delete `api/` (`git rm -r api`) and every `api` and Go line, mirroring the table above: in `ci/manifest`, the `api` lines including `service` and `cache api`, `pin-source go`, `probe go`, `escaping-tree api api`, `escaping-scan go api`, `api/go.mod` from `image-path` and `paths web`, and `api` from `prune-keep-green`; in `ci/ci-job.sh`, `arm_api` and its case line; in the Dockerfile, the three `GO_` arguments, `api/go.mod` from `COPY`, the two toolchain check lines, `go_sha`, the three Go download lines, `/tmp/go.tar.gz`, the three `/home/ci/go` directories and `/home/ci/.cache/go-build`, `/usr/local/go/bin:`, and `GOTOOLCHAIN`, `CGO_ENABLED`, `GOPATH`, `GOCACHE`; in the workflow, the `api:` job and its two outputs; in the `Makefile`, `api`. If your Node tests need Postgres, move the `service` line to `web` instead of deleting it.

**No database?** Delete the `service api ...` line and the `_TEST_PG_URL` check in `arm_api` (line 65).

**Your checks, not the example's.** Each job's function in `ci/ci-job.sh` runs the example's checks (`gofmt`, `go vet`, `go test`; `tsc --noEmit`, `vitest run`) and ends with one counts line whose fields match the job's `count` line in the manifest. Change the commands to yours, but keep the counts line, and make sure every count is above zero when tests have run.

**README.** Write your own, and copy into it, word for word, the trust paragraph from [README.md, How trust works](README.md#how-trust-works), with the two facts it mentions alongside. It says what a local record does and doesn't prove.

*Done when* `git status` shows only the changes you intended and step 4's grep prints nothing.

## 6. Commit and run the checks (5 minutes, most of it the first image build)

The checks read the commit, never your working copy, so commit first:

```sh
git add -A
git commit -m "Adopt quokka-ci-kit"
bash ci/kit/bin/check-manifest
make check
bash ci/kit/bin/check-adapter api
```

The first `make check` builds the kit's base image and your toolchain layer on top of it (a few minutes, once per image key), then runs each job in it. Run `check-adapter` once for each local job except `kit`. The two Python checks, `python3 ci/kit/bin/check-workflow` and `python3 ci/kit/bin/check-escaping-reads`, need PyYAML on your machine; if you don't have it, CI runs both in the `checks` job.

*Done when* each command exits 0, and `git -c gpg.ssh.allowedSignersFile=ci/allowed-signers log --show-signature -1` reports `Good "git" signature for <your email>`. That proves the line in `ci/allowed-signers` is the key that signs your commits.

## 7. Land it (5 minutes, mostly waiting)

Push without the hook, so that CI runs every job once and writes the first green records itself:

```sh
git push --no-verify -u origin adopt-kit
gh pr create --fill
```

In that run, the `checks` job plans: its summary table lists each job with its key, the record it found, the platform that record's run was on, where the record came from, and the decision. Every decision is **run** this time, because no record exists yet for your new keys and `main`'s signers file still names the example's author. Each job that passes writes `refs/ci/green/<job>/<key>`.

*Done when* every job in the run is green.

## 8. Protect `main` (2 minutes)

In **Settings, Rules, Rulesets, New branch ruleset**, target the default branch and turn on: require a pull request before merging; require signed commits; require status checks to pass, adding `checks` and each job you kept (`api`, `web`, `kit`); block force pushes. A required job that the plan skips counts as passing, which is the whole point. Then merge the pull request.

*Done when* the ruleset reads back with those four rules and the pull request is merged. The merge's run on `main` shows every test job as **green** with provenance `ci`, and only `checks` ran.

## 9. Turn on the hook and push a real change (3 minutes)

On each machine you work from, once per clone:

```sh
git switch main && git pull
git config core.hooksPath .githooks
bash ci/kit/bin/check-prereqs
```

Then change something under `api/` or `web/` (a change only to docs changes no key, so nothing would run), commit it on a new branch, and push without `--no-verify`. The hook runs each job whose key changed, inside the same image CI uses, and on a pass signs and pushes a record. Look for lines like `api: signed refs/ci/local/api/<key>`. A failing job stops the push. Open the pull request.

*Done when* the pull request's plan summary shows `local: <your email>` in the provenance column next to each job you changed, and only `checks` ran.

```text
        .-"""-.          .---.
       / .===. \        ( 29:58 )
       \/ ^ ^ \/ _______/'---'
       ( \_o_/ )
  _____\/___\/________
```

Two seconds to spare, and the runner never woke up.

## On Mondays

Every Monday at 06:20 UTC (`weekly-cron` in `ci/manifest`; the workflow check keeps the workflow's `schedule` equal to it), CI ignores every local record and runs, on the tip of `main`, each job that CI itself hasn't passed for that exact content. After a week of local records that is usually every job, so budget for one full run a week. That is deliberate. The `prune` job lists records older than 30 days; a manual run with `task: weekly` and `delete` ticked deletes them. To move the day or time, change both lines together.

*Done when* the first Monday's run is green and its summary includes the sentence saying that the weekly run ignored every local record. Read it each week: a red there is a red `main`.

## What still takes thought

- **A different Go or Node version.** The pin lives where the tool reads it (`toolchain` in `api/go.mod`, `web/.nvmrc`) and again in `ci/image/Dockerfile`, with the SHA-256 of each release archive for amd64 and arm64; the build stops if the two disagree. Take Node's sums from `https://nodejs.org/dist/v<version>/SHASUMS256.txt` and Go's from `https://go.dev/dl/` (the `linux-amd64` and `linux-arm64` archives). Change both files in one commit; every local job runs again.
- **A different layout.** A directory name appears in the manifest (`paths`, `image-path`, `pin-source`, `install`, `escaping-tree`, `escaping-scan`), in the job's `cd` in the test script, in the Dockerfile's `COPY`, and in the workflow's `hashFiles(...)` cache keys. A job's `paths` must cover everything its tests read; anything outside the job's own directory belongs on its `paths` line.
- **Escaping reads.** `check-escaping-reads` only looks at Go and TypeScript test files. A helper your tests import that reads files at run time (migrations through a relative path, a fixture loader), or a path built up at run time rather than written out, is yours to declare with an `escaping-read` line ([kit/README.md](kit/README.md) §3.4).
- **Job time bounds.** Each workflow job has a `timeout-minutes`. Measure your suite's real time in CI and set it to about one and a half times the worst run.

## Python

`kit-init --python <dir>` writes a complete Python job when the directory contains an exact `.python-version` (such as `3.12.15`) and a `requirements*.txt` whose lines carry `--hash=` and include `pytest`. The job image carries that CPython from python-build-standalone, checked against its published checksums for both architectures. The job creates a virtual environment, installs your requirements with `pip install --require-hashes`, runs pytest and counts the tests that passed. Without a hashed requirements file it writes a stub that refuses to run until you fill it in. The template has no Python job, so for a Python project use Faster: kit-init above. No scanner reads Python tests, so declare any test that reads outside its own directory with an `escaping-read` line ([kit/README.md](kit/README.md) §3.4).

## When the template isn't the right start

To adopt the kit in a repository you already have, with its history, use Faster: kit-init at the top of this page. Every step, each with a way to confirm it is done, together with the route for writing the files by hand, is in [kit/README.md](kit/README.md) §7: the same files, and one extra pull request, because your `main` has no signers file until you add one.
