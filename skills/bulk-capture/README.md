# bulk-capture

A Claude Code skill for bulk-capturing many live web app pages into Figma in one shot — using `generate_figma_design` and Chrome DevTools MCP.

## What it does

The `generate_figma_design` tool handles single-page capture on its own. This skill covers what it doesn't: capturing many pages at once.

It uses Chrome DevTools MCP to open every tab simultaneously with a capture session embedded in the URL hash. Combined with parallel polling, an entire site section — 13, 20 pages or more — lands in Figma in the time it takes to capture one page manually. No manual clicking required.

## Prerequisites

- Figma MCP server connected (remote), with `generate_figma_design` available
- Chrome DevTools MCP server connected, with `new_page` available
- A running dev server
- Figma's capture script injected in your app's root layout (see SKILL.md)

## Installation

### Claude Code

Copy `SKILL.md` into your project's `.claude/skills/bulk-capture/` directory. Claude Code will pick it up automatically.

Or add it to a shared skills directory loaded via `additionalDirectories` in `~/.claude/settings.json`.

### Other MCP clients

Place `SKILL.md` wherever your client loads agent skill files.

## Usage

Invoke with `/bulk-capture` in a Claude Code session, or describe what you want:

> "Capture the dashboard, settings, profile, and billing pages into Figma file [URL]"

The skill handles the rest: generating capture IDs, opening all tabs simultaneously, and polling until complete.

## MCP Tools used

- `generate_figma_design` — Figma MCP (remote)
- `new_page` — Chrome DevTools MCP

## License

CC0 1.0 Universal — public domain. See [LICENSE](LICENSE).
