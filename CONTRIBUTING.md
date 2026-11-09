# Contributing

```text
                          .-"""-.
   run the tests first   / .===. \
   under both bashes    \/ 6 6 \/
                      __( \___/ )__
```

Thank you for looking. The kit decides whether CI may skip a job, so a change here is a change to the trust code of every repository that vendors it. That is the reason behind every rule below.

## Running the kit's tests

From the top of a clone:

```sh
bash kit/tests/run                    # every family that needs no Docker daemon, on shims
bash kit/tests/run --list             # the family names
bash kit/tests/run --family plan      # one family
bash kit/tests/image                  # the base image and the entry script, against your Docker
python3 kit/tests/plants              # one planted defect per family; each must turn its family red
```

`tests/run` needs bash and `python3`. Two groups of tests need a little more: the runner-only ones need bash 4.2 or later, and the workflow check's tests need a `python3` with PyYAML. Where either is missing, the harness prints a `NOT RUN` line for those tests rather than passing them, so run it at least once somewhere that has both (the kit's CI does). `tests/image` needs Docker running. Its first build compiles bash 3.2.57 and installs the base image's packages, which takes several minutes; later runs reuse Docker's build cache. It never starts, stops or reconfigures Docker, and it removes the images it tagged. Each harness prints `ok:` or `FAIL:` for each case and `NOT RUN <leg>: <reason>` for anything your platform cannot run, and exits 0 only when every case passed. Exit codes are read directly, never through a pipe.

A change that adds or alters a rule comes with a test case that fails without the change and, when a single deleted line could silently remove the rule, with a planted defect in `kit/tests/plants`. A test that has only ever passed proves nothing until you have seen it fail.

The kit's own workflow runs the following on every pull request: `bash -n` over every bash file under `kit/`, `bash kit/bin/check-manifest --file kit/tests/fixture/manifest --no-vendor-check`, `python3 kit/bin/check-host kit/`, `python3 kit/bin/check-workflow --fixture kit/tests/fixture/workflows/`, then `bash kit/tests/run` and `bash kit/tests/image`. Run the same before you open a pull request.

## The host side is bash 3.2

The scripts that run on a developer's workstation (`kit/bin/local-ci`, `kit/bin/pre-push`, `kit/bin/container`, `kit/bin/keys`, `kit/bin/kit-update`, `kit/bin/check-prereqs`, and the functions of `kit/lib/keys.sh` they call) have to work under the macOS `/bin/bash`, which is version 3.2, as well as under anything newer. The hook shim and the Makefile run whichever `bash` is first on `PATH`, so the same file meets 3.2 on one machine and 5.x on the next. In those files:

- no `declare -A`, `mapfile`, `readarray`, `${x,,}`, `${x^^}`, `[[ -v`, `wait -n`, negative subscripts or `coproc`;
- only the commands on the host allow-list ([kit/README.md](kit/README.md) §2.2); anything that needs a parser runs inside the job image instead;
- every child shell is started as `"$BASH"`, never as whichever `bash` happens to be on `PATH`.

`python3 kit/bin/check-host kit/` enforces this and fails on any finding. The scripts that only run on the CI runner (`plan`, `record`, `prune`) may use bash 4.2.

## Shellcheck, by hand

Shellcheck isn't in the kit's CI yet, because the version on the runner floats. Run shellcheck 0.11.0 on every shell file you changed and paste its exit code into the pull request:

```sh
shellcheck --version          # must say 0.11.0
shellcheck kit/bin/<file>     # exit 0
```

## Documents

`DESIGN.md` is the design reference and `kit/README.md` the reference for the commands and the manifest. Code comments cite their sections as `DESIGN.md 6.2` or `pattern 6.2`, so keep a section's number when you change its text. A change to `kit/README.md` changes the hash of the `kit/` tree, like any change under `kit/`.

## How a release is cut

1. **The release pull request** adds a new entry at the top of `CHANGELOG.md`: the tag, the date, whether it changes keys (every release does, because `ci/kit` is in the key of every job that runs through the test script), what adopters need to do, and the hash of the `kit/` tree the tag will contain, written in place of its placeholder. The tag points at that pull request's merge commit, so the entry in the tagged commit carries the hash. `CHANGELOG.md` sits outside `kit/`, so writing the entry doesn't change the hash it records. Nothing else under `kit/` changes after the entry is written. At the first release, the same pull request fills in the release signer's line and fingerprint in [README.md](README.md), under "Releases and the release signer". Check the hash before you push:

   ```sh
   git rev-parse HEAD:kit
   ```

2. **The tag**, once that pull request has merged. Work from a clone whose `user.signingkey` is the release signer's key, the one README.md publishes and adopters list in `ci/KIT-SIGNERS`. Make a signed, annotated tag on `main` whose message starts with exactly `quokka-ci-kit <tag>`:

   ```sh
   git fetch origin main
   git tag -s v1.2.3 origin/main -m "quokka-ci-kit v1.2.3"
   git push origin v1.2.3
   ```

   `kit-update` accepts nothing else as a release. It refuses a lightweight tag, a tag whose message doesn't start with that line, a tag whose own name differs, and a commit that isn't on `main`. Protect `refs/tags/v*` with a tag ruleset: creation by the release signer only, no updates, no deletions.

3. **Verify it before anyone vendors it**, from any clone that has a `ci/KIT-SIGNERS`:

   ```sh
   bash kit/bin/kit-update --verify-only v1.2.3
   ```

   It prints the five `KIT-VERSION` lines. The `tree` line must equal the one in the CHANGELOG entry, which is `git rev-parse v1.2.3^{commit}:kit`.

A release candidate (`v1.0.0-rc.1`) is cut the same way. The final release is tagged on the same `kit/` tree as the last candidate, once an adopter has run that candidate successfully, so nobody has to re-test twice for one release.

## Versioning

- **MAJOR** means adopters have to do something: the manifest format (`manifest v2`), the test script's contract, the key formula or a predicate sentence has changed. A predicate sentence is something the plan decides by and your workflow reads: a decision value, the name or meaning of an existing output, or the outcome of a trust check on a record an earlier release wrote ([kit/README.md](kit/README.md) §5.1).
- **MINOR** adds something optional: a new directive, kind or scanner. Nothing an adopter has written needs to change.
- **PATCH** changes no interface: a fix, or a bump of the base image's digest.

Every release, patches included, changes the key of every job that runs through the test script, so every adopter re-tests once per upgrade. Say so in the entry.
