# Security read of kit-init, v1.1.0-rc.1 (KI-s, 2026-11-03)

The read was made at commit `dbeebc2`, and every line number below is that tree's. Its six fixes (findings 1 to 6) are commit `0a1d328`; their rulings, the criteria AC-KI-40 to AC-KI-45 and the disposition of the six notes are [docs/kit-init.md](../kit-init.md), Revision 4. This file is the read as it was reported, kept as the kit's record; it is not amended when the code moves.

Verdict: GO WITH CONDITIONS. Nothing critical or high. One medium: kit-init follows symlinks when it writes, so it can write outside the repository. Fix that before tagging.

The most important recommendation: before write_all, refuse any target path, or any parent of one, that is a symlink or whose realpath falls outside the top of the working copy. Then append with O_NOFOLLOW.

Method: I read the code and ran it. kit-init ran in two throwaway repositories in a temporary directory, with the kit copied in as ci/kit and https_proxy pointed at a dead port. No file in the worktree was edited.

## Findings, most severe first

1. MEDIUM, confirmed by a run. kit-init writes outside the repository through a committed symlink, and the commit does not show it.
   - Where: kit-init:771 appends with `open(path, "ab")`, which follows a symlink. kit-init:257-269 (read_text) reads through it. kit-init:755-760 creates files under a parent directory that is a symlink. The targets come from 855-861 (AGENTS.md, CLAUDE.md), 972 (Makefile) and 980 (.gitignore).
   - What the run showed: a repository whose CLAUDE.md and Makefile were committed as links to `../outside/*` got the kit section and the check target appended to the outside files. kit-init printed "appended to ...", then "committed ...", and exited on the lints alone. The commit held neither file, so a reviewer sees nothing.
   - For a new output under a symlinked parent (.github, .githooks, ci/image), the file is written outside the repository first. `git add` then fails ("beyond a symbolic link"), so that case at least gets the message "commit failed".
   - Why medium: the appended text is fixed kit text, and every value in it is bounded to NAME shape. So I found no path to code execution. But anyone who can commit to the repository can make the maintainer's run corrupt any file the maintainer can write. Appending to ~/.gitconfig or ~/.ssh/config breaks it. Appending to a shell rc file runs the section's backticked `make check` at every shell start. The write is invisible to git.
   - O_EXCL at :763 does protect the final component of a new file. Parent directories and appended files are not protected.

2. LOW, confirmed by a run. Directory names reach generated shell and the workflow without quoting.
   - The only bound: path_problem (kit-init:272-279) and the manifest's ispath (lib/keys.sh:85-89) refuse white space, `* ? [ ] \ :`, a leading `-`, absolute paths and `.`/`..` segments. Both allow `$ ( ) ; & | ' " \` # < > { }`.
   - DIR and PIN_FILE land unquoted in adapter/go.sh:4,8, node-vitest.sh:4,8 and python-pytest.sh:4,8,12 (`cd @@DIR@@`, `awk ... @@PIN_FILE@@`, `< @@PIN_FILE@@`).
   - They also land inside the double-quoted echo of a RUN line that runs as root (image/go.Dockerfile:9, node:9, python:14, evaluated only when the pin check fails), in COPY (image/run.Dockerfile:5), and in `hashFiles('...')` (workflow/cache-gomod.yml:5, cache-npm.yml:5).
   - What the run showed: `--go 'api$(touch${IFS}PWNED):api'` passed check-manifest. It wrote `cd api$(touch${IFS}PWNED)` into ci/ci-job.sh:64, and that substitution runs at `make check` inside the job container.
   - An agent can reach this. On a crafted directory name, the automatic job name fails NAME_RE, and the refusal at kit-init:389 says "give the directory a name with DIR:JOB". An agent retries with exactly that.
   - Why low: the substitution runs where the repository's own tests already run (the job container, or a root build step of the same image). Nothing is gained beyond that, but the generated files are not what the person meant.
   - Fix: limit DIR to `[A-Za-z0-9._/-]` in path_problem, or quote every path token in the shell templates.

3. LOW, confirmed by a run. kit-init executes a `yaml.py` or `yaml/` at the top of the repository.
   - Where: kit-init:1028 runs `sys.executable -c "import yaml"` from the repository top. With `-c`, sys.path[0] is the current directory.
   - What the run showed: a planted yaml.py was imported (`__pycache__/yaml.cpython-315.pyc` appeared). Its output is hidden by quiet=True.
   - Why low: ci/kit/bin/* from the same tree also runs. But the vendored kit is verified and a top-level yaml.py is not. The adopt guide also tells people no repository code runs before make check.
   - Fix: run `-I -c` (and `-I` on the two python fence calls).

4. LOW, confirmed by a run. kit-init exits 0 when check-workflow did not run.
   - Where: kit-init:1034-1036 skips check-workflow without PyYAML and leaves rc alone.
   - This contradicts the docstring (kit-init:26, "every lint held") and docs/adopt-with-an-agent.md:82 ("Exit 0 means every check held").
   - Both runs printed "NOT RUN check-workflow" and still exited 0 on lints otherwise green.
   - Why low: step 4 of the guide reruns the fence and CI runs it too. But an agent reading the exit code alone is told something false.

5. LOW, confirmed by reading. Exit 1 does not always mean committed.
   - Where: docs/adopt-with-an-agent.md:82 says "Exit 1 means it wrote and committed the files".
   - But kit-init:1017-1019 also returns 1 when the commit failed and nothing is committed.
   - Why low: step 3's done-when check (`git status` shows nothing) catches it.

6. LOW, needs verification. The main branch is guessed, and a detached HEAD is accepted.
   - Where: kit-init:936-941 reads main from refs/remotes/origin/HEAD and falls back to "main" when it is missing. It refuses only `branch == main_branch`, so a detached HEAD commits.
   - If the guess is wrong, a protected `master` or `trunk` gets `main-branch main`. CI then reads its trust inputs (ci/allowed-signers) from origin/main, which anyone with push could create.
   - Why low: it is loud in practice, because no pull request to the real main would trigger CI. Modern git also writes origin/HEAD on fetch.
   - Fix: a `--main` flag checked against the answer to the guide's question 1, and refuse a detached HEAD.

7. NOTE, confirmed by reading. The python-build-standalone tag from the network is not validated.
   - Where: kit-init:523 takes tag_name from the GitHub API unchecked, and it reaches image/python.Dockerfile:9 and 20. The --checksums path requires `\d+` (kit-init:459).
   - It is bounded by git refname rules (no white space or control characters) and by the requirement that it match the asset and SHA256SUMS names. As an ARG value it cannot inject Dockerfile instructions.
   - Fix: check `^\d{8}$` to match the offline path.

8. NOTE, confirmed by reading. Checksums are trusted on TLS alone.
   - Where: kit-init:499-533.
   - Go's sha256 comes from the go.dev JSON index. Node's comes from SHASUMS256.txt with no check of its .sig. Python's comes from the SHA256SUMS of the same release as the archive, so a compromised release defeats it. GitHub's asset `digest` field is not cross-checked. urlopen follows redirects, including to http, and reads with no size cap.
   - Nothing fetched is executed. Values are checked against 64 hex (kit-init:439, 510), and each is printed with its source (954-957).
   - Fix: cross-check the GitHub asset digest for python-build-standalone, and say in docs/kit-init.md that these values are trust on first use.

9. NOTE, confirmed by reading. The prune job is outside the workflow fence.
   - Where: workflow/prune.yml:5-31.
   - It holds contents: write and puts GH_TOKEN into git's extraheader. It is not a manifest job, so W-8, W-11 and W-12 never look at it (check-workflow:639, 657 iterate scheme_names).
   - As generated it is correct: actions pinned by sha, main only through the plan's `main` output, kit code at main's sha.
   - But a later edit (an unpinned action, a wider `if:`) passes the fence unseen.

10. NOTE, needs verification. `pip --require-hashes` may not hash-check build dependencies.
    - Where: adapter/python-pytest.sh:12.
    - As I understand pip, the build dependencies of an sdist, installed in an isolated build environment, are not hash-checked.
    - Fix: consider `--only-binary :all:` (or `--no-build-isolation`).

11. NOTE, confirmed by reading. Small hygiene points.
    - image/go.Dockerfile:17 and python.Dockerfile:22 extract as root without `--no-same-owner` (node has it at :18). This is after the sha256 check, so it is ownership hygiene only.
    - curl `-L` has no `--proto-redir =https`.
    - The signer principal is git's user.email, unvalidated (kit-init:1001). It is the person's own value and goes only to `.next`.
    - agent_files_only raises a traceback, not a refusal, if keys dump lacks a directive (kit-init:879).

12. NOTE, a deliberate ruling (docs/kit-init.md R3-3, AC-KI-33). The fence passes a manifest with no escaping-scan line.
    - Where: check-escaping-reads:385, 431.
    - A Go or TS manifest whose escaping-scan lines are deleted now passes with a printed sentence, where before it exited 2.
    - The change is visible in the manifest diff, and it forces every job to re-run. A tighter form would refuse "no escaping-scan line" while any job pins go or node.

## The questions

Q1, inputs:
- No input can execute anything in kit-init itself, other than findings 2 and 3.
- No input can break a `@@TOKEN@@` substitution. render() (kit-init:215-228) substitutes one pass from a closed token set. A token in a value is not re-expanded, and check_tokens (890-897) refuses any token left over.
- Where each value reaches a generated file, and its bound:
  - project: PROJECT_RE (917).
  - cron: CRON_WORD_RE inside YAML double quotes (921). No quote or backslash can pass.
  - job: NAME_RE, at most 32 characters (384).
  - pins: exact-version regexes (309, 330, 358).
  - postgres major: `[0-9]+` (397).
  - sha256 values and digests: 64 hex.
  - requirements file name: a regex at 363, with only the name used.
  - main: the manifest NAME shape, enforced by the check-manifest run on the scratch manifest before any write (988-991).
  - kit_tag: only the -m argument value of git commit.
  - DIR, PIN_FILE, LOCKFILE, SCAN_DIR: only path_problem and ispath (finding 2).
  - package.json: parsed by json.loads; only its dependency keys are read.
  - An existing Makefile or AGENTS.md: only searched for a marker, then appended to (finding 1).
- Writes outside the repository: yes, through symlinks (finding 1).

Q2, network:
- Three HTTPS GETs with a fixed User-Agent and a 30 s timeout:
  - go.dev/dl JSON
  - nodejs.org SHASUMS256.txt
  - the api.github.com releases page, plus the release's SHA256SUMS (which must be https)
- Plus `docker buildx imagetools inspect postgres:N` when --postgres is given.
- Every failure is one refusal per value, naming its URL and pointing to --checksums. A mismatch or a missing line refuses, and nothing is written.
- Nothing fetched is executed; the values are trusted on TLS (finding 8).
- --checksums makes no network call, and docker is not called for digests (lookup_values returns at kit-init:478). Confirmed by a run with https_proxy at a dead port: exit 0, values from the file.

Q3, subprocesses:
- The closed set holds: git, docker, ssh-keygen, bash and sys.executable, each called as a list (kit-init:102-128), never through a shell.
- No user string can become an option:
  - git add and git commit put `--` before paths (1013-1014).
  - ssh-keygen gets an absolute path in the git directory.
  - bash gets fixed script names, plus `--file` with an absolute path.
  - docker gets `postgres:<digits>`.
- Output handling: docker and ssh-keygen output is regex-parsed. keys dump is split on tabs. Lint output passes through to the terminal. check-manifest's refusal output is echoed.
- Exception: finding 3.

Q4, signer:
- `--signer auto` requires gpg.format ssh. It reads either a `key::` literal or a file ending in `.pub`, using `<signingkey>.pub` when that exists. It never opens a non-.pub path (kit-init:550-568). Confirmed: with a private signingkey and no `.pub` beside it, it refuses without reading.
- `--signer FILE` reads the named file whole. A private key is refused as "no single SSH public key line", and only the path is printed (confirmed with a throwaway key).
- What is written: only `<email> namespaces="git" <type> <base64>`, into `<git common dir>/<project>-ci/init/allowed-signers.next` (kit-init:1043), and only after the checks. The scratch signer.pub is removed at close (813-825).
- The landing commit's ci/allowed-signers is the template's single comment line (724). .git is never added.
- No path copies a private key.

Q5, workflow:
- Permissions: workflow-level contents: read. checks has contents: write and checks: read. Each local job, kit and prune have contents: write.
- Every action is pinned by a 40-hex sha, the same three as the golden example f161e85: checkout 11d5960a, cache 0057852b, ghaction-github-runtime 04d248b8. Every checkout has persist-credentials: false.
- No pull_request_target or workflow_run trigger.
- The token is in an env only at the plan step (GH_TOKEN), at each record step (GITHUB_TOKEN), and in prune.
- Where the fence does not look: the prune job (finding 9). kit-init's own run can skip the fence and still exit 0 (finding 4).

Q6, Dockerfile:
- Runs as root: COPY of the pins, then one RUN that does the pin cross-check, dpkg architecture, then per tool curl, then `sha256sum -c`, then tar. It finishes with rm and install -d, then `USER ci`.
- Verify comes before extract, under `set -eu` with the pipefail SHELL.
- The toolchain layer pulls nothing unpinned: FROM is the locally built base, and Postgres is pinned by digest.
- pip runs at verdict time, not at build, with --require-hashes (finding 10). GOTOOLCHAIN=local stops toolchain downloads.

Q7, fence:
- within() widens acceptance only for the path equal to the tree itself, and for the tree `.`. Both are correct.
- The depth fix at check-escaping-reads:230 removes a miscount under `.`. Before, depth was computed short, which was stricter.
- The test/ detection at 302 now finds top-level test/ under a scan root of `.`, which narrows what slips past.
- The real widening is the ruled no-scan-line pass (finding 12). I found no other path shape that newly passes.

Q8, agent documents:
- Hard rule 3 and question 5 match the auto path: public half only, nothing committed, and the file location (adopt-with-an-agent.md:13, 39, 154) is right.
- "Commit on a branch" holds except for a detached HEAD and a wrong guess of main (finding 6).
- The exit-code text is wrong in two places (findings 4 and 5).
- Step 1 runs check-prereqs and kit-update from an unverified clone of main before the signature check (adopt-with-an-agent.md:51, 60). That is the trust bootstrap the earlier reviews covered; I note it but do not count it here.
- SKILL.md's fallback rules are stricter than the guide's, which is safe.

## Properties checked and found holding
- Nothing is written before every refusal is collected. Templates render in memory, unreplaced tokens are refused, and the scratch manifest is linted before the first write.
- A failed write removes the files it created and truncates appended files to their on-disk byte length (write_all 773-792).
- Existing outputs are refused with lexists, so a dangling symlink at an output path is refused. O_EXCL guards the final component.
- No value can create a new token, and render makes one pass.
- Network values are 64 hex. The offline path really is offline.
- The subprocess set is closed and shell-free, with `--` before paths.
- No private key is ever written, copied or printed. allowed-signers.next lives in the git directory and is never committed.
- The generated workflow has least-privilege permissions per job, actions pinned by sha, no dangerous triggers, and persist-credentials false.
- Downloads are verified before extraction, then the user drops to ci.

## Counts
critical 0, high 0, medium 1, low 5, note 6. 9 confirmed (7 by a run, 2 by reading); 2 need verification (findings 6, 10); 1 is a deliberate ruling (finding 12).

The findings concern the generator only. I did not re-review the launcher, plan, records or trust model.
