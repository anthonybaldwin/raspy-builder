# raspy-builder

Builds upstream projects that ship a Dockerfile but publish no image, and pushes
them to GHCR (arm64) so they can be digest-watched and updated like any normal
image. Targets are git submodules under `vendor/`, listed in `targets.json`.

A target builds **only when its submodule moves** — never on plain code/doc
commits. Twice daily a `watch` job checks each upstream and rebuilds the ones that
changed; a manual pin bump (push) builds the changed targets; or run it by hand
from the Actions tab, picking a target — and `force` to rebuild a current pin
without waiting for an upstream change.

## Targets

- `claude-code-trace` — [delexw/claude-code-trace](https://github.com/delexw/claude-code-trace)
- `codex-trace` — [PixelPaw-Labs/codex-trace](https://github.com/PixelPaw-Labs/codex-trace)

## Add a target

```bash
git submodule add <url> vendor/<name>
# add { "name", "context", "dockerfile" } to targets.json
git commit -am "add <name>" && git push   # builds the new target
```

## Bump a pin manually

```bash
git submodule update --remote --recursive
git commit -am "bump" && git push
```
