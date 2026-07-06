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

## Feature fit for a RAG app

### Core four

1. **Citations** — the most RAG-specific feature in the API. Pass retrieved chunks as `document` content blocks with `citations: { enabled: true }`; response text blocks carry a `citations` array (`cited_text`, `document_index`, char/page locations) — grounded, source-attributed answers without custom citation matching. Constraint: citations are **incompatible with structured outputs** (`output_config.format`) — enabling both returns a 400, so keep answer generation and structured extraction on separate calls.
2. **Prompt caching** — RAG requests are large (system prompt + instructions + retrieved context) and caching is a prefix match. Freeze the system prompt with a `cache_control: { type: "ephemeral" }` breakpoint on it; put per-query retrieved chunks *after* the breakpoint (they change every request). For multi-turn chat over the same documents, also cache the conversation prefix — that's where the ~90% input-cost savings compound. Verify via `usage.cache_read_input_tokens`.
3. **Tool runner for agentic RAG** — define a `search_knowledge_base` tool with `betaZodTool` (typed inputs via Zod) and let `client.beta.messages.toolRunner()` drive the loop: Claude decides what to retrieve, reads results, and re-queries until it can answer. Usually beats one-shot context stuffing on hard questions. Make the tool description prescriptive about *when* to call it — current models are conservative about reaching for tools.
4. **Streaming** — `client.messages.stream()` → `MessageStream` (`text` events for deltas, `finalMessage()` for the complete response) so users see the answer forming.

### Supporting

- **Token counting** (`messages.countTokens`) — budget how many retrieved chunks fit before sending; counts are model-specific, so don't estimate with tiktoken-style libraries.
- **Message Batches** — 50% off for offline corpus work: pre-summarizing chunks, extracting metadata, generating eval question sets. Results are unordered — key by `custom_id`.
- **Files API (beta)** — upload PDFs once and reference by `file_id` across requests instead of re-sending base64; pairs with citations on PDF page locations.
- **Structured outputs** (`messages.parse()` + `zodOutputFormat`) — for the ingestion side (chunk metadata, entity extraction), where the citations conflict doesn't apply.

### Gap to plan for

The Anthropic API has **no embeddings endpoint** — embedding and vector search need external pieces (e.g. Cloudflare Workers AI embeddings + Vectorize; the SDK runs on Workers, so the whole pipeline can live in one Worker).

Default answering model: `claude-opus-4-8`.
