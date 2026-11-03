---
name: quokka-adopt
description: Use when the person asks to adopt quokka-ci-kit in a repository (vendor the kit, write its manifest, verdict script, job image, workflow and hook, and get the lints and make check green), or to add, change or debug a CI job in a repository that already has the kit under ci/kit/. It fetches the kit's canonical agent instructions and follows them, so the steps and rules live in one place.
---

# Adopting quokka-ci-kit

Fetch `https://raw.githubusercontent.com/aames/quokka-ci-kit/main/docs/adopt-with-an-agent.md` and follow it top to bottom, including its hard rules, its questions to the person and its report. If the repository already has `ci/kit/` and the person wants a job added or changed, follow its section "If the kit is already vendored: adding a job".

If the fetch fails, tell the person and ask them to paste that file. Until you have it, these three rules hold:

- Do not push anything unless the person tells you to in this conversation; never force anything.
- Never create, read or change a signing key, and never run `ci/kit/bin/local-ci sign` or `push`.
- Never edit anything under `ci/kit/`, and never put a secret in `ci/manifest` or the verdict script.
