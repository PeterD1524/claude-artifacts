# Claude Artifacts for coding agents

This repository describes how a coding agent could work with Claude-hosted artifacts. It contains no integration program.

- [API reference](api/README.md): an unofficial description of Anthropic's HTTP and WebSocket interfaces.
- [Integration design](SPEC.md): proposed behavior for an agent that uses those interfaces.
- [Runtime contracts](runtime/README.md): versioned capability definitions and generated capability guidance.
- [Skills](skills/README.md): the bundled artifact design and diagramming skill files.

The Claude frame protocol is private and subject to change. The spec separates proposed requirements from observed service behavior and identifies the portions that still require end-to-end verification.
