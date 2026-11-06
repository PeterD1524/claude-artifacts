# Claude artifact skills

- [Artifact design](artifact-design/SKILL.md) — the bundled Claude Code 2.1.318 skill file.
- [Artifact diagramming](artifact-diagramming/SKILL.md) — the bundled Claude Code 2.1.318 skill file.
- [Artifact capabilities](artifact-capabilities/0.2.87/skill-output.md) — skill output for contract 0.2.87, with environment-specific paths removed. Its type definitions are in the same directory.
- [Artifact capabilities contract 0.2.96](artifact-capabilities/0.2.96/) — the server-provided prompt and TypeScript definitions for that contract version.

Claude Code constructs the artifact-capabilities skill at runtime using its bundled code and the server-provided contract. The contract files are inputs to that skill; they are not a standalone `SKILL.md`.
