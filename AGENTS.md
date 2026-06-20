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
- `.github/workflows/build.yml` — builds targets on a native arm64 GitHub runner
  and pushes to GHCR. A target builds **only when its submodule moves**, never on
  plain code/doc commits. Two jobs: `watch` decides what to build (and bumps moved
  submodules on a check), `build` runs the matrix. Triggers: twice-daily schedule
  (check upstreams, build the moved ones); `workflow_dispatch` with `target`
  (all/one) + `force` (rebuild a current pin); and `push` scoped to
  `vendor/**`/`.gitmodules`/`targets.json` (builds the targets changed in the push).

## Common tasks

Add a target:

```bash
git submodule add <url> vendor/<name>
# add { "name", "context", "dockerfile" } to targets.json
git commit -am "add <name>" && git push   # builds the new target
```

Packages publish **public** (inherited from this public repo), so consumers pull
without auth. If a new one ever shows up private, flip it once in GitHub → Packages.

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
