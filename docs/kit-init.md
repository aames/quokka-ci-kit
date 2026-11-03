# `kit-init`: the generator of an adopter's files

```text
        .-"""-.        ______
       / .===. \      |______|
       \/ ^ ^ \/     _|______|
       ( \___/ )___/  |______|    six files, one init
    ____ooo___ooo_____|______|____
```

Status: specification, built, corrected and proven; the proofs ratified on 2026-11-03 with four defects ruled into one last code piece, 3b, that lands before `v1.1.0-rc.1` is tagged (Revision 3, below; §11). It is written for the kit's maintainers and contributors. It names what `kit-init` does, what it writes and why, how its templates stay true, and how it is released. Where it cites the kit, the line numbers are those of the `v1.0.0-rc.1` tree (`kit/` tree `16d76d5c`, commit `4083f42`); where it cites the example adopter, they are those of [`quokka-ci-kit-example`](https://github.com/aames/quokka-ci-kit-example) at `8ad9623`, written `example:<path>:<line>`.

## Revision 3 (2026-11-03): the proofs ratified, four defects ruled, the release written

The proofs piece (commit `ed8b518`, on piece 1b `870e195`) was read against §10 and §11 piece 2, and its claims were re-run at `4375618` (the generator branch merged with the public `main` at `v1.0.0`) rather than taken on report. **Verdict: the proofs are ratified; four defects found, three by the proofs and one by this reading, are ruled into one bounded code piece, 3b (§11), and two of them block `v1.1.0-rc.1`.** Each ruling has its reason; the body carries `(rev 3, R3-n)` where it changed. Line numbers of the kit are those of `4375618`.

What was re-run on this workstation (macOS, linux/arm64 job images):

- **K-init:** `bash kit/tests/run --family init`, 465 cases, 0 failures, 34 s.
- **The two end-to-end families:** `bash kit/tests/image --family initgo --family initpy`, 20 cases, 0 failures, 101 s warm. The counts lines seen: `rottnest-ci-job v1 counts api go_packages 1 go_tests 6`, `... web vitest_files 1 vitest_tests 7`, `... kit kit_cases 1258`, `pyshape-ci-job v1 counts py pytest_tests 4`. The kit job counts 1,258 where the host run counts 1,269: the 11 AC-KI-27 cases read the kit repository's own workflow and print an info line instead in an adopter's tree, so the kit job there stays free of `NOT RUN`.
- **The eight `init` plants:** `python3 kit/tests/plants --only <each>`, all eight red (failures per plant 7, 2, 39, 3, 3, 15, 113 and 1).
- **The three defects as reported**, each in a scratch repository, and the fourth below.

### R3-1. The proofs against §10

| Criterion | Ruling | Evidence |
|---|---|---|
| AC-KI-2 | met | the syntax-tree test at 3.9 and its red-once source (`kit/tests/fam/init.sh:112`, `:135`); the version refusal under a faked 3.8.18 (`:143-147`) |
| AC-KI-3 | met | the subprocess walk and three red-once sources (`init.sh:112-129`) |
| AC-KI-4 | met | one case per refusal row of §2.3, each seeing exit 2, its sentence, and the status, index, HEAD and tree unchanged; the network refusal is reached offline through `--checksums` lines that are absent, which is the same sentence |
| AC-KI-6 | met | the family under a closed proxy with `--checksums`; the docker shim sees no digest lookup; plant `init-offline-branch` red with 113 failures |
| AC-KI-9 | met | the CRLF Makefile restored byte for byte after a planted write failure (`init.sh:349-352`); plant `init-restore-decoded` red |
| AC-KI-20 | met | the golden fixture `kit/tests/fixture/init/golden/` and its comparison, comment and blank lines set aside |
| AC-KI-27 | met | the guard step (`.github/workflows/ci.yml:94-131`), red once in-family on a copied body line, tokens filled in or not (`init.sh:567`, `:577`) |
| AC-KI-28 | met, the text narrowed | the eight plants red. "And only K-init" holds by construction for the seven `tests/run` plants: no other family reads `bin/kit-init` or `templates/` as code (their only mentions in `fam/escaping.sh` and `fam/manifest.sh` are comments). `init-python-path` turns `initpy` red, the end-to-end half of the same proofs. §10's text now says so |
| AC-KI-29 | met | `initgo`, above, on arm64; amd64 is the kit's own CI run (R3-7) |
| AC-KI-30 | met as written; widened by R3-3 | all eight shapes through the four checks; the `.` shape declares no `escaping-read` line, which is why defect 2 was not seen there |
| AC-KI-35 | not met for one file; piece 3b | every stack file, pin file, requirements file, checksums file and append target is refused (34 cases); `ci/KIT-VERSION` is not (R3-2) |
| AC-KI-36 | met | `initpy`, above, with plant `init-python-path` red, so the pinned interpreter, not the base image's `python3`, is the one that answers |
| §8's two rules | met | as AC-KI-2 and AC-KI-3 |

The fixture gap the proofs found in `kit/tests/lib.sh` (`fx_new` now vendors `templates`, `:160`) is ratified: the vendored kit job reads its templates like any adopter's.

AC-KI-23 is met for its first, stub and last steps (`init.sh:472`, `:489`, `:510`), not for the order of all twelve; the order was ratified by reading at revision 2 and is not re-opened.

### R3-2. Defect 1: the tag is read after the writes. Fixed in piece 3b; not blocking alone

`kit-init:1009-1010` reads `ci/KIT-VERSION` for the commit message after `write_all` (`:999`). A file there that is not UTF-8 refuses with `ci/KIT-VERSION is not UTF-8 text kit-init can read, so nothing was written.` after the files were written: reproduced, seven files written with `--no-agent-files`, nine without it. **Fix: read the tag once in `preflight_repo` (`kit-init:830-838`), where the file's presence is already checked, and pass it on; nothing else reads the file.** Why: §2.3 promises every refusal before any file is touched, and exit 2 after writing tells the person the opposite of what happened. Only a damaged `ci/KIT-VERSION` reaches it, since `kit-update` writes that file, so it would not hold the release by itself.

- Red once: a K-init case, `ci/KIT-VERSION` holding the byte `0xe9`, sees exit 2, the sentence, and the status, index, HEAD and tree unchanged; it fails on the current line.

### R3-3. Defect 2: the fence's path tests under a tree of `.`. Fixed in piece 3b; blocks the release

`check-escaping-reads:407` tests `rel.startswith(t + "/")`, which is never true for `.`, so a keyed `escaping-read` line under `escaping-tree go .` is refused with `db/a_test.go is declared for job go but is not in ./.`: reproduced. That is the line the fence's own FAILED sentence tells the person to write. Two more tests of the same shape were found beside it:

- `:295` takes a TypeScript test under a scan root's `test/` directory by `rel.startswith(top.rstrip("/") + "/test/")`; for a root of `.` the prefix is `./test/`, so `test/helper.ts` is never scanned. Shown by calling `test_files` on a scratch tree: the result is empty. A test the fence never reads is an under-report.
- `:375` decides whether a tree has a scan line; a tree of `.` with a scan root inside it (`--node .` when `src/` exists writes `escaping-scan ts src`) is told `its reads are declared by hand`, which is false.

**Fix: one helper, illustratively `within(path, tree)`, true when the tree is `.`, the path equals the tree, or the path starts with the tree and `/`; used at `:295` (a root of `.` takes `test/` itself), `:375` and `:407`; and the location word at `:408` reads `the repository's top` for `.` instead of `./`.** Why: `.` is a shape AC-KI-30 claims as supported, and revision 2 fixed its depth (R-2 item 3) but not these three tests of the same path; a supported shape whose fence refuses the declaration it asks for is not supported, and `:295` is a gap in a soundness fence.

- Red once (AC-KI-37): in the escaping family, (i) a keyed `escaping-read` line under a tree of `.` exits 0; (ii) under a `ts` scan root of `.`, a file `test/helper.ts` climbing out of the tree is a finding; (iii) a tree of `.` with the scan root `src` prints no "declared by hand" line for that tree. Each fails before the change. K-init's `.` shape gains one declared, keyed `escaping-read` line, so AC-KI-30 reaches the declaration path.
- Plant: `:407` restored to `rel.startswith(t + "/")`, red in the escaping family.

### R3-4. Defect 3: a stack directory with no test file. The scan line stays; a closing step is added. Fixed in piece 3b; not blocking alone

For a Go or Node job whose directory holds no test file, `kit-init` writes its `escaping-scan` line, the fence refuses with `the escaping-read fence found no test files, so it would prove nothing.` (exit 2), and `kit-init` exits 1: reproduced. **Ruling: keep writing the scan line, keep exit 1, and add one closing step, printed only when `check-manifest` exited 0 and `check-escaping-reads` exited 2: `Write a first test in <the scan directories>, commit it, and run python3 ci/kit/bin/check-escaping-reads again; CI's checks job refuses this tree until you do.`, numbered with the others, before `make check`.** Why not omit the line: the generator never rewrites its files, so a line left out at adoption leaves every test written later unscanned, and the fence would say "declared by hand" about a language it can read. Why not count test files in `kit-init`: that would be a second computer of the fence's own rule, which drifts. Exit 1 is already honest by §2.5 (written, and a check found something); what was missing is the way forward. A project with no test cannot pass `make check` either, because its counts are zero, so this is guidance, not a hole.

- Red once (AC-KI-39): a K-init case, a Go directory with `go.mod` and no `_test.go`, sees exit 1, the fence's sentence, the step naming the directory, and the scan line in the manifest; it fails on the current code for want of the step.

### R3-5. Defect 4, found by this reading: the vendored fixtures are collected by an adopter's own test runners. Fixed in piece 3b; blocks the release

Every adopter vendors `kit/` whole at `ci/kit/`, and the proofs added the kit's first test-shaped sources: seven `.go` files (four `_test.go`), four `.ts` files (three `.test.ts`) and two `test_*.py` files under `kit/tests/fixture/init/`. `v1.0.0`'s tree has none. A test runner that discovers from the repository's top therefore runs the kit's fixtures as the adopter's tests:

- **pytest, measured:** a scratch root holding `ci/kit` and one test of its own gives `2 failed, 5 passed`, both failures from `ci/kit/tests/fixture/init/e2e-python/tests/test_pinned.py`, which asserts Python 3.12.15 and `/usr/local/python`; the passing count includes the kit's tests too.
- **vitest and tsc, by their documented defaults (not run here):** vitest's default include takes `ci/kit/tests/fixture/init/src/novitest/src/plain.test.ts`, which uses `node:test`, so `vitest run` fails on a file with no suite; a root `tsconfig.json` without `include` type-checks every `.ts` under `ci/kit`.
- **The fence:** under `escaping-tree <job> .` the fence scans the kit's fixture tests as that job's (`test_files`, `check-escaping-reads:279-301`); none climbs today.
- **Go:** `go vet ./...` and `go test ./...` skip the fixtures because each sits in a nested module, and they pass `gofmt -l` today; a later fixture could break either.

This breaks `make check` for kit-init's own `--python .` and `--node .` shapes, and on upgrade for any `v1.0.0` adopter whose pytest, vitest or root `tsconfig.json` reads from the top, with no line of theirs changed. Under `kit/README.md` §5.1 that is a change the adopter must act on, in a release that says MINOR and "Adopter's actions: none". **Ruling: no file under `kit/` has a name that a test runner or compiler collects by default from a repository's top: none ends in `.go`, `.ts`, `.tsx`, `.mts`, `.cts`, `.js`, `.jsx`, `.mjs` or `.cjs`, and none is named `test_*.py`, `*_test.py` or `conftest.py`. The fixture sources are stored with `.fixture` appended to each such name, and the copy helpers strip it when they place a fixture in a scratch repository (`kit/tests/fam/init.sh:47` `init_put` and the other `$KI_FIX` copies; `kit/tests/image`'s `e2e` copies).** Why a naming rule and not a path outside `kit/`: the kit job runs `ci/kit/tests/run` inside every adopter, so its fixtures must travel in the vendored tree; a name no tool collects is the one placement every runner respects without configuration.

- Red once (AC-KI-38): a K-init case walks `$KITT_KIT` and fails on any file the rule names, red once on a planted `kit/tests/fixture/init/src/go/planted_test.go` in a copy; every other K-init, `initgo` and `initpy` case stays green on the renamed fixtures.

### R3-6. `kit/README.md` §6.2 follows the build

The family table gains K-init (`tests/run`) and the kit-init end-to-end shapes (`initgo`, `initpy`, in `tests/image`), and the "red once by" cells gain the eight `init` plants as built and the four plants of `v1.0.0-rc.2` and piece 1b (`escaping-top-depth`, `escaping-no-scan`, `escaping-dot-depth`, `manifest-rule6-dot`). §1's `tests/image` row names the two shapes. Why: §6.2 is the adopter's list of what is proven for them, and a family it does not name is a proof nobody knows to run.

### R3-7. The release

- **The CHANGELOG entry** (the first entry of `CHANGELOG.md`, `v1.1.0-rc.1 (unreleased)`) has §9.2's shape and sits above `v1.0.0`, newest first, as the merge placed it. Corrected: "The proofs" line named one shape end to end where two are built, and named no plant count; the `Changed:` line gains piece 3b's fence change and the fixtures' rule; a `Proven on:` line is added. §9.2 below is rewritten to match.
- **The tree hash and the date** are written by the release pull request, from `git rev-parse HEAD:kit` on its last commit, as `CONTRIBUTING.md`'s release steps say.
- **The two architectures (R-5).** amd64 is proven by the kit's own CI run on the release pull request's head: its `kit` job runs `bash kit/tests/image` whole, `initgo` and `initpy` included (`.github/workflows/ci.yml:152-153`). arm64 is one run of `bash kit/tests/image --family initgo --family initpy` on this arm64 workstation at the same head commit, its `cases:` line pasted into the pull request's description. The entry's `Proven on:` line names both. Today's arm64 run at `4375618` ratifies AC-KI-29 and AC-KI-36 here; it is not the release record, because piece 3b changes the tree.
- **The order.** Piece 3b, then its own re-run of `tests/run` under both bashes, the plants and `tests/image` on both architectures, then the release pull request. `v1.1.0-rc.1` waits for 3b.

### R3-8. §12 question 6 is closed

It was answered "rc.2 first": the depth fix shipped as `v1.0.0-rc.2` (`d5955a2`) and `v1.0.0` was tagged on that tree (`9e282b4f`, merge `df18216`). §9.1 is marked; R-2 item 3's depth line for `.` composed with it as revision 2 expected.

### R3-9. The cache-runtime action's comment

The sha `04d248b84655b509d8c44dc1d6f990c879747487` is the action's `v4` and `v4.0.0` tags (read from its repository's tag list). `kit/README.md:398` and `:524` read `# v3`; both now read `# v4.0.0`, as the template and the example do. The fixture workflows under `kit/tests/fixture/workflows/` carry the same stale comment; no rule reads it, and it is corrected at the next touch of those files, not in 3b.

### R3-10. The documents

Piece 3 is this revision's documents: `docs/adopt-with-an-agent.md` (the generator first), `README.md`, `QUICKSTART.md` (a `kit-init` section at the top, the signer rule said once for each route), `kit/README.md`, `docs/agents-section.md`, the skill and `CHANGELOG.md`. `docs/agents-section.md` was a hand-written section that differs from the built template, so the template wins: the document now points at `kit/templates/agents/section.md` and at `kit-init --agent-files-only`, and holds no copy (§3.9's one-copy rule).

## Revision 2 (2026-11-03): the first build ratified, the three shapes ruled

The first build (`kit/bin/kit-init`, 1,083 lines, and `kit/templates/`, 35 files, at commit `c0cabe5`) was read line by line against this document, and its claims were re-run rather than taken on report, each in a scratch copy (the example read through `git archive 8ad9623`, never checked out). **Verdict: ratified as built, with two corrections to the generator and two one-line changes to the rc.1 checks, all in one bounded next piece (§11, piece 1b) that lands before the proofs.** Each ruling has its reason; the body carries markers `(rev 2, R-n)` where it changed. Line numbers of `kit-init` and the templates are those of `c0cabe5`.

What was re-run:

- **The golden run (AC-KI-20).** `--project rottnest --go api --node web --postgres api --checksums <the example's five values>` on the example's tree with its files removed: the manifest, the adapter, the Dockerfile, the workflow, the shim and the Makefile equal the example's with comment and blank lines set aside. `ci/allowed-signers` differs by one line, the signer the example added after its landing, which §3.8 keeps out of the landing. Both scripts are mode 0755; the shim is byte-equal to `ci/kit/hooks/pre-push.shim`.
- **The workflow fence.** `check-workflow` on that output, under a Python with PyYAML: W-1 to W-12 hold, exit 0.
- **The commit (AC-KI-21).** The same run with `ci/` wholly untracked: the commit held exactly the seven written paths plus `ci/kit`, `ci/KIT-VERSION` and `ci/KIT-SIGNERS`. A pathspec-limited `git status` names `ci/kit/` itself (`kit-init:996`), so the vendoring is found even when git would collapse it to `ci/`.
- **The host rule (AC-KI-1 to AC-KI-3).** The source parses under `ast.parse(..., feature_version=(3, 9))`; its six `subprocess.run` calls have first elements `"git"`, `"docker"`, `"ssh-keygen"`, `"bash"`, `sys.executable` and `"bash"`, each a list, none with `shell` (`kit-init:102-128`); it imports `base64`, `json`, `os`, `re`, `subprocess`, `sys` and `urllib.request` and nothing else (`kit-init:30-36`); the file is mode 100755 with `#!/usr/bin/env python3`.
- **The tokens.** `TOKENS` (`kit-init:78-87`) holds 51 names, and the 35 templates use exactly those 51: none unused, none unknown.
- **The three shapes.** Python with pytest alone, Python without a hashed requirements file, and no stack at all are refused by `check-escaping-reads:375-377` and by nothing else; `--go .` is refused by `check-manifest:228` (once for the `escaping-tree` line, once for the `escaping-scan` line) and by nothing else.

Found beyond the build's report:

- **The append path's restore (R-1 a).** `read_text` (`kit-init:257-262`) opens in text mode, which turns CRLF into LF, and the failure path truncates an appended file to the length of that decoded text (`kit-init:773`). A Makefile of two CRLF lines is 16 bytes on disk and 14 decoded, so a write failure after its append would cut two of its original bytes. The success path is sound: it appends the new bytes and never rewrites the old ones (`kit-init:761-763`).
- **A file that is not UTF-8 (R-1 b).** `read_text` catches `OSError` only (`kit-init:261`). A Makefile holding the byte `0xe9` ends the run in a `UnicodeDecodeError` traceback with exit 1, which §2.5 defines as "written, and a lint found something"; nothing was written, so the honest answer is a refusal with exit 2.
- **The fence's depth under a tree of `.` (R-2).** `check-escaping-reads:228` takes the path inside the tree as `rel[len(tree) + 1:]`. For `.` that strips two characters, cutting into the first directory's name: under `escaping-tree go .`, `db/x_test.go` calling `filepath.Join("..", "..", "x")` is not flagged, while `dbx/x_test.go` with the same call is. The `.` shape needs this line as well as the lint's.
- **The template count.** The build named six template files beyond §7.1's list. There are eight: `manifest/stub` and `adapter/postgres.sh` are also new. §7.1 now lists all 35 (R-3).
- **A defect in the rc.1 fence outside this document's reach.** It is put to the maintainers as §12 question 6 and does not block this ratification.

### R-1. The build against §10

Ratified criterion by criterion. "Met" means met by the code as read and, where marked, by the re-run above; "proofs" means the criterion belongs to the proofs piece (§11, piece 2) and is not yet proven.

| Criterion | Ruling | Evidence |
|---|---|---|
| AC-KI-1 | met | `kit-init:1`, `:30-36`; mode 100755 |
| AC-KI-2 | met | re-run `ast.parse` at 3.9; the version refusal at `kit-init:38-41` |
| AC-KI-3 | met | `kit-init:102-128`; the syntax-tree walk above |
| AC-KI-4 | proofs | the refusals are all present (`kit-init:151-194`, `:265-406`, `:897-936`); AC-KI-35's refusal joins them |
| AC-KI-5 | met, with correction a | the rendered manifest is linted at `kit-init:974-978` before `write_all` at `:990`; the restore is corrected by R-1 a |
| AC-KI-6 | proofs | the offline branch returns before any fetch (`kit-init:441-471`) |
| AC-KI-7 | met | `kit-init:458-461`, `:466-470`: each missing value refused with its URL, no network branch after `--checksums` |
| AC-KI-8 | met | `InitDir` (`kit-init:786-818`): the state directory is removed only when this run made it; only `allowed-signers.next` survives |
| AC-KI-9 | new, proofs | assigned by this revision to the write-failure restore (§10) |
| AC-KI-10 | met for five shapes; three after piece 1b | the golden run; the three shapes above |
| AC-KI-11 | met | `kit-init:587`, `:596`; the golden manifest |
| AC-KI-12 | met | the golden `check-workflow`, exit 0; `@@CRON@@` at `manifest/head:16` and `workflow/head.yml:25` |
| AC-KI-13 | met | `adapter/tail.sh:4`, `adapter/stub.sh:6`; neither prints a counts line |
| AC-KI-14 | met | `kit-init:946-948`, `:956-958`; the golden `cmp` |
| AC-KI-15 | met by construction | one `FROM` in `image/head.Dockerfile:15`; the cache directories come from each tool's `dirs` part (`image/go.Dockerfile:20-21`, `image/node.Dockerfile:21-22`), and a non-stub job of that tool declares exactly those cache kinds (`kit-init:706-708`), so the two sets are equal |
| AC-KI-16 | met | `kit-init:980-987`, `:1029-1030`; the landing's file is the one comment line of `templates/allowed-signers` |
| AC-KI-17 | met on success; corrected on failure | `kit-init:761-763`; R-1 a |
| AC-KI-18 | met | `templates/agents/section.md` equals §3.9's text byte for byte; `kit-init:851-874` reads every value from `bin/keys dump HEAD`, where `check-command` is one field (`make check`), so `:865` renders it whole |
| AC-KI-19 | met | `check_tokens` (`kit-init:877-884`) over every text in the plan |
| AC-KI-20 | met by the re-run; the committed fixture is proofs | the golden run above |
| AC-KI-21 | met | `kit-init:993-1007`; the untracked-`ci/` re-run |
| AC-KI-22 | met | `kit-init:1009-1028`: four checks in order, each exit code printed, the `NOT RUN` line when PyYAML is absent |
| AC-KI-23 | met | `kit-init:1037-1076`; numbering ruled at R-4 d |
| AC-KI-24 | met | the generator reads `templates/`, `hooks/pre-push.shim`, `ci/KIT-VERSION`, `bin/keys` output and the adopter's own files |
| AC-KI-25 | met | `kit/bin/check-host` is unchanged between `ae12a6c` and `c0cabe5` |
| AC-KI-26 | met | `kit/README.md:9`, `:22`, `:74`, `:619`, `:622`; no body line of the agent section appears in either README |
| AC-KI-27 to AC-KI-30 | proofs | §11, piece 2 |
| AC-KI-31 | met; amended by piece 1b | `CHANGELOG.md:5-22`; the two check changes add a `Changed:` line and narrow the `Unchanged:` line (§9.2) |

**Correction a (the restore).** On a failure, an appended file is truncated to the length in bytes it had on disk before the append, taken from `os.stat` before the write, and the bytes appended are computed from the file read in binary, never from newline-translated text. Why: AC-KI-17 promises every original byte, and the restore is the one path where the build can break that promise.

**Correction b (a file that is not UTF-8).** Every file `kit-init` reads (a stack's `go.mod`, `package.json`, pin file or requirements file, and every append target) that does not decode as UTF-8 is a refusal, AC-KI-35: `<path> is not UTF-8 text kit-init can read, so nothing was written.` Why: an exit code is the contract (§2.5), and exit 1 after writing nothing tells the person the opposite of what happened.

### R-2. The three shapes: the checks change, and AC-KI-30 stands whole

**Ruling: change the two checks, one line each as the build proposed, plus the fence's depth line for `.`; AC-KI-30 keeps all eight shapes.** Reason: the lint change is a defect fix of a rule's own text, and the fence change states a true limit instead of refusing a project for something it cannot do; neither weakens what either check proves when it has something to prove.

1. **`check-manifest:228`, rule 6 counts `.` as present.** The line becomes, illustratively, `[ "$p" = . ] || in_tree "$p" || finding ...`. No new sentence: every other missing path keeps the existing finding. Why: rule 6's text is "every `escaping-tree` and `escaping-scan` PATH exists at the commit" (`kit/README.md:290`), `.` exists at every commit, and git refuses `<tree>:.` as a path (`fatal: path '.' exists on disk, but not in <tree>`), so the lint was failing its own rule. `covers` already treats `.` as the whole tree (`check-manifest:99`).
   - Red once (AC-KI-32): in the manifest family, a fixture whose manifest at HEAD holds `escaping-tree go .` and `escaping-scan go .` is held to exit 0 with no rule 6 finding; with the old line planted it fails with `escaping-tree names ., which does not exist at`. The existing case for a path that is truly absent stays red.
2. **`check-escaping-reads:375-377`, no `escaping-scan` line is a stated pass, not a refusal.** When the manifest holds no `escaping-scan` line at all, the fence prints `The manifest names no escaping-scan line, so no file was scanned and only the escaping-read lines were checked against the keys.`, runs the declaration checks it runs today (`check-escaping-reads:395-411`), and on success ends with `OK: no file was scanned; <n> escaping-read lines were checked, and every one is keyed.` When at least one `escaping-scan` line exists and its roots hold no test file, the refusal at `:376` stands unchanged with exit 2. Why: a scan root that finds nothing is a misdeclared root, which the refusal exists to catch; a manifest with no scan root has nothing to misdeclare, still has declarations to hold to the keys, and the per-tree sentence at `:373` already says which trees are declared by hand. A pass that prints what it did not do is the design's honesty rule; a refusal there would make every Python project red in its own `checks` job for a gap in the kit.
   - Red once (AC-KI-33): in the escaping family, (i) a manifest with no `escaping-scan` line and one keyed `escaping-read` line exits 0 and prints both sentences, and is red before the change (exit 2); (ii) the same with the declared path outside the key exits 1, so the declarations are still checked; (iii) a manifest whose one `escaping-scan` root holds no test file still exits 2 with `:376`'s sentence.
3. **`check-escaping-reads:228`, the depth under a tree of `.`.** The path inside the tree is `rel` itself when the tree is `.`, else `rel[len(tree) + 1:]`; the depth is counted from that by the existing rule, so a file under `.` is counted exactly as the same relative position under a named tree. Why: without it the `.` shape passes rule 6 and then under-reports climbs whenever its first directory is two characters long.
   - Red once (AC-KI-34): in the escaping family, under `escaping-tree go .`, `db/x_test.go` calling `filepath.Join("..", "..", "x")` is a finding; it is not one before the change.

These ship in `v1.1.0-rc.1`, never in `v1.0.0`, which §9.1 tags on rc.1's tree unchanged. The `v1.0.0` documents therefore stay true as they are: on rc.1 a manifest with no `escaping-scan` line is refused by the fence, and a tree of `.` is refused by rule 6. The CHANGELOG entry gains `Changed: check-manifest rule 6 accepts the path ., as its text always said; check-escaping-reads passes a manifest with no escaping-scan line, saying no file was scanned, and counts depth correctly under a tree of .`, and its `Unchanged:` line no longer claims every check is untouched (§9.2). `kit/README.md:189` gains one sentence: `A manifest with no escaping-scan line has nothing scanned; the fence says so, checks the escaping-read lines against the keys and passes.`

The work is piece 1b of §11.

### R-3. §7.1 and §7.2 amended to the templates and tokens as built

Ratified as built; §7.1 and §7.2 below are rewritten to match. Reasons: the larger token set names values that occur in several files at once (the paths line, `image-path`, `prune-keep-green`, the digest, the release tag, the pin files), which is exactly what §1 says the generator exists to keep consistent, and it is still a closed set with an unknown token and an unreplaced one both refused (`kit-init:215-221`, `:877-884`). The `@@@ <part>` section lines (`kit-init:203-212`) are ratified for the one case that needs them: a tool's image block is spread across one `RUN` (its argument lines, its pin cross-check, its two architecture arms, its download, its directories, its `PATH` entry and its `ENV` lines), and keeping one tool's parts in one file keeps that tool reviewable in one place. The rule that a line holding one token alone, whose value is empty, is dropped (`kit-init:224-226`) is written into §7.2 because it decides output.

### R-4. The four interpretations

- **a. A stub job gets no `pin`, `pin-source`, `cache` or `install` line and no image block. Ratified.** A `pin` line names a tool the image must carry and a probe must answer (rules 4 and 10), and a stub has no image block, so a pin there would be a claim nothing backs. The stub's manifest comment says to add those lines with the arm (`manifest/stub:3-4`). §3.1's `pin` row is amended. The census still refuses a stub whose pin file is missing or inexact (`kit-init:317-325`, `:345-353`), which is ratified too: the person needs that pin the moment they write the arm, and one edit now is cheaper than a stub that cannot be filled without one.
- **b. `--agent-files-only` writes the two files and does not commit them. Ratified.** No check reads them, so nothing needs them committed, and the person running it may be on any branch with other work in hand; the closing sentence says to commit them in a pull request of their own (`kit-init:873`). §2.2's row is amended.
- **c. On a write failure, created files are removed and appended files truncated to their original length. Ratified, corrected by R-1 a**: the length is the on-disk byte length. §2.3 is amended. The criterion is AC-KI-9.
- **d. The closing steps are numbered in sequence, so step 3 disappears and the rest close up when no stub was written. Ratified.** A gap in a numbered list reads as a missing step. §6 is amended.

### R-5. The Python path is proven end to end before `v1.1.0-rc.1`

**Ruling: an end-to-end Python shape in `kit/tests/image` (AC-KI-36) is part of the proofs piece, and it blocks `v1.1.0-rc.1`. There is no "sketch" label route.** Reasons: a release candidate is a claim that each supported shape works, the CHANGELOG entry names Python with pytest as supported, and a generator whose first Python project fails at its first `make check` is a worse first impression than a later candidate; a label in a generated file is read after the failure, not before. The Python arm has risks no lint sees, each of which this shape exercises: the base image carries Debian's own `python3` (`kit/image/base.Dockerfile:60`), so the arm must be shown to answer with the pinned interpreter from `/usr/local/python/bin`; `pip install --require-hashes` must reach the index over the job's bridge network (`kit/bin/container:645`); the python-build-standalone archive must extract to `/usr/local/python`; and the counts line must equal the fixture's passing tests. The kit's own workflow runs `kit/tests/image` on `ubuntu-latest` (`.github/workflows/ci.yml:94,108`), which proves amd64; the release pull request also records one green run of both end-to-end families on an arm64 workstation, with its summary line, because each image is built for its own machine's architecture and the arm64 checksums are otherwise never used.

If the shape cannot be made green in the proofs piece, the defect it finds is fixed before the tag, and the tag waits.

### R-6. The asset page bound of ten releases is ratified

`PBS_RELEASES` asks for the ten most recent releases (`kit-init:67`). Ratified. Reasons: the build's run timed out at `per_page=100` with about 990 assets a release, so a larger page is not available inside the 30-second bound (`kit-init:425`); and python-build-standalone builds only the newest patch of each CPython minor in each release, so a pin missing from the ten newest releases is an old patch that a larger page would seldom rescue. That case is refused with the URL and the `--checksums` route (`kit-init:476-479`), which is the documented path. §4.1 is amended.

### R-7. The proofs piece's brief

Written into §11, piece 2. It owns AC-KI-4, AC-KI-6, AC-KI-9, AC-KI-20's committed fixture, AC-KI-27 to AC-KI-30, AC-KI-35's case and AC-KI-36, each seen red once. It starts after piece 1b has landed, because AC-KI-30's three shapes are green only then.

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
| `--agent-files-only` | nothing; refuses every other flag | only the two agent sections, their values read from the committed manifest through `bin/keys dump HEAD` (`kit/README.md:13`), for a project that adopted by hand or before this command existed; it writes them and does not commit them (rev 2, R-4 b) | off |

**Job names.** `JOB` defaults to the directory with `/` written as `-` (`services/api` gives `services-api`), and to the stack's name for `.` (`go`, `node`, `python`). It must be a manifest NAME (`kit/README.md:135`). The generator also declares two jobs of its own, the always-on `checks` and the kit job `kit`, and writes a workflow job `prune`, so `checks`, `kit` and `prune` are refused as stack job names, as are `image` and `base`, which the manifest itself refuses (`kit/README.md:137`). Why fixed names for the two kit jobs: they are the example's (`example:ci/manifest:11,13`) and every reader of the kit's documents meets them there; a project that wants others renames them in its own pull request later, and the lints hold the rename.

**Repeated stacks.** Two `--go` directories make two jobs in one image, and the image carries one Go. So every Go directory's `toolchain` line must name the same version; the first directory's `go.mod` is the one `pin-source go` line and the one image input, and each later job's arm still reads its own `go.mod` at run time, so a later drift is refused by that arm (§3.2). The same holds for Node and Python. Why: `pin-source` takes one line per tool (`kit/README.md:185`), and an image with two Go versions would be two images.

### 2.3 The refusals

Every refusal is one sentence ending `so nothing was written.`, printed before any file is touched, with exit 2. The generator renders every file in memory first, then checks every refusal, then writes; a write that fails part way removes the files this run created, truncates each file it appended to back to the length in bytes that file had on disk before the run, and touches nothing else (rev 2, R-1 a, R-4 c).

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
| a file it reads that is not UTF-8 (a stack file, a pin file, a requirements file, an append target, or `ci/KIT-VERSION`, whose tag is read before anything is written) | `<path> is not UTF-8 text kit-init can read, so nothing was written.` (rev 2, R-1 b; rev 3, R3-2; AC-KI-35) |

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
| `pin <job> <tool>` | `go`, `node` or `python` | one per stack job that is not a stub; a stub gets no `pin`, `pin-source`, `cache` or `install` line, and its manifest comment says to add them with the arm (rev 2, R-4 a) |
| `count <job> ...` | Go `go_packages go_tests`; Node `vitest_files vitest_tests`; Python `pytest_tests`; a stub `tests`; the kit job `kit_cases` | the arm's counts line (§3.2); the kit job's line is fixed (`kit/README.md:172`) |
| `install <job> npm-ci <dir>` | for a Node job whose directory holds `package-lock.json` | the one install kind (`kit/README.md:173`); without a lockfile the arm is a stub (§3.2) |
| `service` | `service <job> postgres image postgres:<major>@sha256:<digest> url-env <PROJECT>_TEST_PG_URL flag fsync=off flag synchronous_commit=off` | `--postgres`; the digest from §4.2; the line is the example's (`example:ci/manifest:47`) |
| `cache` | Go `gomod gobuild`; Node `npm`; Python none; a stub none (rev 2, R-4 a) | the three kinds (`kit/README.md:177`); Python has no cache kind in `manifest v1`, so its runs install cold (§3.2) |
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
| Python tarball sha256 and its release tag | `https://api.github.com/repos/astral-sh/python-build-standalone/releases` (unauthenticated, the ten most recent releases, `per_page=10`; rev 2, R-6) to find the newest release carrying `cpython-<V>+<tag>-...-install_only_stripped.tar.gz` for both architectures, then that release's `SHA256SUMS` | CPython publishes no Linux binary; this project's builds are the relocatable ones the Python tool ecosystem installs, with a checksum file per release. §12 asks whether the kit accepts it |
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

The steps are numbered in sequence: when no stub was written, step 3 is not printed and the steps after it close up (rev 2, R-4 d).

When `check-manifest` exited 0 and `check-escaping-reads` exited 2 (no scan root holds a test file), one step comes before step 1: `Write a first test in <the scan directories>, commit it, and run python3 ci/kit/bin/check-escaping-reads again; CI's checks job refuses this tree until you do.` The scan line is still written (rev 3, R3-4; AC-KI-39).

Then one line on what was declared by hand: for each Python job, that its escaping reads are a census by hand (`kit/README.md:188`).

## 7. The templates

### 7.1 Where they live

`kit/templates/` inside the vendored tree, so every template is part of the `kit/` tree a release signs and `kit-update` verifies (`kit/README.md:347-357`), and `ci/kit/templates/` in an adopter. As built, 35 files (rev 2, R-3; the first list here named 27, and the build added eight):

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

A closed set of `@@NAME@@` tokens, replaced by plain string substitution (rev 2, R-3: the first list here named 14, too few to express values that appear in several files at once). As built, 51 names, each used by some template and none outside the set (`kit/bin/kit-init:78-87`):

- the project and its fixed names: `PROJECT`, `PROJECT_UPPER`, `MAIN`, `CRON`, `ADAPTER`, `WORKFLOW`, `SIGNERS`, `CHECK`;
- a job: `JOB`, `FN` (the job with `-` written `_`, for the arm's function name), `DIR`, `PATHS`, `PIN_FILE`, `PIN_BASE`, `SCAN_DIR`, `LOCKFILE`, `REQUIREMENTS`, `NEED_BINS`, `TSC_STEP`, `SERVICE_CHECK`;
- lists over all jobs: `JOBS`, `KEEP_GREEN`, `IMAGE_PATHS`, `CASES`, `OUTPUTS`, `CACHES`, and the manifest's grouped lines `LINES_JOB`, `LINES_PATHS`, `LINES_PIN`, `LINES_COUNT`, `LINES_INSTALL`, `LINES_SERVICE`, `LINES_CACHE`, `LINES_PIN_SOURCE`, `LINES_PROBE`, `LINES_ESCAPING_TREE`, `LINES_ESCAPING_SCAN`;
- a tool and a service: `VERSION`, `SHA256_AMD64`, `SHA256_ARM64`, `PBS_TAG`, `DIGEST`;
- the image's assembled parts: `ARGS`, `PIN_FILES`, `CROSS_CHECKS`, `AMD64`, `ARM64`, `TOOL_NAMES`, `FETCH`, `FINISH`, `ENV`.

Three rules decide output, each enforced in the generator:

1. **A token outside the set is a defect of the templates**, refused with exit 2 before anything is written (`kit-init:215-221`); **a token left unreplaced** in any file the run would write is refused the same way (`kit-init:877-884`).
2. **A line that is one token alone, whose value is empty, is dropped** (`kit-init:224-226`), so an absent part leaves no blank line behind.
3. **A part that repeats is rendered once per item.** Most such parts are a file of their own (a job's manifest lines, an arm, a case line, a cache step). Two kinds of template hold several named parts in one file, each part opening with a line `@@@ <part>` that is read as data, never output (`kit-init:203-212`): a tool's image block (`name`, `args`, `check`, `amd64`, `arm64`, `fetch`, `tmp`, `dirs`, `path`, `env-comment`, `env`), whose parts are spread across the one `RUN` and `ENV` of `image/run.Dockerfile`, so keeping one tool's parts in one file keeps that tool reviewable in one place; and `workflow/output.yml` (`decision`, `key`). A `@@@` line is a part header only in those four files.

No template language, no `eval`, no shell: the generator is standard-library Python. Why so plain: a template engine would be a dependency on every workstation, and logic inside templates is code the lints cannot see.

### 7.3 How they stay true

- **Every supported shape is generated and linted in the kit's own tests.** A new family, K-init, in `kit/tests/run` (no Docker daemon, offline through `--checksums` and the existing `docker` shim, `kit/tests/fixture/shims/docker`) generates each shape of AC-KI-30 into a fixture repository with a vendored `ci/kit` copy and holds it to: the generator's exit 0; `check-manifest` at HEAD (rules 1 to 12); `check-workflow`; `check-escaping-reads`; and the adapter's unknown-job leg run directly (`bash ci/ci-job.sh no-such-job` exits 2 with no counts line, `check-adapter`'s leg (a), `kit/bin/check-adapter:8-10`), because `check-adapter` itself needs the job image.
- **Two shapes end to end.** `kit/tests/image` gains a family that generates the Go, Node and Postgres shape, builds its job image against the real checksums of a fixed recorded version, runs `local-ci run` for each job and then `check-adapter` for each stack job; and the same for the Python with pytest shape against a recorded python-build-standalone release (rev 2, R-5; AC-KI-36).
- **The example as a golden output.** The fixture directory holds the example's six files for the `rottnest` shape; K-init compares its output with them, comment lines set aside, so a template that drifts from the reference adopter turns red.
- **Plants.** `kit/tests/plants` gains planted defects for the rules a single deleted line could remove: the existing-file refusal, the Dockerfile's pin cross-check, the shim copied from anywhere but the kit's file, an unreplaced token passing, and the restore truncating by decoded length (rev 2). The three check changes of R-2 carry their own plants in piece 1b. As built, eight `init` plants: those five, the UTF-8 refusal, the offline branch and the pinned Python off the image's `PATH` (the last seen by `initpy`) (rev 3, R3-1).
- **Fixtures no adopter's runner collects (rev 3, R3-5).** The kit travels whole into every adopter at `ci/kit/`, so no file under `kit/` has a name a test runner or compiler collects by default from a repository's top (AC-KI-38): fixture sources carry `.fixture` after their real name, and the helpers that copy them into a scratch repository strip it.

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

(Rev 3, R3-8: as it happened, §12 question 6 put the fence's depth fix alone into `v1.0.0-rc.2`, and `v1.0.0` was tagged on rc.2's tree, `9e282b4f`. `kit-init` stayed out of 1.0 as ruled here and ships as `v1.1.0-rc.1`.)

Later moves of the workflow template's action shas, or of the default Postgres major, are PATCH releases: they change generated output for new adopters and no interface.

### 9.2 The CHANGELOG entry

As written at revision 3 (R3-7), equal to the entry in `CHANGELOG.md`; the release pull request writes the two placeholders and the date.

```markdown
## v1.1.0-rc.1 (unreleased)

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
- Unchanged: the manifest format (`manifest v1`), the adapter contract, the key
  formula, the plan's sentences, `check-host`'s file list and allow-list; every
  check but those two.
- The proofs: the K-init family in `kit/tests/run` (every supported shape
  generated offline and held to the lint and both fences), two shapes end to
  end in `kit/tests/image` (Go, Node and Postgres; Python with pytest), and
  planted defects in `kit/tests/plants`, each seen red.
- Proven on: linux/amd64, the kit's own CI run on the release pull request, and
  linux/arm64, a workstation run of both end-to-end shapes at the same tree;
  both are recorded in that pull request: `<written by the release pull request>`.
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
- **AC-KI-9** (rev 2, R-1 a) A write failure planted part way through the writes leaves every file the run created removed and every file it appended to byte-equal to its original, a Makefile with CRLF line endings among them, and the run exits 2 with its sentence.

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
- **AC-KI-28** Each planted defect of §7.3 turns K-init red, and only K-init. (Rev 3, R3-1: "only K-init" holds by construction for the plants run by `tests/run`, since no other family reads `bin/kit-init` or `templates/` as code; the plant of the Python image block turns `initpy` red, K-init's end-to-end half.)
- **AC-KI-29** The end-to-end family in `kit/tests/image` generates the Go, Node and Postgres shape, `local-ci run` passes each job, and `check-adapter` passes each stack job.
- **AC-KI-30** K-init covers at least these shapes: Go alone; Node with vitest alone; Go, Node and Postgres (the example's); Python with pytest alone; Node without vitest (a stub); two Go directories with one toolchain; no stack at all; a stack directory of `.`. (Rev 2, R-2: the last three are green only once piece 1b has landed; on the rc.1 checks they fail at `check-escaping-reads:375-377` and `check-manifest:228`.)
- **AC-KI-31** The CHANGELOG entry has §9.2's shape and the release is MINOR.

**Added by revision 2**

- **AC-KI-32** `check-manifest` rule 6 finds nothing for an `escaping-tree` or `escaping-scan` path of `.`, and still finds a path absent at the commit; proven red once in the manifest family.
- **AC-KI-33** `check-escaping-reads`, given a manifest with no `escaping-scan` line, prints `The manifest names no escaping-scan line, so no file was scanned and only the escaping-read lines were checked against the keys.`, exits 0 when every `escaping-read` line is keyed with `OK: no file was scanned; <n> escaping-read lines were checked, and every one is keyed.`, and exits 1 when one is not; given an `escaping-scan` root holding no test file, it still exits 2 with `the escaping-read fence found no test files, so it would prove nothing.`; proven red once in the escaping family.
- **AC-KI-34** Under `escaping-tree <job> .`, the fence counts a file's depth as it would the same relative position under a named tree: `db/x_test.go` calling `filepath.Join("..", "..", "x")` is a finding; proven red once in the escaping family.
- **AC-KI-35** A file `kit-init` reads that does not decode as UTF-8 is refused with exit 2 and `<path> is not UTF-8 text kit-init can read, so nothing was written.`, and nothing is written. (Rev 3, R3-2: `ci/KIT-VERSION` among them.)
- **AC-KI-36** The end-to-end family in `kit/tests/image` generates the Python with pytest shape against a recorded python-build-standalone release and its real checksums, and `local-ci run` passes the job with the pinned interpreter answering (not the base image's own `python3`), `pip install --require-hashes` succeeding inside the job, and a counts line whose `pytest_tests` equals the fixture's passing tests; `check-adapter` then passes the job. It runs in the kit's own workflow (amd64), and the release pull request records one green run of AC-KI-29 and AC-KI-36 on an arm64 workstation.

**Added by revision 3**

- **AC-KI-37** (R3-3) Under a tree of `.`, `check-escaping-reads` accepts a keyed `escaping-read` line, scans a TypeScript file under the `test/` directory of a `ts` scan root of `.`, and does not call a tree of `.` "declared by hand" when a scan root lies inside it; each proven red once in the escaping family, and K-init's `.` shape carries one declared, keyed read.
- **AC-KI-38** (R3-5) No file under `kit/` ends in `.go`, `.ts`, `.tsx`, `.mts`, `.cts`, `.js`, `.jsx`, `.mjs` or `.cjs`, or is named `test_*.py`, `*_test.py` or `conftest.py`; a K-init case holds this over the kit under test and is red once on a planted file, and every K-init, `initgo` and `initpy` case passes on the renamed fixtures.
- **AC-KI-39** (R3-4) For a Go or Node directory with no test file, `kit-init` writes the `escaping-scan` line, exits 1, and prints the step of §6 naming that directory before `make check`; proven red once in K-init.

## 11. Build plan

Four pieces of work, in order (rev 2: piece 1 is built and ratified; piece 1b is added by R-1 and R-2; piece 2's brief is rewritten by R-7. Rev 3: pieces 1b, 2 and 3 are done and ratified; piece 3b is added by R3-2 to R3-5 and is the last before the tag). Each names who builds it, the files it may touch, the criteria it meets and an estimate in agent tokens from the measured medians of builds of this size.

**1. The generator and its templates.** A build engineer. Files: `kit/bin/kit-init` (new), `kit/templates/**` (new), `kit/README.md` (§1 table row, §2.2 sentence, prerequisites, §7 gaining the generator route with steps 4 to 6 folded into it), `CHANGELOG.md` (the unreleased entry). Meets AC-KI-1 to AC-KI-3, AC-KI-5, AC-KI-7, AC-KI-8, AC-KI-10 to AC-KI-19, AC-KI-21 to AC-KI-26, AC-KI-31, each by the build's own run on the example's tree and on one scratch repository per stack. Estimate about 500k (range 400k to 600k): roughly 700 to 900 lines of Python and 600 lines of templates, three stacks, the lookups and the offline path.

**1b. The corrections and the two check changes (rev 2).** A build engineer, against this document's Revision 2; no new design is needed, and a question that arises mid-way stops the piece and comes back here. Files, and nothing else:

- `kit/bin/check-manifest`: line 228 only (R-2 item 1);
- `kit/bin/check-escaping-reads`: line 228 (R-2 item 3), lines 375-377 and the closing `OK:` line at 419 (R-2 item 2), and the exit-code paragraph of its docstring (lines 55-58) to say that no `escaping-scan` line is a pass;
- `kit/bin/kit-init`: `read_text` (lines 257-262) and `write_all` (lines 738-784) only (R-1 a and b);
- `kit/tests/fam/manifest.sh` and `kit/tests/fam/escaping.sh`: the red-once cases of AC-KI-32, AC-KI-33 and AC-KI-34, each seen failing on the unchanged line before the change;
- `kit/tests/plants`: three plants, each turning its family red: the `.` arm removed from `check-manifest:228`; the no-scan branch restored to exit 2; the depth under `.` restored to `rel[len(tree) + 1:]`;
- `kit/README.md`: line 189 gains R-2's one sentence;
- `CHANGELOG.md`: the `Changed:` line of §9.2.

Meets AC-KI-32 to AC-KI-34 with their proofs, and AC-KI-9's and AC-KI-35's code (their proofs are piece 2's). The harness count rises from 788 by the new cases, and every existing case stays green. Estimate about 150k (range 120k to 200k): three one-line check changes, two generator functions, six or seven cases and three plants.

**2. The proofs.** A test engineer, tests only, after piece 1b has landed (AC-KI-30's last three shapes are green only then). Files, and nothing else: `kit/tests/fam/init.sh` (new, the K-init family); its entry in `kit/tests/run`'s family list (`kit/tests/run:26`); `kit/tests/fixture/init/**` (new); two families in `kit/tests/image`; five plants in `kit/tests/plants`; one step in the `guards` job of the kit's `.github/workflows/ci.yml`.

The fixture: a minimal source tree for each shape of AC-KI-30 (a Go module with one test, a Node package with vitest, a lockfile and one test, a Python directory with `.python-version` and a hashed `requirements.txt` naming pytest and its dependencies, a Node package without vitest); the example's six files at `8ad9623` (read with `git show`) as the golden output; one checksums file holding the example's five values and a recorded python-build-standalone release's two checksums; a Makefile with CRLF line endings; a file holding a byte that is not UTF-8.

What it owns, each seen red once:

- **AC-KI-4**: one case per refusal row of §2.3, AC-KI-35's included, each seeing exit 2, its sentence, and the working tree, the index and HEAD unchanged (the porcelain status, `git write-tree` and `git rev-parse HEAD` equal before and after).
- **AC-KI-6**: the whole family runs with `--checksums`, the existing `docker` shim (`kit/tests/fixture/shims/docker`), the `curl` shim unreachable, and `https_proxy` and `HTTPS_PROXY` set to a closed local port, so any lookup that escaped the offline path would fail; it passes.
- **AC-KI-9**: a write failure planted part way (a later output's directory made read-only), with the CRLF Makefile as an append target: every created file gone, the Makefile `cmp`-equal to its original, exit 2 with its sentence.
- **AC-KI-20**: the golden fixture committed, and K-init comparing its rottnest output with it, comment and blank lines set aside.
- **AC-KI-27**: the guard step, red once on a planted copy of a body line of `kit/templates/agents/section.md` in `kit/README.md`, green when the documents cite the path; it passes while `docs/adopt-with-an-agent.md` does not yet exist.
- **AC-KI-28**: the five plants (the existing-file refusal removed; one tool's pin cross-check removed from its `check` part; the shim read from a path other than `ci/kit/hooks/pre-push.shim`; `check_tokens` disabled so an unreplaced token passes; the restore truncating by decoded length), each turning K-init red, and every other family green against the same planted copy.
- **AC-KI-29**: the Go, Node and Postgres shape end to end, as §7.3 says.
- **AC-KI-30**: all eight shapes, each held to the generator's exit 0, `check-manifest` at HEAD (rules 1 to 12), `check-workflow`, `check-escaping-reads`, and `bash ci/ci-job.sh no-such-job` exiting 2 with no counts line; the stub shape's arm exiting 2 with its sentence and no counts line.
- **AC-KI-35**: its K-init case.
- **AC-KI-36**: the Python with pytest shape end to end, as written in §10.
- **§8's two rules** (AC-KI-2, AC-KI-3) as committed syntax-tree tests in K-init, which §8 requires and the build proved only by its own run.

Estimate about 260k (range 220k to 320k): the 220k first planned, plus the Python end-to-end shape, AC-KI-9's planted failure and AC-KI-35's case.

**3. The documents.** A documentation pass. Files: `docs/adopt-with-an-agent.md` (new; it cites the agent template, never copies it), the root `README.md`'s quickstart pointer to `kit-init`, `docs/kit-init.md` amended to what was built. Meets AC-KI-26 and AC-KI-27's passing half. Estimate about 100k. (Rev 3: done with revision 3, R3-10.)

**3b. The four defects (rev 3).** A build engineer, against Revision 3; no new design is needed, and a question that arises mid-way stops the piece and comes back here. It writes its own red-once cases and plants, as piece 1b did. Files, and nothing else:

- `kit/bin/kit-init`: `preflight_repo` (lines 830-838) reads the tag and returns it, and the late read at 1009-1010 goes (R3-2); `main`'s check loop (1030-1038) keeps each check's exit code, and `closing` (1046 onward) prints R3-4's step;
- `kit/bin/check-escaping-reads`: the helper, used at 295, 375 and 407, and the location word at 408 (R3-3);
- `kit/tests/fixture/init/**`: every file AC-KI-38 names renamed with `.fixture` appended, contents unchanged (R3-5);
- `kit/tests/fam/init.sh`: `init_put` (line 47) and every other `$KI_FIX` copy strip the suffix; the cases of R3-2, R3-4 (AC-KI-39) and AC-KI-38; the `.` shape's declared read (AC-KI-37);
- `kit/tests/image`: the `e2e` copies in `fam_initgo` and `fam_initpy` strip the suffix;
- `kit/tests/fam/escaping.sh`: AC-KI-37's three cases;
- `kit/tests/plants`: `:407` restored, red in the escaping family; the AC-KI-38 walk disabled, red in K-init.

It does not touch `CHANGELOG.md`, `kit/README.md` or `docs/`, whose lines for it are written at revision 3; the release pull request reads them against what landed. Then: `bash kit/tests/run` under bash 3.2 and a current bash, `python3 kit/tests/plants` whole, `python3 kit/bin/check-host kit/`, and `bash kit/tests/image` on this arm64 workstation, each exit code pasted. Meets AC-KI-35 for `ci/KIT-VERSION`, AC-KI-37, AC-KI-38 and AC-KI-39. Estimate about 250k (range 200k to 320k): two small generator edits, one fence helper at three sites, about fifteen fixture renames with their copy helpers in two harnesses, about eight cases and two plants, and the full re-runs, `tests/image` cold among them because the fixtures move.

Total first planned about 820k, range 700k to 1.0M; from here, pieces 1b, 2 and 3 are about 510k (range 420k to 620k). `v1.1.0-rc.1` waits for all three, AC-KI-36 included (rev 2, R-5), and for piece 3b (rev 3, R3-3 and R3-5). A security read is worth asking for once the first piece lands, scoped to the generator's network lookups, its subprocess set and the signer handling.

## 12. Open questions

Each has the answer this specification takes when nobody rules otherwise.

1. **The Python toolchain's source.** (Rev 2: built on the default and ratified; the path is proven end to end before the tag, R-5.) CPython publishes no Linux binary tarball. Accept python-build-standalone's builds, verified against that project's per-release `SHA256SUMS` (§4.1)? Or leave Python's image block as a stub the person fills in, with only the arm and the manifest lines generated? **Default: accept python-build-standalone**, named in the generated Dockerfile's comment so the reviewer sees the source.
2. **The signer in the landing.** `--signer` could write the signer's line straight into `ci/allowed-signers`, saving the second full CI run DESIGN §9 step 4 costs. That puts the adoption's one trust decision inside a large generated pull request. **Default: never in the landing**; the line is printed and kept for its own pull request (§3.8).
3. **The release order.** Tag `v1.0.0` on rc.1's tree first and ship this as `v1.1.0-rc.1`, or fold it into a `v1.0.0-rc.2`? **Default: `v1.0.0` first, then `v1.1.0-rc.1`** (§9.1).
4. **The default weekly minute.** Every generated project would run at `20 6 * * 1` unless it passes `--cron`. A minute derived from the project name would spread the load on the shared runners and keep each project stable. **Default: the example's `20 6 * * 1`**, because a value the person can read in the example is easier to reason about than a derived one; revisit if adopters report queueing.
5. **The agent files by default.** Write both `AGENTS.md` and `CLAUDE.md` when neither exists, or only append to those already present? **Default: write both**, because a repository without them is the one an agent most needs told, and `--no-agent-files` opts out.
6. **The fence's count at the top of a tree (found at revision 2; CLOSED at revision 3, R3-8: answered "rc.2 first", shipped as `v1.0.0-rc.2`, and `v1.0.0` tagged on that tree).** `check-escaping-reads:228` counts the directories between a test file and its tree as `len(os.path.dirname(<path inside the tree>).split("/"))`, and in Python `"".split("/")` is `[""]`, so a file directly inside its tree is counted one level below it. Measured on the rc.1 fence: `api/x_test.go` calling `filepath.Join("..", "web", "x")` reads outside `api` and is not flagged; with `filepath.Join("..", "..", "web", "x")` it is flagged, reported as "a path call climbs 2 `..` from 1 below api/". The same line is in the reference implementation the fence was lifted from. The fix is one line (a file directly inside its tree counts 0) with one red-once case. It makes the fence stricter, so a project green today can turn red on upgrade, and it sits in the trust code that §9.1 keeps unchanged for `v1.0.0`. Fix it in a `v1.0.0-rc.2` carrying only that line and its case, then tag `v1.0.0` on rc.2's tree? Or tag `v1.0.0` on rc.1's tree and fix it in `v1.1.0-rc.1`, with an adopter's action in the CHANGELOG? **Default proposed: rc.2 first.** §9.1 keeps 1.0 unchanged so that candidate adopters need not re-test the trust code for an unrelated change, and this change is to the trust code itself; a final release should not knowingly carry a gap in a soundness fence. Either answer composes with R-2's depth change for `.`, which keeps the existing count.

One inconsistency found while writing this, for the documentation pass: the cache-runtime action's trailing comment reads `# v3` in `kit/README.md:397` and `# v4.0.0` in `example:.github/workflows/ci.yml:123,172,213` for the same sha `04d248b8`. The template follows the example's file, since the repository's file is the one that runs (`kit/README.md:278`); the README line should be corrected to whichever release that sha is. (Rev 3, R3-9: it is `v4.0.0`; `kit/README.md` now says so at both of its lines.)
