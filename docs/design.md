# design

## Main components

- **broker** — receives completion from Claude Code, routes it to the right extension
- **extension** — VSCode extension, renders the ghost text
- **claude code** — infers the completion, pushes it to the broker

## Routing strategy

Route by **workspace path**, not by focused window.

Claude Code always runs inside a workspace directory (its cwd). The VSCode
extension knows its workspace folder. The broker uses workspace path as the
routing key — no focus tracking, no window_id, no race between focus and delivery.

## Full flow

1. Open VSCode → extension registers: `POST /register { workspace_path, port }`
2. Move cursor → extension tells broker: `POST /context { workspace_path, document_uri, cursor }` (fires on `onDidChangeTextEditorSelection`)
3. User types `/whisper` in Claude Code terminal → Claude Code calls `GET /context?workspace=<cwd>` → gets that workspace's active file + cursor
4. Claude Code infers → `POST /completion { workspace_path, effort, text }` to broker
5. Broker looks up `workspace_path` → forwards to that extension's port
6. Extension renders ghost text
7. Close VSCode → extension sends `POST /unregister { workspace_path }`

## Rendering ghost text — pull vs push

The completion arrives by **push** (HTTP), but VSCode renders ghost text by **pull**:
VSCode decides when to ask the provider for a suggestion, and it only asks on
events like typing or cursor movement. Setting a `pending` variable does not make
VSCode ask.

This caused a bug: the completion landed in `pending`, but nothing showed until the
user pressed an arrow key — *any* cursor movement made VSCode pull again, and only
then did the suggestion appear.

Two extra problems:
- The trigger only renders in a **focused** editor. Firing it while focus was still
  in the terminal ran the provider, consumed `pending`, and rendered nothing.
- Clicking the spot where the cursor already sits does not move the cursor, so it
  fires no event and triggers no pull.

**Fix:** when a `low`/`medium` completion arrives, the extension fakes the pull —
it focuses the editor (`showTextDocument`), then calls
`editor.action.inlineSuggest.trigger`. Focus first so the render lands, trigger
second so VSCode asks the provider while `pending` is set.

## Broker API

**`POST /register`** — extension registers on startup
```ts
{ workspace_path: string, port: number }
```

**`POST /unregister`** — extension cleans up on deactivate
```ts
{ workspace_path: string }
```

**`POST /context`** — extension sends current file + cursor whenever they change
```ts
{ workspace_path: string, document_uri: string, cursor: { line: number, col: number } }
```

**`GET /context?workspace=<path>`** — Claude Code reads context for its workspace
```ts
// response
{ document_uri: string, cursor: { line: number, col: number } }
// null if registered but no cursor yet; 404 if workspace not registered
```

**`POST /completion`** — Claude Code pushes ghost text
```ts
{ workspace_path: string, effort: "low" | "medium" | "high", text: string }
```
Broker looks up `workspace_path` → `POST http://127.0.0.1:<port>/completion { effort, text }` to extension.
Returns `404` if workspace not registered, `502` if forward to extension fails.

## Broker state

```ts
type WindowEntry = {
  port: number
  context: { document_uri: string, cursor: { line: number, col: number } } | null
}

const registry = new Map<string, WindowEntry>() // workspace_path → entry
```

## Repo structure

```
whisperer/
├── packages/
│   ├── broker/
│   │   ├── src/
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── extension/
│       ├── src/
│       │   └── extension.ts
│       ├── package.json      # VSCode extension manifest
│       └── tsconfig.json
├── assets/
│   └── whisper.md            # /whisper slash command source
├── package.json              # Bun workspaces root
└── biome.json                # lint + format config
```

- `broker` — runs with `bun src/index.ts`, pure Bun HTTP server on port `2323`
- `extension` — compiles with `tsc`, output to `out/`, packaged with `vsce`

