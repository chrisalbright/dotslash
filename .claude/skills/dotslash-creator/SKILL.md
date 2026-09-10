---
name: dotslash-creator
description: Create and update dotslash files that vendor platform-specific CLI tools from GitHub Releases. Use this skill whenever the user wants to add, install, vendor, or update a CLI tool in this repository — even if they don't explicitly say "dotslash." Any mention of adding a tool (e.g., "add ripgrep", "I need fd", "vendor jq", "update rg", "bump to latest") should trigger this skill. Also triggers on "create a dotslash file" or references to making executables available in the repo.
---

# Dotslash File Creator

This skill creates and updates [dotslash](https://dotslash-cli.com) files — lightweight JSON executables that download and cache platform-specific binaries on first run.

## Dotslash File Format

```
#!/usr/bin/env dotslash

{
  "name": "<tool-name>",
  "platforms": {
    "<platform>": {
      "size": <bytes>,
      "hash": "sha256",
      "digest": "<hex>",
      "format": "<archive-format>",
      "path": "<path/to/binary/inside/archive>",
      "providers": [
        {
          "type": "github-release",
          "repo": "<owner/repo>",
          "tag": "<release-tag>",
          "name": "<asset-filename>"
        }
      ]
    }
  }
}
```

## Default Platforms

Always include these three unless the user specifies otherwise:
- `macos-aarch64`
- `macos-x86_64`
- `linux-x86_64`

Add `windows-x86_64`, `linux-aarch64`, or `windows-aarch64` only if the user asks.

## Workflow: Creating a New Dotslash File

### 1. Identify the tool and version

If the user didn't specify a version, find the latest non-prerelease release:

```bash
gh api repos/{owner}/{repo}/releases/latest --jq '.tag_name'
```

Tell the user which version you're using: "Using {tool} {version} (latest)."

### 2. List release assets

```bash
gh release view {tag} --repo {owner}/{repo} --json assets --jq '.assets[].name'
```

### 3. Map assets to platforms

Match asset filenames to platforms using these heuristics:

| Platform | Filename patterns |
|----------|-------------------|
| `macos-aarch64` | `darwin-arm64`, `darwin-aarch64`, `macos-arm64`, `macos-aarch64`, `apple-darwin` + `aarch64`/`arm64` |
| `macos-x86_64` | `darwin-amd64`, `darwin-x86_64`, `macos-x86_64`, `apple-darwin` + `x86_64`/`amd64` |
| `linux-x86_64` | `linux-amd64`, `linux-x86_64`, `linux-musl` + `x86_64`/`amd64`, `unknown-linux` + `x86_64` |

If multiple assets match a single platform (e.g., `musl` vs `gnu` variants), ask the user which to use. If no asset matches a platform, skip that platform and tell the user.

### 4. Download each asset, compute size and hash

For each matched asset:

```bash
gh release download {tag} --repo {owner}/{repo} --pattern '{asset-name}' --dir /tmp/dotslash-work/
```

Then compute size and sha256:

```bash
wc -c < /tmp/dotslash-work/{asset-name} | tr -d ' '
shasum -a 256 /tmp/dotslash-work/{asset-name} | awk '{print $1}'
```

### 5. Inspect archive for executable path

**If the asset is a raw binary** (no archive extension like `.tar.gz`, `.zip`, etc. — just a bare filename like `jq-linux-amd64`):
- Omit the `format` field entirely from the platform entry
- Set `path` to the desired binary name (e.g., `"jq"`)
- Skip archive inspection — there's nothing to extract

**If the asset is an archive**, list its contents and find the executable:

For `.tar.gz`, `.tar.xz`, `.tar.zst`, `.tar.bz2`:
```bash
tar -tf /tmp/dotslash-work/{asset-name}
```

For `.zip`:
```bash
unzip -l /tmp/dotslash-work/{asset-name}
```

Look for the binary — typically the file matching the tool name, without extensions, inside the archive. If there's only one executable-looking file, use it. If there are multiple candidates, ask the user.

The `path` field uses forward slashes and is relative to the archive root (e.g., `ripgrep-14.1.0-x86_64-unknown-linux-musl/rg`).

### 6. Determine archive format

Map the asset file extension to dotslash's `format` field:
- `.tar.gz` / `.tgz` → `"tar.gz"`
- `.tar.xz` → `"tar.xz"`
- `.tar.zst` → `"tar.zst"`
- `.tar.bz2` → `"tar.bz2"`
- `.zip` → `"zip"`
- `.gz` (not tar) → `"gz"`

### 7. Write the dotslash file

Place the file at the repo root, named after the binary found inside the archive. Make it executable:

```bash
chmod +x {filename}
```

### 8. Clean up

```bash
rm -rf /tmp/dotslash-work/
```

## Workflow: Updating an Existing Dotslash File

When the user wants to update/bump a tool:

1. Read the existing dotslash file
2. Extract the GitHub repo from the provider entries (the `repo` field)
3. Determine the target version — if unspecified, fetch latest
4. Re-run the full creation workflow using the extracted repo and new version
5. Overwrite the existing file

Tell the user what changed: "Updated {tool} from {old-tag} to {new-tag}."

## Workflow: HTTP URL Fallback

If the binary isn't on GitHub Releases (user provides direct URLs):

1. Ask for download URLs for each platform
2. Download each URL, compute size and sha256
3. Inspect archive contents for the executable path
4. Write the dotslash file using HTTP providers:

```json
"providers": [
  { "url": "https://example.com/tool-linux-x86_64.tar.gz" }
]
```

## Important Notes

- Always verify the dotslash file is valid JSON (after the shebang line) before finishing
- The shebang `#!/usr/bin/env dotslash` must be the very first line, followed by a blank line, then the JSON
- Use 2-space indentation in the JSON for readability
- If a tool doesn't provide builds for all three default platforms, that's fine — include only the platforms that have assets and tell the user which are missing
