# Workers AI Reference

Workers AI runs open-source models on Cloudflare's global edge network. Free tier: 10,000 neurons/day. When routed through AI Gateway, it provides a unified endpoint for both free and paid models.

## Endpoint: `/compat` vs `/workers-ai/v1`

Workers AI has two endpoints through AI Gateway:

| Endpoint | Path | Format | Recommended? |
|----------|------|--------|-------------|
| Compat | `.../compat` | OpenAI `/chat/completions` format | **Yes** |
| Native | `.../workers-ai/v1` | Workers AI native format | No |

**Always use `/compat`.** The compat endpoint speaks the standard OpenAI `/chat/completions` format that Laravel AI SDK and Prism PHP expect. The native endpoint uses a different request/response format that isn't compatible with these frameworks.

### Two endpoints, two model formats

| Endpoint | URL pattern | Model in JSON body | How gateway determines provider |
|----------|------------|-------------------|-------------------------------|
| **Universal compat** (recommended) | `.../compat` | `workers-ai/@cf/meta/...` | Reads `workers-ai/` prefix from model name |
| **Provider-specific** | `.../workers-ai/v1` | `@cf/meta/...` | Already knows from `workers-ai` in URL path |

**Always use `/compat`.** It's the recommended endpoint because:
- It uses the standard OpenAI `/chat/completions` format
- It supports routing to multiple providers from one endpoint
- The `workers-ai/` model prefix is stripped in the response — you get back just `@cf/...`

The full URL pattern is:
```
https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/compat
```

The SDK appends `/chat/completions` automatically, so the final request goes to `.../compat/chat/completions`.

**Important:** Do NOT add `/v1` after `/compat`. The URL `.../compat/v1/chat/completions` will return "No route for that URI".

## Driver: `workers-ai` (via `meirdick/prism-workers-ai`)

**Install:**
```bash
composer require "meirdick/prism-workers-ai:^0.4"
```

This package provides a first-class `workers-ai` driver for Prism PHP with reasoning model support, session affinity, and fixes for the built-in `xai` driver:

1. **String content format** — The xAI driver wraps user content in `[{type: "text", text: "..."}]` (array format). Workers AI `/compat` expects a plain string for text-only messages. The `workers-ai` driver sends string content.
2. **Object content in structured output** — `/compat` may return `content` as a JSON object instead of a string, causing a TypeError crash. The `workers-ai` driver handles this gracefully.
3. **Embeddings support** — The xAI driver doesn't support embeddings. The `workers-ai` driver has a built-in Embeddings handler.

**Do NOT use the `openai` driver.** It uses `/responses` (as of Prism v4+ / laravel/ai v0.3+), which Workers AI doesn't support. This causes a silent 500 error.

```php
// Laravel AI SDK (config/ai.php)
'workers-ai' => [
    'driver' => 'workers-ai',
    'key' => env('CLOUDFLARE_AI_API_TOKEN'),
    'url' => env('WORKERS_AI_URL'),
],

// Prism PHP (config/prism.php)
'workers-ai' => [
    'api_key' => env('CLOUDFLARE_AI_API_TOKEN', ''),
    'url' => env('WORKERS_AI_URL'),
],
```

## Authentication

Workers AI requires a Cloudflare API token with `Workers AI: Read` permission.

**Creating the token:**
1. Go to dash.cloudflare.com/profile/api-tokens
2. Click "Create Token"
3. Use "Custom token" template
4. Add permission: `Account` > `Workers AI` > `Read`
5. Save the token as `CLOUDFLARE_AI_API_TOKEN` in your `.env`

The token is sent as a standard `Authorization: Bearer` header, which the gateway proxies to Workers AI.

## Recommended Models

All model names must be prefixed with `workers-ai/` when routing through the AI Gateway `/compat` endpoint, so the gateway knows which provider to route to.

| Model | Use Case | Notes |
|-------|----------|-------|
| `workers-ai/@cf/google/gemma-4-26b-a4b-it` | **Recommended default** | Reasoning model, fast, good quality. Uses `delta.reasoning` field |
| `workers-ai/@cf/moonshotai/kimi-k2.5` | **Frontier — smartest** | 256K context, tool calling, vision, structured output. Uses `delta.reasoning_content` field |
| `workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast` | General purpose (no reasoning) | Best balance of quality and speed for non-reasoning tasks |
| `workers-ai/@cf/meta/llama-3.1-8b-instruct` | Cheap/fast tasks | Smallest, fastest, lowest cost |
| `workers-ai/@cf/qwen/qwq-32b` | Reasoning | Multi-step reasoning tasks |
| `workers-ai/@cf/qwen/qwen2.5-coder-32b-instruct` | Code generation | Optimized for code |
| `workers-ai/@cf/mistralai/mistral-small-3.1-24b-instruct` | Multilingual | Strong multilingual support |
| `workers-ai/@cf/deepseek-ai/deepseek-r1-distill-qwen-32b` | Reasoning (alt) | DeepSeek R1 distilled |
| `workers-ai/@cf/meta/llama-4-scout-17b-16e-instruct` | Multimodal | 131K context, MoE, function calling |
| `workers-ai/@cf/baai/bge-large-en-v1.5` | Embeddings | 1024 dimensions. Default model for `Str::toEmbeddings()` when `default_for_embeddings` is `workers-ai` |

## JSON Mode

Workers AI supports JSON mode (`response_format: {type: 'json_schema', ...}`) only with specific models:

**Supported models:**
- `@cf/moonshotai/kimi-k2.5`
- `@cf/meta/llama-3.3-70b-instruct-fp8-fast`
- `@cf/meta/llama-3.1-70b-instruct`
- `@cf/meta/llama-3.1-8b-instruct` / `llama-3.1-8b-instruct-fast`
- `@cf/meta/llama-3.2-11b-vision-instruct`
- `@cf/meta/llama-3-8b-instruct`
- `@cf/deepseek-ai/deepseek-r1-distill-qwen-32b`
- `@hf/nousresearch/hermes-2-pro-mistral-7b`

Models not on this list (e.g., `@cf/qwen/qwq-32b`, `@cf/mistralai/mistral-small-3.1-24b-instruct`) return `"This model doesn't support JSON Schema"`.

**Tips for reliable JSON mode (via curl/direct HTTP only — see SDK limitations below):**
- Keep field names short and simple
- Avoid deeply nested objects
- Keep total fields under ~8
- Use short descriptions
- Avoid nullable schema fields — use empty strings instead

## Reasoning Models (Gemma 4, Kimi K2.5)

Several Workers AI models support chain-of-thought reasoning — they think before answering. The response shape differs by model:

| Model | Streaming field | Non-streaming field |
|-------|----------------|-------------------|
| Gemma 4 (`gemma-4-26b-a4b-it`) | `delta.reasoning` | `message.reasoning` |
| Kimi K2.5 (`kimi-k2.5`) | `delta.reasoning_content` | `message.reasoning_content` |

The `meirdick/prism-workers-ai` package (`^0.4`) handles both transparently via the `ExtractsThinking` trait — no per-model configuration needed.

**Key considerations:**
- **Reasoning tokens count against `max_tokens`.** If `max_tokens` is too low, all tokens go to reasoning and `content` comes back null/empty. Set `max_tokens` to **1000+ for simple tasks, 2000+ for complex ones.**
- **Non-streaming:** Thinking is in `$response->steps[0]->additionalContent['thinking']`.
- **Streaming:** Emits `ThinkingStartEvent` → `ThinkingEvent` (deltas) → `ThinkingCompleteEvent`, then text events. `StreamEndEvent.additionalContent['thinking']` has the full accumulated reasoning.
- **If reasoning exhausts all tokens** (`finish_reason=length`, zero content), `ThinkingCompleteEvent` is still emitted (v0.4.4+ safeguard). Consumers can detect this via empty text + Length finish reason.
- **`reasoning_effort` control:** Pass `->withProviderOptions(['reasoning_effort' => 'high'])` to influence how much the model reasons (model support varies).

**Session affinity (prefix caching):** For multi-turn conversations, pass `->withProviderOptions(['session_affinity' => 'ses_' . $conversationId])` to route requests to the same Workers AI instance. This enables prefix caching (lower TTFT, discounted token costs). Off by default.

## SDK Compatibility

With the `meirdick/prism-workers-ai` package installed, all major Prism features work through Workers AI:

| Feature | Status | Notes |
|---------|--------|-------|
| Text generation | Works | String content format, no garbled responses |
| Structured output | Works | Handles object content gracefully |
| Tool calling | Works | Standard OpenAI function calling format |
| Streaming | Works | SSE streaming via `/chat/completions` |
| Embeddings | Works | Via `/embeddings` endpoint. Requires v0.4.1+ (null tokens fix). 1024 dims with `bge-large-en-v1.5` |
| json_schema mode | Works | On supported models (see JSON Mode section) |
| Reasoning content | Works | `additionalContent['thinking']` for text, `ThinkingEvent` for streaming. Supports both Gemma 4 (`reasoning`) and Kimi K2.5 (`reasoning_content`) field names |
| Session affinity | Works | `->withProviderOptions(['session_affinity' => 'ses_...'])` for prefix caching |
| Provider options | Works | `->withProviderOptions(['reasoning_effort' => 'high'])` forwarded to API |

### Remaining `/compat` notes

- **json_object mode** may not be enforced by Workers AI — use `json_schema` mode instead.
- **Multimodal** (image content) is untested through `/compat` — may work with models like `@cf/meta/llama-4-scout-17b-16e-instruct` or `@cf/moonshotai/kimi-k2.5`.
- **Responses API** — Some models (`@cf/openai/gpt-oss-120b`) support `/responses`, but most Workers AI models only support `/chat/completions`.

### Embeddings with pgvector

Workers AI embeddings work through the Laravel AI SDK's `Embeddings` class and `Str::toEmbeddings()`:

```php
// Set default_for_embeddings to workers-ai in config/ai.php
// Then embeddings "just work":
$embedding = Str::of('custody schedule modification')->toEmbeddings();
// Returns array of 1024 floats

// Or explicitly:
$response = Embeddings::for(['text 1', 'text 2'])->generate('workers-ai');
$response->embeddings; // [[0.123, ...], [0.456, ...]]
```

**pgvector storage:** Use `saveQuietly()`, not `updateQuietly()`, to store embeddings in pgvector columns — the model attribute setter is needed for proper vector casting:

```php
// WRONG — silently fails with pgvector:
$model->updateQuietly(['embedding' => $embedding]);

// CORRECT:
$model->embedding = $embedding;
$model->saveQuietly();
```

**Migration:** 1024 dimensions for `bge-large-en-v1.5`:
```php
Schema::ensureVectorExtensionExists();
$table->vector('embedding', dimensions: 1024)->nullable()->index();
```

**Querying:**
```php
// Auto-embeds the query string via the default provider:
$results = Task::query()
    ->whereVectorSimilarTo('embedding', 'custody hearing preparation')
    ->limit(10)
    ->get();
```

### What always works

Gateway routing for **paid providers** (OpenAI, Anthropic, Gemini, Groq, etc.) works perfectly through the SDK — no issues. The compatibility notes above are specific to Workers AI's `/compat` endpoint.

## Future Compatibility

Laravel AI is moving providers to direct gateways that bypass Prism. The `workers-ai` Laravel AI bridge currently depends on `PrismGateway`. Prism standalone usage is unaffected. If `agent()` calls break after a `laravel/ai` upgrade, update `meirdick/prism-workers-ai` for a compatible version.

## Limitations

### Llama 4 Scout multimodal
`@cf/meta/llama-4-scout-17b-16e-instruct` is natively multimodal and categorized as "Text Generation" (not Image-to-Text). Multimodal content through `/compat` is untested.

## Cost Tiering

With the `workers-ai` driver, Workers AI can be used through the full SDK — no need for direct HTTP fallbacks.

**Tier 1a — Workers AI small models (free/credits, lowest cost):** Chatbots, content suggestions, internal tools, embeddings, simple structured output. Models: Llama 3.1 8B, Llama 3.3 70B.

**Tier 1b — Workers AI reasoning (free/credits, moderate cost):** General purpose with chain-of-thought reasoning, chat, structured output. Models: Gemma 4.

**Tier 1c — Workers AI frontier (free/credits, higher cost):** Complex reasoning, tool calling, document analysis (up to 256K context), quality-critical tasks, vision, agentic workflows. Models: Kimi K2.5.

**Tier 2 — Paid providers via AI Gateway:** Tasks requiring specific provider capabilities (e.g., extended thinking, provider-specific APIs), or when Workers AI models don't meet quality bar.
