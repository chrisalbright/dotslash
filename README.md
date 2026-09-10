# dotslash

Vendored CLI tools as [dotslash](https://dotslash-cli.com) files. Each file at the repo
root is a small JSON executable that downloads, verifies, and caches a platform-specific
binary the first time you run it.

The point is that the binaries themselves stay out of version control. What gets committed
is a pinned version, a URL, and a SHA-256 digest — so every machine runs the same bytes.

## Usage

Install dotslash, then run the files directly:

```sh
brew install dotslash        # or: cargo install dotslash
./rg --version
./jq '.name' package.json
```

Add the repo to your `PATH` to use the tools without the `./` prefix.

## Tools

| File | Tool | Version | Platforms |
|------|------|---------|-----------|
| `jq` | [jqlang/jq](https://github.com/jqlang/jq) | 1.8.2 | macos-aarch64, macos-x86_64, linux-x86_64 |
| `rg` | [BurntSushi/ripgrep](https://github.com/BurntSushi/ripgrep) | 15.2.0 | macos-aarch64, macos-x86_64, linux-x86_64 |
| `indexion` | [trkbt10/indexion](https://github.com/trkbt10/indexion) | v0.18.0 | macos-aarch64, linux-x86_64 |
| `indexion-macos15` | [trkbt10/indexion](https://github.com/trkbt10/indexion) | v0.16.0 | macos-aarch64 |

Upstream indexion publishes no Intel macOS build, hence the missing `macos-x86_64` entry.

### Why `indexion-macos15` exists

Upstream moved to macOS 26 CI runners at v0.17.0, so the current `darwin-arm64` binaries
link against symbols that don't exist on macOS 15 and abort with a `dyld` error. v0.16.0 is
the last release built with `minos 15.0`. Delete `indexion-macos15` once everything is on
macOS 26.

## Adding or updating a tool

Ask Claude Code — the `dotslash-creator` skill in `.claude/skills/` handles release
discovery, hash computation, and archive inspection. See `CLAUDE.md` for details.

To check whether a pin is stale:

```sh
gh api repos/{owner}/{repo}/releases/latest --jq '.tag_name'
```

## Version control

This is a [Jujutsu](https://jj-vcs.github.io/jj/) repo. Use `jj`, not raw `git`.
