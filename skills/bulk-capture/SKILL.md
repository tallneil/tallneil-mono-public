---
name: bulk-capture
description: Bulk-capture many live web app pages into Figma in one shot — no manual clicking — using generate_figma_design and Chrome DevTools MCP. Use when the user wants to send multiple pages, screens, or an entire flow to Figma simultaneously. For single-page capture, generate_figma_design alone is sufficient.
disable-model-invocation: false
---

# bulk-capture — Bulk capture live web pages into Figma

Captures many live pages from a running dev server into Figma simultaneously — all tabs open in parallel, all poll in parallel, zero manual clicking required.

## Why this skill exists

The `generate_figma_design` tool handles single-page capture on its own. What it doesn't provide is a way to capture many pages at once. Without this skill, you'd open each tab manually, wait for it to complete, then move to the next.

This skill uses Chrome DevTools MCP to open every tab simultaneously with the capture session embedded in the URL hash. Combined with parallel polling, an entire site section lands in Figma in the time it takes to capture one page manually.

## Prerequisites

1. **Figma MCP server** connected with `generate_figma_design` available (remote-only; requires a paid Figma plan)
2. **Chrome DevTools MCP server** connected with `new_page` available
3. **Dev server running** — note the base URL (e.g. `http://localhost:3000`)
4. **Capture script injected** — the app must load Figma's capture script on every page you want to capture, gated behind an environment variable so it never runs in production (see [Setup](#setup))
5. **Target Figma file** — extract `fileKey` from any Figma URL: `figma.com/design/:fileKey/...`. Optional: `nodeId` to target a specific destination frame.

## Setup

Add Figma's capture script to your app's root layout — the file that wraps every page — gated on a local-only environment variable:

```tsx
// Next.js App Router example — app/layout.tsx
{process.env.NEXT_PUBLIC_ENABLE_FIGMA_CAPTURE === 'true' && (
  <Script
    src="https://mcp.figma.com/mcp/html-to-design/capture.js"
    strategy="afterInteractive"
  />
)}
```

The same pattern applies to any framework. Inject the `<script>` tag in your root layout file (Remix `root.tsx`, SvelteKit `+layout.svelte`, Nuxt `app.vue`, etc.), conditioned on a local-only env var.

Add the env var to your local `.env.local` (do not commit this):

```sh
NEXT_PUBLIC_ENABLE_FIGMA_CAPTURE=true   # Next.js
# VITE_ENABLE_FIGMA_CAPTURE=true        # Vite-based frameworks
```

Restart the dev server after adding it. Once loaded, the script activates automatically when it sees a `figmacapture=<id>` hash in the URL — no further configuration needed.

---

## Workflow

### Step 1 — Generate all capture IDs in parallel

Call `generate_figma_design` once per page, all at the same time. Each call returns a unique `captureId`. All target the same `fileKey` (and optional `nodeId`).

```
generate_figma_design(outputMode: "existingFile", fileKey: "...", nodeId: "...")
generate_figma_design(outputMode: "existingFile", fileKey: "...", nodeId: "...")
generate_figma_design(outputMode: "existingFile", fileKey: "...", nodeId: "...")
# ... one per page, all at once
```

### Step 2 — Open all tabs simultaneously

Use `new_page` with `background: true` to open every tab at once without waiting. Each URL gets its own `captureId` in the hash fragment.

```
new_page(url: "<base-url>/dashboard#figmacapture=<id1>&figmaendpoint=...&figmadelay=2000", background: true)
new_page(url: "<base-url>/settings#figmacapture=<id2>&figmaendpoint=...&figmadelay=2000",  background: true)
new_page(url: "<base-url>/profile#figmacapture=<id3>&figmaendpoint=...&figmadelay=2000",   background: true)
# ... all pages simultaneously
```

`figmadelay=2000` gives the page 2 seconds to finish rendering before the capture fires. Increase to 3000–5000 for pages with heavy data fetching or slow animations.

### Step 3 — Poll all capture IDs in parallel

Once the tabs are open, poll every ID simultaneously — no sequential waiting.

```
generate_figma_design(captureId: "<id1>")
generate_figma_design(captureId: "<id2>")
generate_figma_design(captureId: "<id3>")
# ... all at once
```

Each returns `status: "completed"` when done. They typically all finish within one poll round.

## Hash fragment format

```
<base-url>/<path>#figmacapture=<captureId>&figmaendpoint=https%3A%2F%2Fmcp.figma.com%2Fmcp%2Fcapture%2F<captureId>%2Fsubmit&figmadelay=2000
```

Example:

```
http://localhost:3000/dashboard#figmacapture=abc123&figmaendpoint=https%3A%2F%2Fmcp.figma.com%2Fmcp%2Fcapture%2Fabc123%2Fsubmit&figmadelay=2000
```

## Gotchas

### Pages that redirect drop the hash fragment

If a route redirects to a sub-route (e.g. `/account` → `/account/profile`), the hash fragment is lost and the capture fails silently. Use the final destination URL directly.

### Captures come back empty

Two common causes:

- **Auth wall or missing state** — the page requires login or localStorage state (e.g. selected items, feature flags). Open the page manually in the DevTools browser first to seed the session, then re-capture with fresh `captureId`s.
- **`figmadelay` too short** — data hasn't loaded before the capture fires. Increase to 3000–5000 ms and retry.

### Chrome DevTools MCP browser conflict

If `new_page` fails with *"The browser is already running for …/chrome-profile. Use --isolated to run multiple browser instances."*, a stale DevTools browser is still running. Ask the user to quit that Chrome window, then retry. Do not fall back to Bash `open` — it opens the URL in the default browser where the capture script is not injected.

### Partial failures

If only some captures succeed, generate fresh `captureId`s for the failed pages only and re-open just those tabs. Each `captureId` is single-use — never reuse one across pages or retries.
