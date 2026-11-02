# Contributing

```text
                          .-"""-.
   run the tests first   / .===. \
   under both bashes    \/ 6 6 \/
                      __( \___/ )__
```

Thank you for looking. The kit decides whether CI may skip a job, so a change here is a change to trust code in every repository that vendors it. Every rule below exists for that reason.

## Running the kit's tests

From the top of a clone:

```sh
bash kit/tests/run                    # every family that needs no Docker daemon, on shims
bash kit/tests/run --list             # the family names
bash kit/tests/run --family plan      # one family
bash kit/tests/image                  # the base image and the entry script, against your Docker
python3 kit/tests/plants              # one planted defect per family; each must turn its family red
```

`tests/run` needs bash and `python3`. It looks for a bash 4.2 or later for the runner-only families and a `python3` with PyYAML for the workflow fence's family; where either is missing it prints a `NOT RUN` line for those legs instead of passing them, so run it once where both exist (the kit's CI does). `tests/image` needs Docker running; its first build compiles bash 3.2.57 and installs the base's packages, which takes several minutes, and later runs reuse Docker's build cache. It never starts, stops or reconfigures Docker, and removes the images it tagged. Each harness prints `ok:` or `FAIL:` per case and `NOT RUN <leg>: <reason>` for a leg your platform cannot run, and exits 0 only when every case held. Exit codes are read directly, never through a pipe.

A change that adds or changes a rule adds a case that fails without the change, and a plant in `kit/tests/plants` when the rule is one a single deleted line could silently remove. A test that only passes proves nothing until you have seen it fail.

The kit's own workflow runs, on every pull request: `bash -n` over every bash file under `kit/`, `bash kit/bin/check-manifest --file kit/tests/fixture/manifest --no-vendor-check`, `python3 kit/bin/check-host kit/`, `python3 kit/bin/check-workflow --fixture kit/tests/fixture/workflows/`, then `bash kit/tests/run` and `bash kit/tests/image`. Run the same before you open a pull request.

## The host side is bash 3.2

The scripts that run on a developer's workstation (`kit/bin/local-ci`, `kit/bin/pre-push`, `kit/bin/container`, `kit/bin/keys`, `kit/bin/kit-update`, `kit/bin/check-prereqs`, and the host-reached functions of `kit/lib/keys.sh`) must work under the macOS `/bin/bash`, which is 3.2, and under anything later: the hook shim and a Makefile run whichever `bash` is first on `PATH`, so the same file meets 3.2 on one machine and 5.x on the next. In those files:

- no `declare -A`, `mapfile`, `readarray`, `${x,,}`, `${x^^}`, `[[ -v`, `wait -n`, negative subscripts or `coproc`;
- only the host allow-list of commands ([kit/README.md](kit/README.md) §2.2): anything that needs a parser runs inside the job image instead;
- every child shell is started as `"$BASH"`, never a `bash` found on `PATH`.

`python3 kit/bin/check-host kit/` holds the host files to this and fails on a finding. The runner-only scripts (`plan`, `record`, `prune`) may use bash 4.2.

## Shellcheck, by hand

Shellcheck is not in the kit's CI yet, because the runner's package floats. Run shellcheck 0.11.0 on every shell file you changed and paste its exit code in the pull request:

```sh
shellcheck --version          # must say 0.11.0
shellcheck kit/bin/<file>     # exit 0
```

## Documents

`DESIGN.md` is the design reference and `kit/README.md` the command and manifest reference; code comments cite their sections as `DESIGN.md 6.2` or `pattern 6.2`. Keep a section's number when you change its text. A change to `kit/README.md` changes the `kit/` tree hash, like any change under `kit/`.

## How a release is cut

1. **The release pull request** adds the top entry to `CHANGELOG.md`: the tag, the date, whether it moves keys (every release does, because `ci/kit` is in the key of every adapter-run job), the adopter's actions in one list, and the `kit/` tree the tag will name, written into the entry in place of its placeholder. The tag names that pull request's merge commit, so the entry in the tagged commit carries the hash; `CHANGELOG.md` sits outside `kit/`, so writing the entry does not move the hash it records. Nothing else under `kit/` changes after the entry is written. The same pull request fills the release signer's line and fingerprint in [README.md](README.md), section "Releases and the release signer", at the first release. Check the hash before you push:

   ```sh
   git rev-parse HEAD:kit
   ```

2. **The tag**, after that pull request merges, from a clone whose `user.signingkey` is the release signer's key that README.md publishes, the key the adopters' `ci/KIT-SIGNERS` lists: a signed, annotated tag on `main` whose message's first line is exactly `quokka-ci-kit <tag>`:

   ```sh
   git fetch origin main
   git tag -s v1.2.3 origin/main -m "quokka-ci-kit v1.2.3"
   git push origin v1.2.3
   ```

   `kit-update` accepts nothing else as a release: a lightweight tag, a tag whose message does not open with that line, a tag whose own name differs, or a commit not on `main` is refused. Protect `refs/tags/v*` with a tag ruleset (creation by the release signer only, no update, no deletion).

3. **Verify it before anyone vendors it**, from any clone holding a `ci/KIT-SIGNERS`:

   ```sh
   bash kit/bin/kit-update --verify-only v1.2.3
   ```

   It prints the five `KIT-VERSION` lines. The `tree` line must equal the CHANGELOG entry's, which is `git rev-parse v1.2.3^{commit}:kit`.

A candidate (`v1.0.0-rc.1`) is cut the same way. A final release is tagged on the same `kit/` tree as the last candidate once an adopter has run that candidate green, so no adopter re-tests twice for one release.

## Versioning

- **MAJOR** means an adopter must act: the manifest format (`manifest v2`), the adapter contract, the key formula or one of the plan's sentences changed.
- **MINOR** adds something optional: a new directive, kind or scanner. Nothing an adopter has written needs to change.
- **PATCH** changes no interface: a fix, or a base image digest bump.

Every release moves the keys of every adapter-run job, a patch included, so every adopter re-tests once per upgrade. Say so in the entry.
