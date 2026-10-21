# Pi integration

`PiRuntimeConnection` connects desktop root sessions through `createPiRuntime` from
`@cherrystudio/agent-runtime-pi`. Cherry owns product messages and branch bindings; Pi owns
native history, execution, and compaction. Independent desktop child projections and Durable
desktop integration remain separate work.

The Pi adapter is a production dependency loaded with dynamic `import()`. Keep its ESM-only native
dependencies outside the bundled CommonJS Main entry; a static import fails during Electron startup.

## AI SDK model binding

`createAiSdkModelBinding` constructs a native Pi `ModelRuntime` and model for Cherry's AI SDK
execution path. Pass its model runtime and model to the Pi session builder. Through the builder's
session requirements, every session on that runtime installs the binding's tool-call guard ahead
of its own extensions and runs with Pi's retry off, whatever settings the host gives.

Each Pi request is converted to AI SDK messages and tools without `execute`. The supplied model
call performs one `streamText` step; Pi alone executes tools and decides whether another step or
compaction is needed. This callback can invoke ai-core's executor with model middleware and request
plugins. It must not install another agent loop or replace Pi's context with Cherry chat history.

`aiSdkProvider.ts` converts streaming text, reasoning, tool calls, errors and usage back to native
Pi events. `aiSdkMessages.ts` applies Pi's replay rules (`transformMessages`) before conversion.
`aiSdkMetadata.ts` keeps provider metadata in Pi signature fields.

## Desktop ownership

`piModelPort.ts` calls Cherry's `buildAgentParams` and ai-core for each native model request.
This preserves provider configuration, credentials, request options, middleware and one billing
owner without entering `AiService`'s agent-session route. Pi receives tools without AI SDK executors;
only Pi executes them. Reasoning, tier and fast-mode changes rebuild the binding before the next turn.

`ThreadProjection` converts ordered runtime items into UI chunks. A snapshot establishes its baseline;
it never replays recovered content into the product archive. Tool items use the Pi facet's opaque
lookup and the existing tool renderer. Observation loss fails the turn and retires the owner.

Before dispatch, Main records submission identity and facts on the existing user row. Native placement
and settlement update those facts; final assistant persistence commits response associations in the
same database transaction. Attachments retain Cherry's existing file-path input representation.
Unmapped provider output fails explicitly instead of silently losing content.

A root opens only the native identity bound to its captured product branch. Confirmed or previously
submitted contexts require one existing, identity-checked file. Only a never-submitted creation intent
may have no file yet; Pi flushes lazily and Cherry publishes its resume token after that evidence.
There is no reconstruction from product messages. The connection stays owned while idle until explicit
retirement or application shutdown; stopping aborts native work and flushes observed input facts.
Queued inputs Pi never placed are marked aborted and are not automatically resent after a stop/error.

## Host-facing contracts

- **`ModelCallPort`** – the only way out to a model. The request carries a `requestId`, Pi's
  `sessionId`, the Pi `model` it is for, and `options`, the host's own per-request settings. The
  bridge never hands the port a request whose signal is already aborted (it can fire while
  `before_provider_request` handlers run).
- **Reasoning** – the host decides it through `requestOptions` (forwarded verbatim on every
  request), never through Pi's thinking level, which lacks Cherry levels such as `ultra`.
- **Output tokens** – every request sets `maxOutputTokens` as Pi's built-in providers do: the
  call's cap, or the model's `maxTokens` when Pi gives none (its agent never does), lowered to what
  the context window leaves after the context estimate and a 4,096-token margin (at least 1).
- **Side channel** – `onError` receives the original error object (for example `APICallError`
  with `statusCode` and `responseBody`; not called on abort), and `onUnmappedPart` receives
  `source`, `file` and provider-executed tool parts. Both get the request identity. Pi keeps only
  the error message, as the provider wrote it, so Pi's context-overflow detection still works.
- **Invalid tool calls** – a call whose input is unparsable or not a JSON object, or whose input
  the stream never finalized (it ended after `tool-input-start`), is replayed with `{}`, and the
  tool never runs on it. When the tool's schema has required fields, Pi's argument validation
  rejects `{}`; otherwise the binding's guard, fed by `onInvalidToolCall` and installed ahead of
  every other extension's `tool_call` handler (an approval prompt never sees the call), answers
  with the AI SDK's error (for an unfinished call: that the stream ended before its input was
  complete). A call to an unknown tool gets Pi's `Tool <name> not found`. A host registering
  `createAiSdkProvider` itself must block the calls `onInvalidToolCall` reports the same way.
- **Usage** – the usage handed to Pi feeds its context accounting and compaction only. The host's
  AI SDK calls already account it, so it must not be billed again.
- **Retries** – Pi's retry is always off for sessions on the binding (a session requirement that
  overrides the host's settings). Retries belong to the AI SDK layer.
- **Model identity** – the binding takes a Cherry model and derives its native identity from
  Cherry's unique model id (`providerId::modelId`) alone: the Pi provider is the Cherry provider
  id, the Pi model id is the Cherry model id and the API is `ai-sdk`. Pi replays a stored reply's
  reasoning and signatures only to that same provider/api/model, so a session reopened through a
  new binding for the same Cherry model keeps them, whatever its name or limits now are. Subagent
  model keys (`provider/id`, kept in a subagent's origin for its continuations) follow the same
  rule, so a continued subagent finds its model again through a binding for that Cherry model.
  Pi 1.3.0 drops another model's reasoning from replay instead of sending it as plain text; that
  model's text and tool calls are kept without their signatures.

## Known gaps

- Pi's `after_provider_response` extension event never fires for these models: `streamText` does
  not expose response headers before the body is consumed.
- Tool call ids from another provider are replayed as-is (no `normalizeToolCallId`). A provider
  with stricter id rules may reject history recorded by a different one.
CherryIN Anthropic tool-only replay uses the stored `responseId` to restore its required thinking
block in `piModelPort.ts`. The bridge forwards this metadata only for the same provider/API/model.

The `aiSdk*.test.ts` integration tests cover the bridge with real AI SDK streaming and a local HTTP
provider. `PiRuntimeConnection.test.ts` covers the real native loop and SQLite associations; `piModelPort.test.ts`
exercises the production provider path with a local HTTP server. These are separate from Electron
and live-provider acceptance.
