# Changelog

One entry per release, newest first. Each entry names the tag, the date, whether it moves keys (every release does: `ci/kit` is in the key of every adapter-run job, so a kit change re-tests every such job once; see [kit/README.md](kit/README.md) §5.4), the `kit/` tree the tag names (`git rev-parse <tag>^{commit}:kit`), and the adopter's actions in one list. Versions follow [kit/README.md](kit/README.md) §5.1: MAJOR means the adopter must act; MINOR adds an optional directive, kind or scanner; PATCH is a fix that changes no interface. A base image digest bump is a PATCH release with a line here saying so. A release is a signed, annotated tag whose message's first line is `quokka-ci-kit <tag>`, the form `kit-update` accepts. An entry headed `(unreleased)` is not a release yet: nothing can be vendored from it. The release pull request replaces that word with the date and writes the `kit/` tree hash into the entry, and the tag names that pull request's merge commit, so the entry in the tagged commit carries the hash ([CONTRIBUTING.md](CONTRIBUTING.md), "How a release is cut").

## v1.2.0 (2026-11-10)

- The platform release. Its `kit/` tree is `v1.2.0-rc.1`'s, unchanged: the
  example repository vendored the candidate and ran it, every job ran in CI
  and wrote its green record in the new shape (a tag object named
  `ci/green/<job>/<key>`, which GitHub accepted and stamped with its own
  tagger, its facts reading `platform linux/amd64`), and a second push with
  nothing under any key changed skipped all three jobs on those records. The
  candidate's one open fact is closed by that run.
- Moves keys: no change from `v1.2.0-rc.1` (the same tree); yes from `v1.1.0`
  (every adapter-run key moves with `ci/kit`).
- The `kit/` tree this tag names: `f9b648a78fb85390babef3559e3811b8776e3106`
- Adopter's actions: re-vendor with `kit-update v1.2.0` (from `v1.2.0-rc.1`,
  only `ci/KIT-VERSION` changes; from `v1.1.0`, nothing beyond the re-vendor).
  One check: `<job>_platform` is now a kit output name, so a plan extension
  that writes one fails the plan, naming it; rename yours if you have one.

## v1.2.0-rc.1 (2026-11-10)

- MINOR: every record names the platform (os/arch) its verdict ran on, and the
  plan shows it. The manifest format, the adapter contract, the key formula and
  every decision are unchanged: each record an earlier release wrote is decided
  as before. The plan gains a `Platform` column, a `<job>_platform` output and a
  last clause on a trusted record's note, all additive (kit/README.md §5.1).
- New: a CI green record is an unsigned tag object of the tested commit whose
  message is a facts block (`<project>-green v1`: job, key, commit, platform,
  host, image, run, workflow_ref), with `refs/ci/green/<job>/<key>` naming the
  tag object (DESIGN.md §5.2). It is still trusted by the check run on its
  commit, never by a signature.
- New: a `local` job's platform is the job image's, read inside the image by
  `container run` after a passing verdict and handed to `record` in the ran
  file `<git common dir>/<project>-ci/ran/<job>/<key>`. `record` refuses with
  exit 2, writing nothing, when that file is absent or malformed. Any other
  job's platform is the runner's.
- New: the plan trusts both shapes of green record (DESIGN.md §5.4), peeling a
  tag object to its commit and refusing one whose facts disagree with its ref,
  in its own sentence. Its summary table gains a `Platform` column between
  `Record` and `Provenance`, and its outputs gain `<job>_platform` (empty when
  no record decided or the record is a lightweight one), which an extension
  cannot write.
- New: `kit_platform_of` in `lib/keys.sh`, the one computer of the platform
  string, called by `local-ci`, `container` and `record`.
- Changed: in the note of a green record that is a tag object, `<sha12>` names
  the tag object, as it always has for a local record; a lightweight record's
  note still names its commit.
- Changed: `local-ci sign` and `push` name the record's platform in their
  lines. The local facts block and its format line are unchanged, so every
  local record written by an earlier release stays valid.
- Fixed: `prune green` ages a green record that is a tag object by its commit's
  committer date, where it would have kept it as dateless.
- Moves keys: yes (every adapter-run key moves with `ci/kit`).
- Proven on: linux/arm64, a workstation run at this tree on 2026-11-10 (`bash
  kit/tests/run` under bash 3.2 and 5.3: `cases: 1440, failures: 0`; `python3
  kit/tests/plants`: 112 of 112 red; `bash kit/tests/image --family kitjob`:
  `cases: 4, failures: 0`), and linux/amd64, the `kit` job of the release pull
  request's CI run.
- The `kit/` tree this tag names: `f9b648a78fb85390babef3559e3811b8776e3106`
- Adopter's actions: none; the workflow's record step is unchanged, and
  records written by earlier releases stay trusted. One check: `<job>_platform`
  is now the kit's output name for every job your manifest declares, so a plan
  extension that writes one fails the plan, naming it; rename yours if you have
  one.
- To be proven live by this candidate: GitHub accepting a tag object named
  `ci/green/<job>/<key>` from the job token. The first write is the example's
  run of this candidate; `v1.2.0` is cut only after that run is green.

## v1.1.0 (2026-11-03)

- The `kit-init` release. Its `kit/` tree is `v1.1.0-rc.1`'s, unchanged: the
  candidate was vendored and run green by the example repository, and the
  generator's two end-to-end shapes (Go with Node and Postgres; Python with
  pytest) ran green on linux/amd64 and linux/arm64 at this tree, so nothing
  moved between the candidate and this entry.
- Moves keys: no change from `v1.1.0-rc.1` (the same tree); yes from `v1.0.0`
  (every adapter-run key moves with `ci/kit`).
- The `kit/` tree this tag names: `70287a6873ac019c95964ba7589736976f8fdbfc`
- Adopter's actions: re-vendor with `kit-update v1.1.0` (from `v1.1.0-rc.1`,
  only `ci/KIT-VERSION` changes; from `v1.0.0`, nothing beyond the re-vendor).
  A project adopted by hand may add the agent section with
  `python3 ci/kit/bin/kit-init --agent-files-only`.

## v1.1.0-rc.1 (2026-11-03)

- New: `kit/bin/kit-init`, the generator of an adopter's files (docs/kit-init.md):
  the manifest, the adapter, the job image's Dockerfile, the workflow, the hook
  shim, the Makefile's check target, the signers file and an agent section in
  AGENTS.md and CLAUDE.md, for Go, Node with vitest and Python with pytest,
  with a refusing stub arm for anything else. Python 3.9 or later, standard
  library only; run once, by hand.
- New: `kit/templates/`, the files it renders, signed with the release.
- Changed: `check-manifest` rule 6 accepts the path `.`, as its text always
  said. `check-escaping-reads` passes a manifest with no escaping-scan line,
  saying no file was scanned; under a tree of `.` it counts depth, accepts a
  declared escaping-read line and scans a `test/` directory exactly as it does
  under a named tree.
- Changed: no file under `kit/` has a name a test runner or compiler collects
  from a repository's top. The kit's own fixture sources end in `.fixture`, so a
  project whose tests run from its top never runs the kit's.
- Fixed: `kit-init` writes no file through a symbolic link. A file it would
  write or append to that is a link, or sits under one, is refused, and nothing
  is written.
- Fixed: every directory and file name `kit-init` puts into a generated file is
  letters, digits and `. _ / -`, so no name can carry shell or quotes into the
  verdict script, the job image or the workflow.
- Fixed: `kit-init` runs its own `python3` in isolated mode (`-I`), so a
  `yaml.py` at the top of the repository is never imported.
- Fixed: `kit-init` exits 3, not 0, when a check did not run (the workflow
  fence without PyYAML), and says which check and how to run it.
- Fixed: when the commit fails, `kit-init` says so in one line with git's exit
  code and message, and exits 1.
- Fixed: `kit-init` refuses a detached HEAD when it would commit, and says when
  it guessed the main branch because `origin/HEAD` is not set.
- Reviewed: a security read of `kit-init` before this release found one medium
  and five low findings, all fixed above, and six notes left for later releases
  (docs/kit-init.md, Revision 4).
- Unchanged: the manifest format (`manifest v1`), the adapter contract, the key
  formula, the plan's sentences, `check-host`'s file list and allow-list; every
  check but those two.
- The proofs: the K-init family in `kit/tests/run` (every supported shape
  generated offline and held to the lint and both fences), two shapes end to
  end in `kit/tests/image` (Go, Node and Postgres; Python with pytest), and
  planted defects in `kit/tests/plants`, each seen red.
- Proven on: linux/amd64, the kit's own CI run on the release pull request, and
  linux/arm64, a workstation run of both end-to-end shapes at the same tree;
  both are recorded in that pull request. The arm64 run: `bash kit/tests/image
  --family initgo --family initpy` at kit tree 70287a68 on 2026-11-03, `cases: 20,
  failures: 0`. The amd64 run: the `kit` job of the release pull request's CI run.
- Moves keys: yes (every adapter-run key moves with `ci/kit`).
- The `kit/` tree this tag names: `70287a6873ac019c95964ba7589736976f8fdbfc`
- Adopter's actions: none. A project that adopted by hand may add the agent
  section with `python3 ci/kit/bin/kit-init --agent-files-only`.

## v1.0.0 (2026-11-03)

- The first release. Its `kit/` tree is `v1.0.0-rc.2`'s, unchanged: the two
  candidates were vendored and run by the example repository, and nothing
  moved between them but this entry.
- Moves keys: no change from `v1.0.0-rc.2` (the same tree); yes from
  `v1.0.0-rc.1` (every adapter-run key moves with `ci/kit`).
- The `kit/` tree this tag names: `9e282b4f70e5d5a70abf5d35e01b9b5e3ccba9ba`
- Adopter's actions: re-vendor with `kit-update v1.0.0` (from `v1.0.0-rc.2`,
  only `ci/KIT-VERSION` changes).

## v1.0.0-rc.2 (2026-11-03)

- Fixed: `check-escaping-reads` counted a test file sitting directly in its
  escaping-tree as one level below the tree, so `api/x_test.go` reading
  `../web` was not flagged while `api/sub/x_test.go` reading `../../web` was.
  Such a file is now depth zero, and one `..` leaves its tree. A PATCH: a fence
  made sound, no interface changed. It is stricter, so a project green on rc.1
  can turn red on upgrade, and the fix is the fence's own advice: declare the
  read by an `escaping-read` line and key it.
- Unchanged: everything else in `v1.0.0-rc.1`.
- Moves keys: yes (every adapter-run key moves with `ci/kit`).
- The `kit/` tree this tag names: `9e282b4f70e5d5a70abf5d35e01b9b5e3ccba9ba`
- Adopter's actions: re-vendor with `kit-update v1.0.0-rc.2`; if the fence then
  names a test directly in its tree, declare that read and key it.

## v1.0.0-rc.1 (2026-11-03)

- The first public release, derived from the author's private reference implementation.
- The names it carries:
  - the Docker label on the base and job images:
    `io.github.aames.quokka-ci-kit.image-key`;
  - `kit-update`'s default URL: `https://github.com/aames/quokka-ci-kit.git`;
  - the release tag message's first line: `quokka-ci-kit <tag>`.
- The runner allowance: the launcher `kit/bin/container` passes the constant
  `CI=true` into every job container beside `GITHUB_ACTIONS=true`, under the
  same condition and no other, so a project's allowances that read `CI` apply
  on the runner; a workstation run carries neither, since `local-ci` unsets
  `GITHUB_ACTIONS`. `CI` never enters the facts.
- `CI` is a reserved name (`KIT_RESERVED_ENV` in `kit/lib/keys.sh`): an `env`
  line naming it is refused by the parser with its line number, by the lint
  and by the launcher, so a workstation exporting `CI` cannot loosen its own
  local record's bounds.
- The proofs: four cases in K-launcher and one in K-manifest (the constant
  under the runner's variables, its absence without them, its absence through
  `local-ci`, the refusal of `env ... CI`), and four planted defects in
  `kit/tests/plants`, each red on the real code.
- Moves keys: yes (every adapter-run key moves with `ci/kit`).
- The `kit/` tree this tag names: `16d76d5ca9e10b25652eaaba2bdadd42e844e44d`
  (`git rev-parse HEAD:kit` on that pull request's branch, equal to
  `git rev-parse v1.0.0-rc.1^{commit}:kit` once tagged).
- Adopter's actions: none; this is the first release.
