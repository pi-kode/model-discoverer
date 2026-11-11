# @pi-kode/model-discoverer

A Pi extension that discovers, refreshes, and configures model catalogs using authentication already configured for a provider — a `/login` credential or a `models.json` `apiKey`. Targets **Earendil Pi 1.5.2+** and its native provider APIs; older Mario Zechner Pi releases are not supported.

## Capabilities

- **Built-in provider discovery** — Refreshes catalogs for authenticated native Pi providers by delegating to Pi's native refresh pipeline (handling API keys, OAuth renewal, provider endpoints, and persistence without reading `auth.json` directly).
- **Custom provider discovery (`models.json`)** — Opt any OpenAI-compatible (`openai-completions`, `openai-responses`) or Anthropic-compatible (`anthropic-messages`) provider into live `/models` discovery with `"discoverModels": true`, including per-model API overrides (`modelApis`) and Anthropic cursor pagination (`has_more` / `last_id`).
- **Automatic session-start refresh & lifecycle safety** — Runs a non-forced refresh on `session_start` (reusing Pi's startup refresh when fresh), guards against overlapping refreshes, enforces a 30-second timeout, and aborts in-flight requests on `session_shutdown`.
- **On-demand catalog refresh (`/discover-models`)** — Force-refresh all authenticated dynamic providers or a single provider by exact ID, with per-provider status summaries.
- **Interactive capability overrides (`/configure-model`)** — Interactively edit `reasoning`, `contextWindow`, and `maxTokens` for discovered models and persist them under `modelOverrides` in `models.json` so overrides survive future catalog refreshes.
- **Offline persistence & fallback** — Persists discovered catalogs through Pi's model store so models are restored before any network fetch; failed refreshes retain the previous catalog and any handwritten fallback `models`.
- **Security-first networking** — Requires HTTPS (or HTTP on loopback only), blocks HTTP redirects, enforces same-origin endpoints across config reloads, rejects non-API-key credentials for custom endpoints, and sanitizes all error messages so credentials, headers, and upstream response bodies are never leaked.

## Install

Install from npm (requires Earendil Pi 1.5.2+):

```sh
pi install npm:@pi-kode/model-discoverer
```

Restart Pi or run `/reload`. For a one-off trial, use:

```sh
pi -e npm:@pi-kode/model-discoverer
```

To remove the npm-installed extension, run `pi remove npm:@pi-kode/model-discoverer` (for local installs, use the source shown by `pi list`) and reload. `models.json` entries and `/login` credentials are left untouched; delete any `discoverModels` keys you no longer want honored.

## Commands

```text
/discover-models [provider-id]
/configure-model [provider-id]
/model
```

### `/discover-models [provider-id]`

Force-refreshes all authenticated providers that support discovery, or one exact provider ID (matching `[a-z0-9][a-z0-9._-]{0,99}`).

- Does not select a model or change your default model.
- Discovery also runs automatically on `session_start` (without forcing a network fetch when Pi already refreshed).
- After a new `/login`, run `/discover-models`; the extension does not intercept Pi's login command.

### `/configure-model [provider-id]`

Interactively edits per-model capability overrides for a `discoverModels` provider in an interactive Pi session:

- `reasoning`: `true`, `false`, or remove override
- `contextWindow`: positive integer token limit, or clear (`""`, `-`, `remove`, `inherit`, `default`)
- `maxTokens`: positive integer token limit, or clear

Values are merged into `modelOverrides` in `~/.pi/agent/models.json` (empty override objects are automatically pruned). A backup is saved as `models.json.bak` before writing; note that rewriting drops JSON comments and custom formatting in `models.json`. Run `/reload` afterwards to apply.

Overrides sit at Pi's topmost configuration layer, so they survive every catalog refresh. Once a model is marked `reasoning: true`, use Pi's built-in `/thinking` (or `Shift+Tab`) to choose reasoning effort at runtime.

## Built-in providers

No configuration is needed. The extension delegates to Pi's **native refresh pipeline**, which handles stored API keys, OAuth renewal, provider-specific endpoints, filtering, and catalog persistence. It never reads `auth.json` directly and does not replace built-in providers.

A refreshed catalog is **not proof of account entitlement**. Some native providers refresh metadata from Pi's public catalog rather than querying an account-specific endpoint; others keep a static list. Providers without a refresh implementation are reported as unsupported for native discovery and retain their existing models. Actual availability remains subject to Pi's provider policy and the upstream service.

This extension does not invent `/models` endpoints for subscription OAuth providers or send subscription tokens to generic API endpoints.

## Discovering a custom provider (`models.json`)

Opt a provider into discovery with the custom key **`discoverModels: true`** in its `~/.pi/agent/models.json` entry (JSONC `//` comments and trailing commas are supported):

```json
{
  "providers": {
    "mws": {
      "baseUrl": "https://gpt.example.com/projects/abc/openai/v1",
      "api": "openai-completions",
      "apiKey": "$MWS_API_KEY",
      "discoverModels": true
    }
  }
}
```

Pi itself ignores `discoverModels`; this extension reads `models.json` at startup and registers a discovery hook for each opted-in provider. Credentials stay with Pi: the provider's `apiKey` (literal, `$ENV_VAR`, or `${ENV_VAR}`) or a key stored via `/login <provider>` is resolved by Pi's auth layer and passed to the hook. If both exist, the stored `/login` credential wins. Never put discovery-only keys anywhere else.

Reload with `/reload` after editing `models.json`, then run `/discover-models mws` and pick a model with `/model`.

### Provider fields

- `discoverModels`: `true` opts the provider into discovery. This is the extension's custom key.
- `baseUrl`: API base URL (required). HTTPS only, except HTTP on `localhost`, `127.0.0.1`, or `[::1]`.
- `api`: `openai-completions`, `openai-responses`, or `anthropic-messages` (required for discovery).
- `modelsUrl`: optional complete catalog URL; defaults to `{baseUrl}/models`. Must share `baseUrl`'s origin. URL credentials, query strings, and fragments are rejected.
- `modelApis`: optional per-model API overrides, e.g. `{ "model-a": "openai-responses" }`. Unmatched models use the provider `api`.
- `apiKey`, `headers`, `models`, `modelOverrides`, `compat`: ordinary Pi `models.json` fields, applied as usual. Handwritten `models` remain as offline fallback when discovery fails.

Only configure trusted origins: discovery necessarily sends the resolved API key to the configured endpoint. Redirects are blocked. If `models.json` changes the effective `baseUrl` origin after models were discovered, discovery refuses rather than sending the credential to the new origin.

### Catalog format

Custom endpoints must return a JSON object with `data: [{ "id": "chat-model" }]`.

- **Authentication & pagination**: OpenAI-protocol providers (`openai-completions`, `openai-responses`) send `Authorization: Bearer <key>`. Anthropic (`anthropic-messages`) sends `x-api-key` plus `anthropic-version: 2023-06-01` and supports `has_more` / `last_id` cursor pagination via `after_id`. Other pagination schemes fail rather than silently installing an incomplete catalog.
- **Metadata & defaults**: Optional fields recognized per model are `name` or `display_name`, `context_window` or `context_length`, `max_tokens` or `max_output_tokens`, boolean `reasoning`, and `input: ["text", "image"]`. Absent capabilities default to text-only, `reasoning: false`, `128000` context window, and `16384` max tokens (clamped to `contextWindow`). Costs are zero placeholders (unknown, **not necessarily free**).
- **Filtering & limits**: Entries with an explicit non-chat `type` (anything other than `"model"` or `"chat"`) are excluded; an untyped `/models` response is assumed to contain chat models. Requests are bounded to 4 MiB per page, 20 pages, and 10000 models. Duplicate IDs are collapsed. Invalid responses fail without replacing the previous catalog. An explicitly empty successful list clears the discovered catalog.
- **Persistence**: Discovered catalogs are **persisted through Pi's model store**, so they are available offline on the next start and restored before any network fetch. A failed refresh retains the previous catalog. No credentials, headers, or complete upstream responses are logged or stored by this extension.

## Contributing & Development

See [CONTRIBUTING.md](./CONTRIBUTING.md) for local setup, running tests and type checks, architecture guidelines, and the release process.
