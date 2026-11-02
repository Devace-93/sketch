# sketch

**Draw what the site should look like; let an agent build it.**
`sketch.3m4.net` is a canvas for UI mocks: free drawing, ready-made stencils
(header, menu, hero, section, form, card grid, tab bar…), device frames
(desktop browser, mobile browser, iOS, Android), notes about colours,
behaviour and content, and, for existing sites, a real screenshot as the
background so changes are drawn on top of what is there. Every sketch is saved
as a PNG **and** a structured JSON, so a [flows](https://github.com/Devace-93/flows)
workflow or any MCP client can hand it to an agent to implement. MIT.

> Built by Enrique Magallon Alvarez. Hosted edition at `sketch.3m4.net`; the
> core is free to self-host.

This repo is **empty on purpose**: this README is the brief for the agent that
builds it. The shared contract between the platform apps lives in
[flows/CONTRACT.md](https://github.com/Devace-93/flows/blob/main/CONTRACT.md);
read it first. The canvas is built in-house (SVG/Canvas in Qwik), not on top of
a third-party whiteboard library.

---

## What sketch owns

- **Sketches**: a document with one or more **frames** (artboards), each with
  a kind (`web-desktop 1440×900`, `web-tablet`, `web-mobile 390×844`, `ios`,
  `android`, `free`) and a layer stack.
- **Elements** (the JSON vocabulary an agent reads):
  - primitives: `rect`, `ellipse`, `line`, `arrow`, `path` (freehand), `text`,
    `image`;
  - stencils: `header`, `nav-menu`, `sidebar`, `hero`, `section`, `card-grid`,
    `list`, `table`, `form` (with `fields[]`), `button`, `input`, `modal`,
    `tab-bar`, `footer`, `image-placeholder`, `avatar`, `breadcrumb`,
    `pagination`. Each has typed props (e.g. `nav-menu.items[]`,
    `button.variant`, `section.columns`);
  - `note`: sticky text attached to an element or a region: colour, behaviour,
    copy, responsive rules;
  - `link`: "this button opens frame X" (navigation between frames);
  - `background`: a screenshot (sha256 blob) locked under everything, with
    opacity.
  Every element has `id`, `bbox`, `z`, optional `label`, `notes[]`, `style`
  (fill, stroke, radius, font, colour tokens), and a free `props` object.
- **Screenshots**: `POST /api/v1/screenshots {url, viewport, full_page}` runs
  Playwright in the worker (CONTRACT §10 SSRF rules) and returns a blob; or
  upload a PNG. Captures are versioned with the sketch.
- **Brief**: `GET …/brief` renders the JSON into agent-ready Markdown: frames,
  layout tree (top → bottom, left → right), every stencil with its props,
  every note in place, navigation links, colours used, plus the PNG links.
  This is what agents actually consume; the JSON is for tooling.
- **Versions**: every save is a version; "named versions" for hand-offs;
  diff view highlights moved/added/removed elements.
- **MCP server** (`/mcp`).

Not in sketch: implementing the mock (flows + crew `mock-implementer`),
diagrams (blueprint).

## Architecture

```
Caddy (edge, sketch.3m4.net)
 ├─ web      Qwik City (Bun)   /sketches /sketches/:id (editor) /embed /settings
 ├─ api      Flask             REST (/api/v1), brief renderer, PNG export, MCP (/mcp)
 └─ worker   Python            screenshots (Playwright + Chromium), PNG rasterisation, outbox
     Postgres (workspaces, members, sketches, versions, blobs, tokens, service_keys, outbox)
     Redis (rate limits, job queue) · blobs volume (/data/blobs, sha256 addressed)
```

Bootstrap from `~/payments` (shape). The worker image adds Playwright +
Chromium (~400 MB; fine). Remove Stripe.

## Editor

- Tools: select, pan/zoom, pen (freehand, smoothed), shapes, arrow, text,
  note, stencil palette (drag onto the frame; snaps to columns), image
  (upload / paste / screenshot from URL), link tool (frame → frame).
- Frames: add from presets; device chrome drawn around the frame (browser bar
  with the URL field, phone notch/home bar); toggle light/dark.
- Grid and snapping: 8 px grid, 12-column layout guides for web frames,
  smart guides to siblings.
- Properties panel per selection: stencil props (menu items, form fields,
  variant), style, label, notes.
- Keyboard: standard (V select, P pen, R rect, T text, N note, S stencil
  search, ⌘Z/⇧⌘Z, ⌘D duplicate, ⌘G group, arrows nudge).
- Collaboration later; v1 is single-editor with optimistic `If-Match` saves
  and autosave every few seconds.
- Export: PNG per frame (server rasterises the same JSON so PNGs are
  deterministic), SVG, the brief (Markdown), the JSON.

## API (v1)

| Method | Path | Does |
|---|---|---|
| `GET/POST` | `/api/v1/sketches` | list (filters `q`, `tag`) / create (`{name, frames?: [preset]}`) |
| `GET/PUT/DELETE` | `/api/v1/sketches/:id[?version=]` | envelope (CONTRACT §4), `data` = document JSON, `If-Match` |
| `POST` | `/api/v1/sketches/:id/versions/:n/name` | name a version |
| `GET` | `/api/v1/sketches/:id/brief[?version=&frame=]` | Markdown brief |
| `GET` | `/api/v1/sketches/:id/frames/:frame.png|.svg` | rendered frame |
| `GET` | `/api/v1/sketches/:id/diff?from=&to=` | element-level diff |
| `POST` | `/api/v1/screenshots` | `{url, viewport, full_page}` → `202 {job_id}` → `GET /api/v1/jobs/:id` → blob |
| `POST/GET` | `/api/v1/blobs` · `/api/v1/blobs/:sha256` | upload / fetch (auth by workspace) |
| `GET` | `/api/v1/stencils` | the stencil vocabulary with prop schemas (for tooling and the brief) |
| `GET/POST/DELETE` | `/api/v1/tokens` · `/api/v1/service-keys` | CONTRACT §3 |
| `GET` | `/healthz` · `/.well-known/m4.json` | |

## MCP (`/mcp`)

Tools: `sketch.list`, `sketch.get` (JSON), `sketch.get_brief` (Markdown; the
one agents should call), `sketch.get_frame_png` (returns an image content
block), `sketch.screenshot` (URL → background on a new frame), `sketch.create`
and `sketch.update` (so an agent can propose a mock back: validated against
the stencil schemas), `sketch.diff`. Resources: `3m4://sketches/*` (JSON
envelope; brief as `text/markdown`; frames as `image/png`).

## Events out (CONTRACT §5)

`sketch.created|updated|deleted`, `sketch.version_named`
with `data: {version, frames_changed: [ids], named: "handoff-1"}`. A typical
flows trigger: "when a version is named, run `mock-implementer` on repo X".

## Web

`/` (sketch list with thumbnails), `/sketches/:id` (editor), `/sketches/:id/v/:n`
(read-only version + diff), `/embed/sketches/:id` (CONTRACT §7; `?frame=` to
pin one frame; `?mode=view|edit`), `/settings`.

## Environment (beyond the template)

`SKETCH_URL`, `FLOWS_URL`/`FLOWS_SERVICE_KEY` (events), `TRACE_URL`/`TRACE_SERVICE_KEY`,
`BLOBS_DIR=/data/blobs`, `BLOB_MAX_MB=20`, `SCREENSHOT_TIMEOUT_S=10`,
`SCREENSHOT_ALLOW_PRIVATE=false`, `SCREENSHOT_MAX_HEIGHT=8000`.

## Rules

- The JSON document is the source of truth; PNGs are derived server-side
  from it (deterministic rendering shared by web preview and worker through
  one TypeScript renderer run under Bun in the worker).
- Stencil vocabulary and prop schemas live in `packages/shared/stencils.json`
  and are the only definition (editor palette, validation, brief, MCP all
  read it).
- Screenshot fetching follows CONTRACT §10 (scheme allowlist, private-range
  check after DNS resolution, timeouts, size caps, no cookies).
- Feature flags: `site-building`, `screenshots`, `mcp-write`, `svg-export`.
- Tests: `make test` (API + renderer snapshots), `make lint`, `make e2e`
  (Playwright drives the editor: draw, stencil, note, save, reload, brief).

## Milestones

1. **Skeleton**: bootstrap, login, workspaces, tokens, service keys,
   manifest, blobs, deploy runbook.
2. **Document + editor core**: JSON schema, frames, primitives, select/pan/
   zoom, undo, autosave with versions, list page with thumbnails.
3. **Stencils + notes + links**: palette, props panel, snapping, notes,
   frame links, `stencils.json`.
4. **Screenshots + brief + PNG**: worker capture, background layer, brief
   renderer, deterministic PNG/SVG export.
5. **MCP + events + embeds**: all tools, events to flows, embed route; verify
   end to end: name a version → flows run → agent reads the brief.
6. **Polish**: diff view, device chrome, dark mode frames, keyboard map.

## Quick start (once built)

```bash
cp .env.example .env
make up            # http://localhost:8089  (Mailpit http://localhost:8029)
make test && make e2e
```

## License

[MIT](LICENSE) © 2026 Enrique Magallon Alvarez.
