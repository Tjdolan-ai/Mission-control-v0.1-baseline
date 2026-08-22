# Mission Control MCP App v0.1

Mission Control is a shared human-assistant control plane. It persists a binary mission outcome and 4–10 bounded tasks; tasks have an actor, mode, capabilities, dependencies, evidence, verification result, approval policy, and optimistic version.

The server owns canonical SQLite state. The widget only projects it and calls MCP tools. Every command returns a fresh canonical snapshot and appends an audit event.

v0.1 includes local/manual execution only. It has no external side effects. Hermes, Telegram, scheduling, provider routing, autonomous execution, and production deployment are deferred.
