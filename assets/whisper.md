Determine the mode from "$ARGUMENTS". If empty, default to "medium". Valid values: "low", "medium", "high", "check". If the argument is none of these, stop and say: `Invalid mode "<arg>". Use low, medium, high, or check.`

## check — is this workspace registered?

If the mode is `check`, run the request below, report the result, and **stop** (don't read files or push anything):

```
curl -s -w "\n%{http_code}" "http://127.0.0.1:2323/context?workspace=$(pwd)"
```

- **curl failed / empty** → `Broker not running on :2323. Start it: bun run --cwd packages/broker dev`
- **status 404** or body `not found` → `Not registered — no VSCode window for $(pwd). Start the broker before VSCode, or reload the window.`
- **body `null`** → `Registered ✓ — but no cursor yet. Click into a file in VSCode.`
- **status 200 with JSON** → `Registered ✓ — active file <document_uri>, line <line> col <col>.`

## 1. Get the cursor context

Run, capturing both body and HTTP status:

```
curl -s -w "\n%{http_code}" "http://127.0.0.1:2323/context?workspace=$(pwd)"
```

Handle the result:
- **curl failed / empty output** (connection refused) → the broker isn't running. Stop and say: `Broker not running on :2323. Start it: bun run --cwd packages/broker dev`
- **status 404** or body `not found` → no extension registered for this workspace. Stop and say: `No VSCode window registered for $(pwd). Open this folder in VSCode (with the Whisperer extension) and try again.`
- **body `null`** → registered, but no cursor yet. Stop and say: `No cursor context yet — click into a file in VSCode first.`
- **status 200 with JSON** → parse `document_uri` and `cursor` ({line, col}) and continue.

## 2. Read the file

Read the file at `document_uri`. Focus around the cursor by effort:
- `low`: a few lines above/below the cursor
- `medium`: the current function or block
- `high`: the entire file

## 3. Generate the completion

- `low`: a few lines continuing from the cursor
- `medium`: a complete code block (function body, loop, etc.)
- `high`: the complete file with your suggestion applied

## 4. Push it to the editor

Run, capturing the status:

```
curl -s -w "\n%{http_code}" -X POST http://127.0.0.1:2323/completion \
  -H "Content-Type: application/json" \
  -d '{"workspace_path": "<pwd>", "effort": "<effort>", "text": "<your completion>"}'
```

Handle the result:
- **curl failed / empty output** → broker went down. Stop and say: `Broker not reachable on :2323.`
- **status 404** → workspace no longer registered (VSCode closed?). Stop and say: `Workspace $(pwd) is no longer registered — is the VSCode window still open?`
- **status 502** → broker reached the registry but couldn't deliver to the extension. Stop and say: `Broker couldn't reach the VSCode extension (it may have crashed or reloaded). Reload the VSCode window.`
- **status 200 / `ok`** → say `Whispered.` For `high`, add: `Open diff in VSCode — accept with the ✓ button or Cmd+Enter.`

