# Architecture v0.1

`Human + assistant → MCP tools and React view → server-authoritative SQLite repository → local/manual adapter`

Mutations verify the supplied entity version and write an audit event in the same SQLite transaction. Substantive evidence is required for deterministic verification. A verified task is required for completion, and explicit approval is required where the task policy says so.
