# The quokka-adopt skill for Claude Code

Copy this `quokka-adopt` directory into your repository's `.claude/skills/`, or into `~/.claude/skills/` to have it available in every repository.
Then ask Claude Code to adopt quokka-ci-kit, or to add a CI job. The skill fetches [docs/adopt-with-an-agent.md](../../../docs/adopt-with-an-agent.md) and follows it, running `kit-init` to write the files.
For Codex or another agent, paste that file into the session instead, or point your `AGENTS.md` at its URL.

```text
        .-"""-.
       / .===. \
       \/ ^ ^ \/       "brought the spanners,
       ( \_u_/ )        pinned to the version"
     __|_[===]_|__
    |__toolbox____|
```
