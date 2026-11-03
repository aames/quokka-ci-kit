# Changelog

One entry per release, newest first. Each entry names the tag, the date, whether it moves keys (every release does: `ci/kit` is in the key of every adapter-run job, so a kit change re-tests every such job once; see [kit/README.md](kit/README.md) §5.4), the `kit/` tree the tag names (`git rev-parse <tag>^{commit}:kit`), and the adopter's actions in one list. Versions follow [kit/README.md](kit/README.md) §5.1: MAJOR means the adopter must act; MINOR adds an optional directive, kind or scanner; PATCH is a fix that changes no interface. A base image digest bump is a PATCH release with a line here saying so. A release is a signed, annotated tag whose message's first line is `quokka-ci-kit <tag>`, the form `kit-update` accepts. An entry headed `(unreleased)` is not a release yet: nothing can be vendored from it. The release pull request replaces that word with the date and writes the `kit/` tree hash into the entry, and the tag names that pull request's merge commit, so the entry in the tagged commit carries the hash ([CONTRIBUTING.md](CONTRIBUTING.md), "How a release is cut").

## v1.1.0-rc.1 (unreleased)

- New: `kit/bin/kit-init`, the generator of an adopter's files (docs/kit-init.md):
  the manifest, the adapter, the job image's Dockerfile, the workflow, the hook
  shim, the Makefile's check target, the signers file and an agent section in
  AGENTS.md and CLAUDE.md, for Go, Node with vitest and Python with pytest,
  with a refusing stub arm for anything else. Python 3.9 or later, standard
  library only; run once, by hand.
- New: `kit/templates/`, the files it renders, signed with the release.
- Changed: check-manifest rule 6 accepts the path ., as its text always said;
  check-escaping-reads passes a manifest with no escaping-scan line, saying no
  file was scanned, and counts depth correctly under a tree of .
- Unchanged: the manifest format (`manifest v1`), the adapter contract, the key
  formula, the plan's sentences, `check-host`'s file list and allow-list; every
  check but those two.
- The proofs: the K-init family in `kit/tests/run` (every supported shape
  generated offline and held to the lint and both fences), one shape end to end
  in `kit/tests/image`, and planted defects in `kit/tests/plants`.
- Moves keys: yes (every adapter-run key moves with `ci/kit`).
- The `kit/` tree this tag names: `<written by the release pull request>`.
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
