# Mission Control v0.1

A private MCP app that turns a bounded outcome into governed work: persisted tasks, evidence, independent verification, explicit approval gates, and an append-only audit trail.

## Run locally

```powershell
cd C:\Users\commi\Documents\Codex\2026-08-22\build-mcp-app-chatgpt-conversation-6a89c308\mission-control
npm run dev
```

Open `http://localhost:3000` for Skybridge DevTools. The MCP endpoint is `http://localhost:3000/mcp`.

## Validate

```powershell
npm run check
npm test
npm run build
```

The first server start creates `mission-control.sqlite` in the project directory. It is local runtime state and intentionally ignored by Git.

## Current v0.1 boundary

The `local/manual` adapter has no external side effects. It records an auditable run but cannot satisfy verification by itself; attach substantive evidence before using `verify_task`. Hermes, Telegram, cron, provider routing, autonomous execution, and deployment are intentionally deferred.

## Connect to ChatGPT for local testing

Start a public HTTPS tunnel (for example, `npm run dev:tunnel`) and use the resulting `/mcp` URL when creating a private app in ChatGPT Developer Mode. Refresh the app after tool or metadata changes.
