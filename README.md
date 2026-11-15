<p align="center">
  <img src="https://unpkg.com/opencode-title-updater@0.1.0/assets/header.webp" alt="Title Updater. New context. A clearer title." width="100%" />
</p>

<p align="center">
  <a href="https://www.npmjs.com/package/opencode-title-updater"><img src="https://img.shields.io/npm/v/opencode-title-updater?color=ed4b27&label=npm" alt="npm version" /></a>
  <a href="https://opencode.ai/v2/docs/build/plugins"><img src="https://img.shields.io/badge/OpenCode-V2-171717" alt="OpenCode V2" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-171717" alt="MIT license" /></a>
</p>

# opencode-title-updater

**Keep the session title in step with the work.**

Long sessions change direction. After OpenCode compacts the context, Title Updater uses the finished summary and previous title to rename the session to match its current focus. When the summary is an opaque native checkpoint, it asks the session's chat model using the session context instead.

Readable summaries use one fresh-context request. Native checkpoints use an auxiliary generation request in the original session, without adding a message to its history.

## Install

Add the package to `plugins` in your global or project `opencode.json(c)`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": ["opencode-title-updater"],
  "agents": {
    "title": {
      "model": "openai/gpt-6.1-sol#low"
    }
  }
}
```

Already have `agents.title.model` configured? Keep it. The plugin uses that exact provider, model, and variant. The model above is only an example; use one available through your connected providers.

OpenCode reloads configuration automatically. No server restart is needed.

**Requires OpenCode V2 2.2.1 or newer.** Plugin ID: `hffmnnj.title-updater`.

## How it works

```text
Context compaction finishes
          │
          ├── Readable summary → Configured title model
          └── Native checkpoint → Session chat model and context
                     │
              Updated session title
```

1. Listen for `session.compaction.ended`, for automatic and manual compactions.
2. Read the session's previous title.
3. For a readable summary, ask the configured title model for a short title using only that summary and the previous title. For an opaque native checkpoint with no readable summary, ask the session chat model for a title from its current context instead.
4. Rename the original session only if the result is valid and the operation is still current.

For readable summaries, V2's `experimental.generate.text` API has no access to session history, project instructions, or tools. Native checkpoints are not decoded or passed as text: `session.generate` uses the session's own model and current context, which may include messages arriving after compaction. Authentication stays with OpenCode's provider integration.

## Small by design

| What happens | What does not |
| --- | --- |
| One title request per new, eligible compaction | No plugin-side transcript replay |
| The previous title provides continuity | No extra assistant turn in the original session |
| Newer compactions supersede older title work | No stale result intentionally overwrites newer work |
| Title edits made during generation are respected | No unconditional rename after a slow response |
| Failures leave the existing title in place | No guessed model when a readable summary lacks a configured title model |
| Unloading cancels pending work | No polling, scheduled jobs, or persistent plugin state |

The package has one runtime dependency: the official `@opencode/plugin` SDK.

## Boundaries

- **Configure a title model for readable summaries.** If `agents.title.model` is absent, the plugin skips that update rather than guessing OpenCode's title-model fallback. Native checkpoints use the session's chat model instead.
- **Native checkpoints remain opaque.** With an empty summary and native checkpoint, the plugin asks the session's chat model for a title using its current context. An empty summary without a native checkpoint is skipped. The plugin never decrypts or sends a checkpoint as text.
- **A short, plain-text title is required.** Empty, multiline, malformed, or excessively long responses do not replace the title.
- **Manual names are not permanently pinned.** Each later compaction may update the current title. A rename made while a title request is running is preserved for that request.
- **Provider usage applies.** An eligible compaction makes an additional generation request to the configured title model or, for a native checkpoint, the session's chat model. No automatic plugin retries are added.
- **Experimental V2 API.** Stateless generation is experimental in OpenCode. This release is built against `@opencode/plugin` 2.2.1; future API changes may require an update.

Warnings identify the plugin and failure category without printing summaries, generated titles, or provider error bodies. Remove the package from `plugins` to disable it.

The final title check and rename are separate API operations: OpenCode does not expose an atomic conditional rename. An edit racing in that small interval cannot be guaranteed to win.

## Development

```sh
bun install
bun run check
npm pack --dry-run
```

The tests exercise the plugin's event boundary, exact model selection, request isolation, duplicate events, overlapping requests, manual edits, failures, and cleanup without calling a provider.

## License

[MIT](LICENSE).
