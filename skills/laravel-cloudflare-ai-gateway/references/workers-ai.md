# Workers AI Reference

Workers AI runs open-source models on Cloudflare's global edge network. Free tier: 10,000 neurons/day. When routed through AI Gateway, it provides a unified endpoint for both free and paid models.

## Endpoint: `/compat` vs `/workers-ai/v1`

Workers AI has two endpoints through AI Gateway:

| Endpoint | Path | Format | Recommended? |
|----------|------|--------|-------------|
| Compat | `.../compat` | OpenAI `/chat/completions` format | **Yes** |
| Native | `.../workers-ai/v1` | Workers AI native format | No |

**Always use `/compat`.** The compat endpoint speaks the standard OpenAI `/chat/completions` format that Laravel AI SDK and Prism PHP expect. The native endpoint uses a different request/response format that isn't compatible with these frameworks.

The full URL pattern is:
```
https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/compat
```

The SDK appends `/chat/completions` automatically, so the final request goes to `.../compat/chat/completions`.

**Important:** Do NOT add `/v1` after `/compat`. The URL `.../compat/v1/chat/completions` will return "No route for that URI".

## Why `xai` Driver, Not `openai`

This is the most common source of 500 errors with Workers AI.

**The problem:** As of Prism v4+ / laravel/ai v0.3+, the `openai` driver uses OpenAI's newer `/responses` API endpoint. Workers AI's `/compat` endpoint only speaks `/chat/completions`. Requests sent to `.../compat/responses` return a 500 error.

**The fix:** Use the `xai` driver. These drivers all use `/chat/completions`:
- `xai`
- `groq`
- `mistral`
- `deepseek`

Any of them would work, but `xai` is the conventional choice for Workers AI.

```php
// Laravel AI SDK (config/ai.php)
'workers-ai' => [
    'driver' => 'xai',       // NOT 'openai' — causes 500 errors
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
| `workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast` | General purpose | Best balance of quality and speed |
| `workers-ai/@cf/meta/llama-3.1-8b-instruct` | Cheap/fast tasks | Smallest, fastest, lowest cost |
| `workers-ai/@cf/qwen/qwq-32b` | Reasoning | Multi-step reasoning tasks |
| `workers-ai/@cf/qwen/qwen2.5-coder-32b-instruct` | Code generation | Optimized for code |
| `workers-ai/@cf/mistralai/mistral-small-3.1-24b-instruct` | Multilingual | Strong multilingual support |
| `workers-ai/@cf/deepseek-ai/deepseek-r1-distill-qwen-32b` | Reasoning (alt) | DeepSeek R1 distilled |
| `workers-ai/@cf/meta/llama-4-scout-17b-16e-instruct` | Multimodal | 131K context, MoE, function calling |
| `workers-ai/@cf/baai/bge-large-en-v1.5` | Embeddings | Text embedding model |

## JSON Mode

Workers AI supports JSON mode (`response_format: {type: 'json_schema', ...}`) only with specific models:

**Supported models:**
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

## SDK Compatibility

Cloudflare redesigned the `/compat` endpoint in June 2025 as a "Unified API" — a drop-in replacement for the OpenAI API that works with existing OpenAI SDKs. In theory, this means array content format, structured output, and other OpenAI-spec features should work.

In practice, some compatibility gaps have been observed. Test after setup — behavior may vary by model and may improve as Cloudflare updates `/compat`.

### Known issues (test to confirm — may be fixed)

| Issue | Symptom | Root cause | Workaround |
|-------|---------|------------|------------|
| Array content format | Garbled responses or "Your input is not sufficient" | Prism xAI driver sends `content` as `[{type: "text", text: "..."}]` — `/compat` may not parse it | Direct HTTP with string content |
| Structured output type mismatch | TypeError crash in `AssistantMessage` | `/compat` returns `content` as object instead of JSON string | Use paid provider for structured output |
| json_object mode not enforced | Model returns prose instead of JSON | `/compat` doesn't enforce `response_format` constraint | Use `json_schema` mode or paid provider |

### Verification

After setup, test both direct HTTP and SDK calls:

```php
// 1. Direct HTTP (should always work)
$response = Http::withToken(env('CLOUDFLARE_AI_API_TOKEN'))
    ->post(env('WORKERS_AI_URL') . '/chat/completions', [
        'model' => 'workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast',
        'messages' => [['role' => 'user', 'content' => 'Say hello in one word.']],
    ]);

// 2. SDK call (test if /compat handles array content)
$response = agent(instructions: 'Be brief.')
    ->prompt('Say hello.', provider: 'workers-ai', model: 'workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast');
```

If SDK calls fail but direct HTTP works, use the direct HTTP fallback for Workers AI while keeping the config ready for when `/compat` improves.

### Direct HTTP fallback

```php
$response = Http::withToken(config('services.cloudflare.ai_token'))
    ->post(config('services.cloudflare.workers_ai_url') . '/chat/completions', [
        'model' => 'workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast',
        'messages' => [
            ['role' => 'system', 'content' => 'You are a helpful assistant.'],
            ['role' => 'user', 'content' => 'What is the capital of France?'],
        ],
    ]);
$text = $response->json('choices.0.message.content');
```

### Responses API (newer models only)

Some Workers AI models (`@cf/openai/gpt-oss-120b`, `@cf/openai/gpt-oss-20b`) support the OpenAI Responses API at `/ai/v1/responses`. For these models, the `openai` driver may work since it uses `/responses`. This is model-specific — most Workers AI models (Llama, Qwen, Mistral) still only support `/chat/completions`.

### What always works

Gateway routing for **paid providers** (OpenAI, Anthropic, Gemini, Groq, etc.) works perfectly through the SDK — no issues. The compatibility questions are specific to Workers AI's `/compat` endpoint.

## Limitations

### Embeddings need separate config
The `xai` driver only supports text generation — not embeddings. If you need Workers AI embeddings, add a separate `workers-ai-embeddings` provider using `'driver' => 'openai'` (the OpenAI driver's embeddings handler posts to `/embeddings`, which `/compat` supports).

### Llama 4 Scout multimodal
`@cf/meta/llama-4-scout-17b-16e-instruct` is natively multimodal and categorized as "Text Generation" (not Image-to-Text), so it may work through `/compat` with multimodal content arrays. This is untested — and given Issue 1 above, unlikely to work through the SDK regardless.

## Cost Tiering

Use Workers AI for features where direct HTTP calls are acceptable (no SDK). For anything requiring the Laravel AI SDK or Prism PHP, use paid providers through the gateway until the SDK compatibility issues are resolved.

**Tier 1 — Workers AI via direct HTTP (free/credits):** Simple chatbots, content suggestions, internal tools — anywhere you can use `Http::post()` directly.

**Tier 2 — Paid providers via AI Gateway (through SDK):** Agents, structured output, tool calling, document generation, vision/OCR, quality-critical output — anything requiring the full SDK feature set.
