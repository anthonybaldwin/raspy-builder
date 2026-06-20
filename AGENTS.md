# AGENTS.md

Guidance for any agent working in this repo. (Claude reads this too — see
[CLAUDE.md](CLAUDE.md).)

## What this repo is

`raspy-builder` builds upstream projects that ship a Dockerfile but publish no
container image, and pushes the resulting images to GHCR (arm64) so they can be
digest-watched and updated like any normal image. It produces images; it is not
an application itself.

## Layout

- `vendor/<name>/` — each target upstream, vendored as a **git submodule**. Do
  **not** edit code under `vendor/`; it's upstream's and would be lost on the next
  bump. Fixes belong in the upstream project.
- `targets.json` — the build manifest: one object per image,
  `{ "name", "context", "dockerfile" }`. `name` becomes `ghcr.io/<owner>/<name>`.
- `.github/workflows/build.yml` — builds every target on a native arm64 GitHub
  runner and pushes to GHCR. Triggers: push to `main`, manual dispatch, and a
  weekly schedule that first bumps submodules to upstream HEAD.

## Common tasks

Add a target:

```bash
git submodule add <url> vendor/<name>
# add { "name", "context", "dockerfile" } to targets.json
git commit -am "add <name>" && git push
```

After its first build, set the new GHCR package to **Public** (GitHub → Packages
→ settings) so consumers pull without auth.

Bump a pinned upstream manually:

```bash
git submodule update --remote --recursive
git commit -am "bump" && git push
```

## Constraints

- **arm64 only.** The workflow builds `linux/arm64` on a native arm64 runner.
  Other arches would need a native runner per arch (or slow QEMU) — out of scope.
- **No secrets in this repo.** It holds only submodule pointers + the workflow;
  CI authenticates to GHCR with the automatic `GITHUB_TOKEN`. Keep it that way —
  it's a public repo.
- Targets must build cleanly from their own Dockerfile with no extra steps. A
  project needing bespoke build logic doesn't belong here.
- Don't add a `pull_request`/`pull_request_target` trigger — the build runs only
  on owner-controlled events (push/dispatch/schedule) by design, so untrusted PR
  code never executes with the repo token.
