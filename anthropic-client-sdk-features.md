# Anthropic Client SDK (`@anthropic-ai/sdk`) — Essential Features

Compiled from `api.md`, `helpers.md`, `README.md`, and the `src/` source (v0.110.0).

## Core API resources

- **Messages API** (`client.messages.create`) — the primary interface to Claude: text, images, PDFs, tool use, extended thinking, citations, web search, code execution result blocks.
- **Message Batches** (`client.messages.batches`) — asynchronous bulk processing of message requests (create, retrieve, list, cancel, stream results).
- **Token counting** (`client.messages.countTokens`) — count prompt tokens before sending.
- **Models API** (`client.models`) — list and retrieve available models.
- **Text Completions** (`client.completions`) — legacy completions endpoint.

## Streaming

- **`client.messages.stream()` → `MessageStream`** — high-level helper: event emitter (`text`, `streamEvent`, `contentBlock`, `message`, …), async iterator, progressive message accumulation (snapshots), `finalMessage()`, `abort()`.
- **`client.messages.create({ stream: true })`** — raw server-sent-events async iterable; lower memory (no accumulation).

## Tools and structured outputs

- **Tool runner** (`ToolRunner` / `BetaToolRunner`, `src/lib/tools/`) — declare runnable tools and let the SDK drive the tool-use loop automatically, including compaction control and session-aware running.
- **Zod and JSON-schema helpers** (`src/helpers/`) — define tool inputs with Zod or JSON Schema (typed via `json-schema-to-ts`) and parse structured outputs.
- **Built-in tool implementations** (`src/tools/`) — agent toolset (filesystem, skills) and memory tool, with Node implementations and browser-safe shims.
- **MCP support** — types and wiring for MCP server tools in messages and sessions.

## Beta platform surfaces (`client.beta.*`)

- **Files API** — upload, list, download, delete files.
- **Skills** — manage skills and skill versions.
- **Managed Agents** — Agents, Environments, Sessions (with event streaming and thread accumulation helpers, `src/lib/sessions/`), Deployments, DeploymentRuns, Vaults (credential injection), MemoryStores, UserProfiles.
- **Self-hosted environment runner** (`src/lib/environments/` poller/worker) — run agent environments on your own infrastructure.
- **Webhooks** — signed webhook verification (built on `standardwebhooks`) for agent and deployment events.

## Client infrastructure

- **Auth options** — `apiKey` (env `ANTHROPIC_API_KEY`), `authToken` bearer auth (env `ANTHROPIC_AUTH_TOKEN`), and a pluggable credential provider chain (`src/lib/credentials/`): user OAuth, OIDC federation, identity tokens, token caching.
- **Retries and timeouts** — automatic retries with backoff (`maxRetries`, default 2), configurable `timeout`, request cancellation via `AbortSignal`.
- **Auto-pagination** — `for await` over list endpoints fetches pages transparently.
- **Typed errors** — `APIError` subclasses per HTTP status (rate limit, authentication, overloaded, …) in `src/core/error.ts`.
- **Middleware and fetch customization** — request/response middleware (`src/core/middleware.ts`), custom `fetch`, `fetchOptions` (proxies, agents), custom headers/query per request.
- **Raw response access** — `.asResponse()` / `.withResponse()` on every `APIPromise`.
- **File uploads** — `toFile()` and support for `File`/streams across runtimes (`src/core/uploads.ts`).

## Runtime and platform support

- **Runtimes** — Node.js ≥ 20, Deno ≥ 1.28, Bun, Cloudflare Workers, Vercel Edge, Nitro; browsers opt-in via `dangerouslyAllowBrowser` (off by default to protect API keys).
- **Companion packages** (`packages/`) — `@anthropic-ai/bedrock-sdk` (AWS Bedrock), `@anthropic-ai/vertex-sdk` (Google Vertex AI), `@anthropic-ai/foundry-sdk` (Azure AI Foundry), plus an AWS SDK integration package — same API shape, platform-specific auth/transport.
