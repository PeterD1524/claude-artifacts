# Artifact runtime contracts

These are the capability contracts for the in-page `claude.use()` API. Their version numbers identify **runtime contracts**, not versions of the HTTP or WebSocket API.

| Version | Contract files |
| --- | --- |
| [0.2.87](contracts/0.2.87/) | Roster, prompt, and TypeScript definitions. |
| [0.2.96](contracts/0.2.96/) | Roster, prompt, and TypeScript definitions. |

In each directory, `contract.json` is the response from `/api/frame/contract/{version}`; `prompt` and the `.d.ts` files are the responses from the corresponding versioned endpoints. The files are preserved as returned by the service. The roster is account-scoped and does not guarantee a capability is available to every viewer or artifact.

Claude Code assembles the artifact-capabilities skill at runtime using the contract and additional instructions. The [0.2.87 skill output](artifact-capabilities/0.2.87.md) shows one assembled result; it is not the server's contract prompt, and environment-specific paths have been removed. Paths to `0.2.87/` definitions in that text are relative to the `contracts/` directory.
