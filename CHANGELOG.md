# Changelog

One entry per release, newest first. Each entry gives the tag, the date, whether it changes keys (every release does: `ci/kit` is in the key of every job that runs through the test script, so a kit change re-tests each of those jobs once; see [kit/README.md](kit/README.md) §5.4), the hash of the `kit/` tree the tag contains (`git rev-parse <tag>^{commit}:kit`), and what adopters need to do, in one list. Versions follow [kit/README.md](kit/README.md) §5.1: MAJOR means adopters have to act; MINOR adds an optional directive, kind or scanner; PATCH is a fix that changes no interface. A bump of the base image's digest is a PATCH release with a line here saying so. A release is a signed, annotated tag whose message's first line is `quokka-ci-kit <tag>`, which is the form `kit-update` accepts. An entry headed `(unreleased)` is not a release yet, and nothing can be vendored from it. The release pull request replaces that word with the date and writes the `kit/` tree hash into the entry, and the tag points at that pull request's merge commit, so the entry in the tagged commit carries the hash ([CONTRIBUTING.md](CONTRIBUTING.md), "How a release is cut").

## v1.2.1 (2026-11-10)

- PATCH: the documents rewritten in plain English (pull request #11). Inside
  `kit/`: `kit/README.md` and `kit/templates/agents/section.md` are reworded,
  so the section `kit-init` writes into an adopter's `AGENTS.md` and
  `CLAUDE.md` now says "the one test script, with a section for each job"
  where it said "each job's one verdict script, with one arm per job", and
  "alongside the branch" where it said "beside"; one expected string in
  `kit/tests/fam/init.sh` follows it; a comment in `kit/tests/plants` no longer
  cites a removed file. No command, directive, sentence a program prints or
  decision changes.
- Outside `kit/`: README, QUICKSTART, DESIGN.md, CONTRIBUTING (with a new "How
  we write" section), SECURITY, docs/kit-init.md (restructured as a plain
  description), the agent guide and the skill, all in the same voice; the
  internal review records under `docs/reviews/` are removed.
- Changes keys: yes (every key that includes `ci/kit`).
- The `kit/` tree this tag contains: `abf83004701c4b733e8e2da669eadb88a318efb9`
- What adopters do: re-vendor with `kit-update v1.2.1`; nothing else. The
  generator never rewrites a file it wrote earlier, so an `AGENTS.md` or
  `CLAUDE.md` written by an earlier release keeps the old wording unless you
  change the two bullets by hand.

## v1.2.0 (2026-11-10)

- The platform release. Its `kit/` tree is `v1.2.0-rc.1`'s, unchanged. The
  example repository vendored the candidate and ran it: every job ran in CI and
  wrote its green record in the new shape (a tag object named
  `ci/green/<job>/<key>`, which GitHub accepted and stamped with its own
  tagger, with facts reading `platform linux/amd64`), and a second push that
  changed nothing under any key skipped all three jobs on those records. That
  run settled the candidate's one open question.
- Changes keys: not from `v1.2.0-rc.1` (the same tree); yes from `v1.1.0`
  (every key that includes `ci/kit`).
- The `kit/` tree this tag contains: `f9b648a78fb85390babef3559e3811b8776e3106`
- What adopters do: re-vendor with `kit-update v1.2.0` (from `v1.2.0-rc.1`,
  only `ci/KIT-VERSION` changes; from `v1.1.0`, nothing beyond the re-vendor).
  One check: `<job>_platform` is now a kit output name, so a plan extension
  that writes one fails the plan, naming it; rename yours if you have one.

## v1.2.0-rc.1 (2026-11-10)

- MINOR: every record now says which platform (operating system and
  architecture) its tests ran on, and the plan shows it. The manifest format,
  the test script's contract, the key formula and every decision are
  unchanged: each record an earlier release wrote is decided exactly as before.
  The plan gains a `Platform` column, a `<job>_platform` output and a last
  clause on a trusted record's note, all additions (kit/README.md §5.1).
- New: a CI green record is an unsigned tag object of the tested commit whose
  message is a facts block (`<project>-green v1`: job, key, commit, platform,
  host, image, run, workflow_ref), with `refs/ci/green/<job>/<key>` pointing at
  the tag object (DESIGN.md §5.2). It is still trusted by the check run on its
  commit, never by a signature.
- New: a `local` job's platform is the job image's, read inside the image by
  `container run` after a passing run and handed to `record` through the ran
  file `<git common dir>/<project>-ci/ran/<job>/<key>`. `record` refuses with
  exit 2, writing nothing, when that file is absent or malformed. Any other
  job's platform is the runner's.
- New: the plan trusts both shapes of green record (DESIGN.md §5.4), peeling a
  tag object to its commit and refusing one whose facts disagree with its ref,
  with its own sentence. Its summary table gains a `Platform` column between
  `Record` and `Provenance`, and its outputs gain `<job>_platform` (empty when
  no record decided or the record is a lightweight one), which an extension
  cannot write.
- New: `kit_platform_of` in `lib/keys.sh`, the one function that computes the
  platform string, called by `local-ci`, `container` and `record`.
- Changed: in the note for a green record that is a tag object, `<sha12>`
  names the tag object, as it always has for a local record; a lightweight
  record's note still names its commit.
- Changed: `local-ci sign` and `push` name the record's platform in what they
  print. The local facts block and its format line are unchanged, so every
  local record written by an earlier release stays valid.
- Fixed: `prune green` ages a green record that is a tag object by its commit's
  committer date, where it would have kept it as dateless.
- Changes keys: yes (every key that includes `ci/kit`).
- Tested on: linux/arm64, a workstation run at this tree on 2026-11-10 (`bash
  kit/tests/run` under bash 3.2 and 5.3: `cases: 1440, failures: 0`; `python3
  kit/tests/plants`: 112 of 112 fail as they should; `bash kit/tests/image
  --family kitjob`: `cases: 4, failures: 0`), and linux/amd64, the `kit` job of
  the release pull request's CI run.
- The `kit/` tree this tag contains: `f9b648a78fb85390babef3559e3811b8776e3106`
- What adopters do: nothing; the workflow's record step is unchanged, and
  records written by earlier releases stay trusted. One check: `<job>_platform`
  is now the kit's output name for every job your manifest declares, so a plan
  extension that writes one fails the plan, naming it; rename yours if you have
  one.
- Left to prove live with this candidate: GitHub accepting a tag object named
  `ci/green/<job>/<key>` from the job token. The first write is the example's
  run of this candidate; `v1.2.0` is cut only once that run is green.

## v1.1.0 (2026-11-03)

- The `kit-init` release. Its `kit/` tree is `v1.1.0-rc.1`'s, unchanged: the
  example repository vendored and ran the candidate successfully, and the
  generator's two end-to-end shapes (Go with Node and Postgres; Python with
  pytest) passed on linux/amd64 and linux/arm64 at this tree, so nothing moved
  between the candidate and this entry.
- Changes keys: not from `v1.1.0-rc.1` (the same tree); yes from `v1.0.0`
  (every key that includes `ci/kit`).
- The `kit/` tree this tag contains: `70287a6873ac019c95964ba7589736976f8fdbfc`
- What adopters do: re-vendor with `kit-update v1.1.0` (from `v1.1.0-rc.1`,
  only `ci/KIT-VERSION` changes; from `v1.0.0`, nothing beyond the re-vendor).
  A project that adopted by hand may add the agent section with
  `python3 ci/kit/bin/kit-init --agent-files-only`.

## v1.1.0-rc.1 (2026-11-03)

- New: `kit/bin/kit-init`, the generator of an adopter's files (docs/kit-init.md):
  the manifest, the test script, the job image's Dockerfile, the workflow, the
  hook shim, the Makefile's check target, the signers file and an agent section
  in AGENTS.md and CLAUDE.md, for Go, Node with vitest and Python with pytest,
  with a stub section that refuses to run for anything else. Python 3.9 or
  later, standard library only; run once, by hand.
- New: `kit/templates/`, the files it renders, signed with the release.
- Changed: `check-manifest` rule 6 accepts the path `.`, as its text always
  said. `check-escaping-reads` passes a manifest with no escaping-scan line,
  saying that no file was scanned; under a tree of `.` it counts depth, accepts
  a declared escaping-read line and scans a `test/` directory exactly as it
  does under a named tree.
- Changed: no file under `kit/` has a name a test runner or compiler collects
  from a repository's top. The kit's own fixture sources end in `.fixture`, so
  a project whose tests run from its top never runs the kit's.
- Fixed: `kit-init` writes no file through a symbolic link. A file it would
  write or append to that is a link, or sits under one, is refused, and nothing
  is written.
- Fixed: every directory and file name `kit-init` puts into a generated file is
  letters, digits and `. _ / -`, so no name can carry shell or quotes into the
  test script, the job image or the workflow.
- Fixed: `kit-init` runs its own `python3` in isolated mode (`-I`), so a
  `yaml.py` at the top of the repository is never imported.
- Fixed: `kit-init` exits 3, not 0, when a check did not run (the workflow
  check without PyYAML), and says which check and how to run it.
- Fixed: when the commit fails, `kit-init` says so in one line with git's exit
  code and message, and exits 1.
- Fixed: `kit-init` refuses a detached HEAD when it would commit, and says when
  it guessed the main branch because `origin/HEAD` is not set.
- Reviewed: a security read of `kit-init` before this release found one medium
  and five low findings, all fixed above, and six notes left for later releases
  (docs/kit-init.md §11).
- Unchanged: the manifest format (`manifest v1`), the test script's contract,
  the key formula, the plan's sentences, `check-host`'s file list and
  allow-list; every check except the two above.
- The tests: the K-init family in `kit/tests/run` (every supported shape
  generated offline and held to the manifest check and both Python checks), two
  shapes end to end in `kit/tests/image` (Go, Node and Postgres; Python with
  pytest), and planted defects in `kit/tests/plants`, each seen to fail.
- Tested on: linux/amd64, the kit's own CI run on the release pull request, and
  linux/arm64, a workstation run of both end-to-end shapes at the same tree;
  both are recorded in that pull request. The arm64 run: `bash kit/tests/image
  --family initgo --family initpy` at kit tree 70287a68 on 2026-11-03, `cases: 20,
  failures: 0`. The amd64 run: the `kit` job of the release pull request's CI run.
- Changes keys: yes (every key that includes `ci/kit`).
- The `kit/` tree this tag contains: `70287a6873ac019c95964ba7589736976f8fdbfc`
- What adopters do: nothing. A project that adopted by hand may add the agent
  section with `python3 ci/kit/bin/kit-init --agent-files-only`.

## v1.0.0 (2026-11-03)

- The first release. Its `kit/` tree is `v1.0.0-rc.2`'s, unchanged: the two
  candidates were vendored and run by the example repository, and nothing
  moved between them but this entry.
- Changes keys: not from `v1.0.0-rc.2` (the same tree); yes from
  `v1.0.0-rc.1` (every key that includes `ci/kit`).
- The `kit/` tree this tag contains: `9e282b4f70e5d5a70abf5d35e01b9b5e3ccba9ba`
- What adopters do: re-vendor with `kit-update v1.0.0` (from `v1.0.0-rc.2`,
  only `ci/KIT-VERSION` changes).

## v1.0.0-rc.2 (2026-11-03)

- Fixed: `check-escaping-reads` counted a test file sitting directly in its
  escaping-tree as one level below the tree, so `api/x_test.go` reading
  `../web` was not flagged while `api/sub/x_test.go` reading `../../web` was.
  Such a file is now at depth zero, and one `..` leaves its tree. A PATCH: a
  check made sound, no interface changed. It is stricter, so a project that
  passed on rc.1 can fail on upgrade, and the fix is the check's own advice:
  declare the read with an `escaping-read` line and key it.
- Unchanged: everything else in `v1.0.0-rc.1`.
- Changes keys: yes (every key that includes `ci/kit`).
- The `kit/` tree this tag contains: `9e282b4f70e5d5a70abf5d35e01b9b5e3ccba9ba`
- What adopters do: re-vendor with `kit-update v1.0.0-rc.2`; if the check then
  names a test directly in its tree, declare that read and key it.

## v1.0.0-rc.1 (2026-11-03)

- The first public release, derived from the author's private reference implementation.
- The names it carries:
  - the Docker label on the base and job images:
    `io.github.aames.quokka-ci-kit.image-key`;
  - `kit-update`'s default URL: `https://github.com/aames/quokka-ci-kit.git`;
  - the release tag message's first line: `quokka-ci-kit <tag>`.
- The runner allowance: the launcher `kit/bin/container` passes the constant
  `CI=true` into every job container alongside `GITHUB_ACTIONS=true`, under the
  same condition and no other, so that a project's allowances that read `CI`
  apply on the runner; a workstation run carries neither, since `local-ci`
  unsets `GITHUB_ACTIONS`. `CI` never enters the facts.
- `CI` is a reserved name (`KIT_RESERVED_ENV` in `kit/lib/keys.sh`): an `env`
  line naming it is refused by the parser with its line number, by the manifest
  check and by the launcher, so a workstation exporting `CI` cannot loosen its
  own local record's bounds.
- The tests: four cases in K-launcher and one in K-manifest (the constant under
  the runner's variables, its absence without them, its absence through
  `local-ci`, the refusal of `env ... CI`), and four planted defects in
  `kit/tests/plants`, each failing against the real code.
- Changes keys: yes (every key that includes `ci/kit`).
- The `kit/` tree this tag contains: `16d76d5ca9e10b25652eaaba2bdadd42e844e44d`
  (`git rev-parse HEAD:kit` on that pull request's branch, equal to
  `git rev-parse v1.0.0-rc.1^{commit}:kit` once tagged).
- What adopters do: nothing; this is the first release.
