# Pi ByteRover Port Design

## Goal

Convert this package from an OpenCode ByteRover plugin into a Pi-only ByteRover extension/package. Preserve the existing memory behavior while using Pi's extension APIs and package conventions.

## Configuration

The extension reads JSON configuration from, in order:

1. `.pi/byterover.json`
2. `~/.pi/agent/byterover.json`
3. built-in defaults

The existing Zod config schema remains the source of truth. Invalid config disables the extension for that load and reports a Pi notification/log message. The defaults stay aligned with the current OpenCode plugin: automatic recall and persist enabled, manual tools enabled, `brv` as the default executable, and the same timeout and prompt values.

## Pi Extension Entry Point

`src/index.ts` exports a default Pi extension factory:

```ts
export default function (pi: ExtensionAPI) { ... }
```

On initialization/session start, the extension:

- loads and validates config;
- skips registration when `enabled` is false;
- bootstraps `.brv/.gitignore` in the current working directory;
- creates a `BrvBridge` using the active config;
- registers manual ByteRover tools when enabled;
- attaches Pi event handlers for recall and curation.

The package becomes Pi-only. OpenCode-specific dependencies, types, docs, tests, and plugin exports are removed.

## Event Mapping

OpenCode behavior maps to Pi as follows:

- `experimental.chat.system.transform` becomes `before_agent_start`.
  - Adds manual tool guidance when manual tools are enabled.
  - Runs automatic recall when enabled and bridge is ready.
  - Appends recalled context to the system prompt inside the configured XML-style tag.
- `session.idle` becomes `agent_end`.
  - Persists the latest completed user request after the assistant has finished.
- `experimental.session.compacting` becomes `session_before_compact`.
  - Persists the latest turn before compaction proceeds.

Automatic persist uses option A from the design discussion: persist once after every completed Pi agent response. It is deduplicated by the IDs of the latest selected branch entries so repeated events for the same completed request do not write duplicate memories.

## Message Selection and Formatting

The OpenCode message helpers are replaced with Pi session helpers based on `ctx.sessionManager.getBranch()`.

Initial Pi support formats only text from message entries with roles `user` and `assistant`. Tool results, custom messages, compaction summaries, and branch summaries are excluded from ByteRover curation/recall in the first port unless they are needed later.

Formatting remains:

```text
[user]: user text

[assistant]: assistant text
```

Recall selection keeps the existing limits:

- `maxRecallTurns`
- `maxRecallChars`

Persist selection captures the latest completed user request: the most recent user message and following assistant messages in the active branch.

## Manual Tools

The extension registers three Pi tools with `pi.registerTool` and `typebox` schemas:

- `brv_recall`
  - Args: `query`, optional `timeoutMs`
  - Calls `BrvBridge.recall`
  - Strips echoed recall query from the result
- `brv_search`
  - Args: `query`, optional `limit`, optional `scope`, optional `timeoutMs`
  - Calls `BrvBridge.search`
  - Formats ranked results as readable text
- `brv_persist`
  - Args: `context`, optional `timeoutMs`
  - Calls `BrvBridge.persist` with `detach: true`
  - Does not wrap text in the automatic curation prompt

The tools return text content for the model and compact details for rendering/debugging.

## Error Handling and Notifications

Bridge failures do not crash Pi. The extension reports warnings/errors through `ctx.ui.notify` when UI is available and logs to stderr or a small internal logger otherwise.

Quiet mode suppresses user-facing notifications but still logs diagnostic messages.

If ByteRover is not ready:

- automatic recall is skipped because recall requires an initialized ByteRover workspace;
- manual recall and search return a clear text message;
- persist does **not** check or block on `BrvBridge.ready()`. ByteRover bootstraps automatically when `persist` is called against an uninitialized workspace, so both automatic persist and `brv_persist` should call persist directly and only handle actual persist failures.

## Package and Documentation

The package manifest is updated for Pi:

- remove OpenCode dependencies;
- add Pi peer dependencies (`@mariozechner/pi-coding-agent`, `typebox`, and any Pi packages used only for extension types);
- add `keywords: ["pi-package"]`;
- add a `pi.extensions` manifest entry pointing to the built extension.

README is rewritten for Pi installation and usage, including `.pi/byterover.json` configuration examples.

## Testing

Tests are updated to mock the Pi extension API and contexts instead of OpenCode plugin inputs.

Coverage should include:

- config loading precedence and validation;
- disabled extension behavior;
- `.brv/.gitignore` bootstrap and normalization;
- manual tool registration and execution;
- automatic recall injection in `before_agent_start`;
- automatic persist on `agent_end`;
- curation deduplication, including concurrent calls;
- `session_before_compact` curation;
- quiet/error paths.

## Out of Scope

- Supporting OpenCode and Pi from the same package.
- Persisting tool results or custom messages into ByteRover by default.
- Adding interactive configuration UI.
- Changing ByteRover bridge behavior or CLI requirements.
