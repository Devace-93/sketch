# sketch — notes for agents

- Brief: `README.md`. Shared platform contract: `/home/enrique/flows/CONTRACT.md` (on GitHub at Devace-93/flows). A change that touches a shared shape is a PR to the contract first.
- Sibling projects and reference implementations: `/home/enrique/payments` (bootstrap from it: Flask + Qwik + compose + worker + deploy shape; drop everything Stripe), `/home/enrique/accounts` (identity; vendor `dbcore/m4session.py` verbatim), `/home/enrique/metrics` (RBAC, API keys, `packages/mcp-server`), `/home/enrique/kinegrama` (a canvas-heavy Vite app, for drawing/rendering ideas only). Platform siblings: `/home/enrique/flows`, `/home/enrique/crew`, `/home/enrique/blueprint`, `/home/enrique/trace`.
- Everything in this repo (code, comments, commits, docs) is written in English.
- Local stack: `make up` → http://localhost:8089, Mailpit 8029. Compose project name `sketch`. Taken locally: 8080–8088, 8090, 8091 (see the contract).
- Build mode: Claude writes the whole codebase unless Enrique says "modo aprender".
- Brand colour: Tailwind **lime**.
- The canvas is built in-house (SVG in Qwik with a thin Canvas layer for freehand); do not add tldraw/Excalidraw/fabric. One renderer (TypeScript, `packages/renderer`) serves the editor, the thumbnails and the worker's PNG export.
- Work milestone by milestone (README § Milestones); plan mode first; one PR per milestone or smaller.
- Security rules: screenshot fetches follow CONTRACT §10 (SSRF); blobs are sha256-addressed and workspace-scoped; tokens and service keys as sha256; never log them.
