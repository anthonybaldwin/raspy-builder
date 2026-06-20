# raspy-builder

Builds upstream projects that ship a Dockerfile but publish no image, and pushes
them to GHCR (arm64) so they can be digest-watched and updated like any normal
image. Targets are git submodules under `vendor/`, listed in `targets.json`;
GitHub Actions builds each on a native arm64 runner and weekly bumps them to
upstream HEAD.

## Targets

- `claude-code-trace` — [delexw/claude-code-trace](https://github.com/delexw/claude-code-trace)
- `codex-trace` — [PixelPaw-Labs/codex-trace](https://github.com/PixelPaw-Labs/codex-trace)

## Add a target

```bash
git submodule add <url> vendor/<name>
# add { "name", "context", "dockerfile" } to targets.json
git commit -am "add <name>" && git push
```

After the first build, set the new GHCR package to **Public** (GitHub → Packages
→ settings) so pulls need no auth.

## Bump a pin manually

```bash
git submodule update --remote --recursive
git commit -am "bump" && git push
```
