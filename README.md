# ghcr-builder

A generic build/release pipeline for upstream projects that **ship a Dockerfile
but publish no prebuilt image** — so there's no registry tag to watch and any
local build has to be rebuilt by hand to stay current.

Each target is vendored here as a **git submodule** under `vendor/` and listed
in [`targets.json`](targets.json). A GitHub Actions workflow builds each one and
pushes it to GHCR, turning a "build-it-yourself" tool into a normal registry
image.

## Why

So any Docker host can manage these like a normal upstream image: a digest
watcher such as [Diun](https://crazymax.dev/diun/) notices the `:latest` digest
change and your update tooling pulls it — updating becomes routine, and the
build runs here in CI instead of on the target host.

## Current targets

| Image | Upstream | Notes |
|-------|----------|-------|
| `claude-code-trace` | [`delexw/claude-code-trace`](https://github.com/delexw/claude-code-trace) | Claude Code session-log viewer (web, port 1421) |
| `codex-trace` | [`PixelPaw-Labs/codex-trace`](https://github.com/PixelPaw-Labs/codex-trace) | Codex session-log viewer (web, port 1422) |

## How it works

`.github/workflows/build.yml`:

- **Weekly schedule** → `git submodule update --remote` fast-forwards every
  submodule to its upstream default-branch HEAD, commits the bump, then builds.
- **push / manual dispatch** → builds the currently-pinned submodule commits.
- A dynamic matrix is generated from `targets.json`, one parallel leg per target.
- Builds on GitHub's **native arm64 runners** (`ubuntu-24.04-arm`) — no QEMU.
- Pushes `ghcr.io/<owner>/<name>:latest` plus a `:<upstream-short-sha>` tag.

No PAT or repo secret is required — the workflow's automatic `GITHUB_TOKEN` has
`packages:write`.

## Adding a target

```bash
git submodule add <upstream-repo-url> vendor/<name>
# then add an entry to targets.json:
#   {
#     "name":       "<image-name>",        # → ghcr.io/<owner>/<image-name>
#     "context":    "vendor/<name>",       # docker build context
#     "dockerfile": "Dockerfile",          # relative to context
#     "platforms":  "linux/arm64",         # add linux/amd64 if you want multi-arch
#     "build_args": ""                     # newline-separated KEY=val, or ""
#   }
git commit -am "add <name> target"
git push      # triggers a build
```

No workflow edits needed. After the first build of a new image, set its GHCR
package visibility to **Public** (GitHub → Packages → package → settings) so
pulls need no auth. (Keep it private instead and consumers authenticate to GHCR.)

## Updating a pin manually

```bash
git submodule update --remote --recursive   # or bump just one
git commit -am "chore: bump vendored targets"
git push      # triggers a build
```

## Constraints / notes

- **Public repo on purpose:** it holds no secrets — only submodule pointers to
  already-public upstreams plus this workflow — and public repos get free native
  arm64 runners.
- **arm64-only by default:** these images currently target aarch64 hosts. Add
  `linux/amd64` to a target's `platforms` for multi-arch (slower; cross-builds).
- Targets that need bespoke build logic don't belong here — this is for projects
  whose own Dockerfile builds cleanly with no extra steps.
