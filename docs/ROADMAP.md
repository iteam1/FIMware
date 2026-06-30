# Whisperer — Implementation Roadmap

How to rebuild this project from scratch, phase by phase. Each task has **Do / Verify / Gotcha**. Tracks the order we actually built it in.

Stack: Bun + TypeScript. Two packages — `broker` (router) and `extension` (VSCode). Claude Code drives it via a `/whisper` slash command.

---

## Phase 0 — Design

The design took ~6 doc commits before any code. It converged by **removing** things — three pivots, each one a simplification. Don't rebuild the rejected versions.

- [ ] **0.1 Lock the concept** (`8c0fa99`, `docs/idea.md`). Claude Code infers a completion; VSCode shows it; you accept. Token usage stays low, human stays in the loop.
- [ ] **0.2 Add effort levels** (`7c5cd72`). `low` → a few lines (ghost text), `medium` → a block (ghost text), `high` → whole file (diff). Each effort maps to a UX surface.
  - **Pivot — drop MCP.** First plan used an MCP server; replaced with a plain `curl` POST from Claude Code to the broker. Simpler, no extra process.
- [ ] **0.3 Write `docs/design.md`** (`d5f22f9`). Components, flow, broker routes, port `2323`.
  - **Pivot — no WebSocket.** Considered a persistent socket for push; plain HTTP request/response is enough. The extension runs its own tiny HTTP server to receive completions.
- [ ] **0.4 Pick the routing key** (`c255632`). Route by **workspace path** (Claude Code's cwd), not window.
  - **Pivot — drop focus tracking.** First design tracked the active window via `window_id` + focus events; that races (focus can change between request and delivery). `workspace_path` ties Claude Code's cwd directly to one extension — deterministic. This commit also added `/unregister` and 404/502 handling.

**Verify:** the flow reads end-to-end on paper: register → stream context → `/whisper` reads context → infer → POST completion → broker forwards → extension renders.
**Dead ends (don't rebuild):** MCP server, WebSocket transport, `window_id`/focus-based routing.

---

## Phase 1 — Monorepo scaffold

- [ ] **1.1 Root `package.json`** — Bun workspaces:
  ```json
  { "name": "whisperer", "private": true, "workspaces": ["packages/*"],
    "devDependencies": { "@biomejs/biome": "latest", "@types/node": "^26.0.1" } }
  ```
- [ ] **1.2 `biome.json`** — lint + format (4-space indent). One config at the root only.
- [ ] **1.3** `bun install`.

**Verify:** `bun install` succeeds; `bunx biome lint packages` runs.
**Gotcha:** keep a single `biome.json` at the root — don't duplicate it per package.

---

## Phase 2 — Broker (`packages/broker`)

- [ ] **2.1 Package + types.** `package.json` (`name: @whisperer/broker`, scripts `dev: bun --watch src/index.ts`, `start: bun src/index.ts`), `tsconfig.json` with `"types": ["bun-types"]`, then `bun add -d bun-types`.
- [ ] **2.2 Registry + server skeleton.**
  ```ts
  type WindowEntry = { port: number, context: {document_uri: string, cursor: {line: number, col: number}} | null }
  const registry = new Map<string, WindowEntry>()
  const server = Bun.serve({ port: 2323, hostname: "127.0.0.1", fetch: async (req) => { /* routes */ } })
  ```
- [ ] **2.3 The 5 routes** (match on method + pathname):
  - `POST /register` `{workspace_path, port}` → `registry.set(...)`
  - `POST /unregister` `{workspace_path}` → `registry.delete(...)`
  - `POST /context` `{workspace_path, document_uri, cursor}` → update entry (404 if unknown)
  - `GET /context?workspace=<path>` → return `entry.context` as JSON (404 if unknown)
  - `POST /completion` `{workspace_path, effort, text}` → look up port, forward `POST http://127.0.0.1:<port>/completion {effort, text}`; 404 if unknown, 502 if forward fails (wrap in try/catch)

**Verify:**
```bash
bun run --cwd packages/broker dev
curl -s -X POST localhost:2323/register -d '{"workspace_path":"/tmp/x","port":9999}'   # ok
curl -s "localhost:2323/context?workspace=/tmp/x"                                        # null
curl -s "localhost:2323/context?workspace=/nope"                                         # not found (404)
```
**Gotcha:** `bun-types` is what makes `Bun.serve` type-check. Without it: `Cannot find name 'Bun'`.

---

## Phase 3 — Extension skeleton (`packages/extension`)

- [ ] **3.1 Manifest + deps.** `package.json` with `engines.vscode`, `activationEvents: ["onStartupFinished"]`, `main: ./out/extension.js`, `publisher` (required for install), build script `tsc -p tsconfig.json`. `tsconfig.json` types `["vscode", "node"]`. `bun add -d @types/vscode @types/node typescript`.
- [ ] **3.2 Activate → server → register.** Read `workspace_path` at module scope (so `deactivate` sees it); return early if undefined. Start `http.createServer` on `listen(0)` (random port); in the `listening` event, read `server.address().port` and `POST /register`.
- [ ] **3.3 Stream context.** On `vscode.window.onDidChangeTextEditorSelection`, `POST /context` with the document path + cursor.
- [ ] **3.4 Run it.** Either `.vscode/launch.json` (`"type": "extensionHost"`) for F5, or symlink for a real window (see Phase 7).

**Verify:** `Output → Whisperer` shows `whisperer activated`; `curl "localhost:2323/context?workspace=<your folder>"` returns non-404.
**Gotchas:**
- No folder open → `workspace_path` is undefined → extension returns early, logs nothing. Open a folder.
- `console.log` from an installed extension goes to DevTools, **not** Output → Extension Host. Use `vscode.window.createOutputChannel("Whisperer")`.
- Start the **broker before** VSCode — the extension only registers at startup.

---

## Phase 4 — Inline ghost text: `low` / `medium`

- [ ] **4.1 Provider + mailbox.** A module variable `pending: string | null`. Register `registerInlineCompletionItemProvider({pattern:"**"}, ...)`; in `provideInlineCompletionItems`, if `pending` return `new InlineCompletionItem(pending, new Range(position, position))` then clear it.
- [ ] **4.2 Handle `/completion`** in the extension's HTTP server: read `{effort, text}`; for `low`/`medium`, set `pending = text`.
- [ ] **4.3 Auto-show (the hard part).** Setting `pending` does nothing on its own — VSCode renders ghost text by **pull** (it asks the provider on type/cursor events), but our data arrives by **push** (HTTP). Fake the pull:
  ```ts
  const editor = vscode.window.activeTextEditor
  if (editor) {
    vscode.window.showTextDocument(editor.document, editor.viewColumn).then(() =>
      vscode.commands.executeCommand("editor.action.inlineSuggest.trigger"))
  }
  ```
  Focus first (ghost text only renders in a **focused** editor), trigger second.

**Verify:** `curl -X POST localhost:2323/completion -d '{"workspace_path":"<folder>","effort":"low","text":"const x = 1"}'` → ghost text appears with no keypress.
**Gotcha:** if you trigger while focus is in the terminal, the provider runs, consumes `pending`, and renders nothing. That's why we `showTextDocument` first.

---

## Phase 5 — High effort: diff + accept

- [ ] **5.1 Virtual document.** `registerTextDocumentContentProvider("whisperer", { provideTextDocumentContent: () => pendingDiff ?? "" })`. The diff's right side is a `whisperer://` URI served from memory.
- [ ] **5.2 Open the diff** on `effort === "high"`: store `pendingDiffTarget = activeEditor.document.uri`, set `pendingDiff = text`, open with a **unique** right-side URI (`whisperer://diff/completed-${n++}`) so VSCode doesn't show cached content:
  ```ts
  vscode.commands.executeCommand("vscode.diff", original, completed, "Whisperer suggestion")
  ```
- [ ] **5.3 Accept command.** Register `whisperer.acceptDiff`: open the target doc, `WorkspaceEdit.replace(fullRange, pendingDiff)`, `applyEdit`, `doc.save()`, then `closeActiveEditor`.
- [ ] **5.4 Contributions in `package.json`:**
  ```json
  "commands":   [{ "command": "whisperer.acceptDiff", "title": "Whisperer: Accept Suggestion", "icon": "$(check)" }],
  "menus":      { "editor/title": [{ "command": "whisperer.acceptDiff", "when": "whisperer.diffOpen", "group": "navigation" }] },
  "keybindings":[{ "key": "ctrl+enter", "mac": "cmd+enter", "command": "whisperer.acceptDiff", "when": "whisperer.diffOpen" }]
  ```
- [ ] **5.5 Scope + reset.** `setContext("whisperer.diffOpen", true)` when opening the diff. Reset to `false` (and clear `pendingDiff`) on `window.tabGroups.onDidChangeTabs` when a closed tab's `input instanceof TabInputTextDiff && input.modified.scheme === "whisperer"`.

**Verify:** `/whisper high` → diff opens → click ✓ (or Cmd+Enter) → file overwrites and saves; `cat` the file to confirm on disk.
**Gotchas:**
- Don't bind `Tab` — it's the most contested key in VSCode; the keypress never reached our command. Use a title-bar button + `Cmd/Ctrl+Enter`.
- `vscode.diff` is **read-only** — there's no built-in accept. You build it (5.3).
- Use `key` + `mac` for cross-platform bindings.

---

## Phase 6 — Claude Code integration (`/whisper`)

- [ ] **6.1 Command source `assets/whisper.md`.** Steps: read effort from `$ARGUMENTS` (default `medium`) → `GET /context?workspace=$(pwd)` → read file around cursor → infer a completion sized to effort → `POST /completion`.
- [ ] **6.2 Error handling.** Capture HTTP status (`curl -s -w "\n%{http_code}"`) and explain each failure instead of failing silently: empty/refused → broker down; 404 → not registered; `null` body → no cursor yet; 502 → extension unreachable.
- [ ] **6.3 Install it.** Copy to `.claude/commands/whisper.md` (project) or `~/.claude/commands/` (global). `.claude/` is gitignored — keep the source in `assets/`.

**Verify:** in a Claude Code session in the project, `/whisper low|medium|high` each render in VSCode.
**Gotcha:** the live command is `.claude/commands/whisper.md`; editing only `assets/` won't change behavior until you copy it across.

---

## Phase 7 — Distribution

- [ ] **7.1 Symlink install.** `ln -s "$(pwd)/packages/extension" ~/.vscode/extensions/locch.whisperer-0.0.1`. The folder name **must** be `publisher.name-version`, then **quit and reopen** VSCode (reload isn't enough).
- [ ] **7.2 `GUIDELINE.md`** — prerequisites, setup, run, use, cleanup.

**Verify:** fresh clone → follow GUIDELINE → working `/whisper`.
**Gotcha:** if VSCode marks the extension obsolete (`~/.vscode/extensions/.obsolete`), the symlink name was wrong — fix the name, not the file.

---

## Phase 8 — Observability & status

Added after the core worked, to make the in-memory-registry gotcha debuggable instead of mysterious.

- [ ] **8.1 Broker logging.** `console.log` on `/register` and `/unregister` with workspace, port, and active count:
  ```
  + register   <workspace> -> :<port>  (N active)
  - unregister <workspace>  (N active)
  ```
  Watch the broker terminal to see windows connect and drop live.
- [ ] **8.2 `/whisper check` mode.** A status-only mode (alongside `low`/`medium`/`high`): queries `GET /context` and reports whether this workspace is registered (and the active file/cursor), without reading files or pushing. Run it to confirm the broker↔extension link before whispering — especially after restarting the broker.

**Verify:** `/whisper check` → `Registered ✓ — active file …` when connected; restart the broker and it flips to `Not registered…` until you reload the VSCode window.
**Why it matters:** the broker's registry is in-memory, so a broker restart silently drops every registration. Logging + `check` make that visible.