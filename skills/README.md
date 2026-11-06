# Claude artifact skills

- [Artifact design](artifact-design/SKILL.md) — the bundled Claude Code 2.1.318 skill file.
- [Artifact diagramming](artifact-diagramming/SKILL.md) — the bundled Claude Code 2.1.318 skill file.

These files are preserved as supplied. The design skill includes Claude Code's `when_to_use` frontmatter, which the Codex skill validator does not accept; changing it would alter the original. Compatibility with other agents is not established.

Claude Code constructs the artifact-capabilities skill at runtime rather than providing a standalone `SKILL.md`. Its [runtime contracts and generated guidance](../runtime/README.md) are separate from these skill packages.
