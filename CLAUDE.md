# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository contains [dotslash](https://dotslash-cli.com) files — lightweight JSON executables that download and cache platform-specific binaries on first run. Each file at the repo root is a vendored CLI tool.

## Adding/Updating Tools

Use the `dotslash-creator` skill (`.claude/skills/dotslash-creator/SKILL.md`). It handles GitHub Release discovery, asset downloading, hash computation, and archive inspection.

Default platforms: `macos-aarch64`, `macos-x86_64`, `linux-x86_64`.

## Version Control

This is a **Jujutsu (jj)** repository. Do NOT use raw git commands — use `jj` equivalents:

- `jj st` — status
- `jj log` — history
- `jj diff --git` — view changes
- `jj desc -m "message"` — describe current commit
- `jj new -m "message"` — start new work
- `jj git push -b <bookmark>` — push

Always use `-m` flags (no interactive editor commands).
