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

## SDK Compatibility (Critical)

Workers AI's `/compat` endpoint is **not fully OpenAI-compatible**. While it works via direct HTTP/curl, there are three breaking incompatibilities when used through the Laravel AI SDK or Prism PHP:

| Feature | Via curl | Via Laravel AI SDK / Prism | Root cause |
|---------|---------|---------------------------|------------|
| Plain text generation | Works (string content) | **Broken** | SDK sends array content format |
| Tool calling | Works | **Broken** (same reason) | User messages use array format |
| Structured output (json_schema) | Partial (returns object) | **Crashes** | SDK expects string, gets object |
| json_object mode | Not enforced | **Broken** | Model returns prose, not JSON |
| Streaming | Works | **Broken** (same reason) | Array content format issue |

### Issue 1: Array content format (affects ALL SDK requests)

The Prism xAI driver always serializes user messages as arrays, even for plain text:
```json
{"role": "user", "content": [{"type": "text", "text": "What is 2+2?"}]}
```

Workers AI `/compat` only accepts content as a string:
```json
{"role": "user", "content": "What is 2+2?"}
```

The array format is valid per the OpenAI spec, but Workers AI doesn't parse it correctly — the model receives garbled input and responds with nonsense like "Your input is not sufficient." This is a Cloudflare `/compat` endpoint limitation, not a Prism or Laravel AI SDK bug.

This affects every request through the SDK, not just multimodal messages.

### Issue 2: Structured output returns object, not string

When using `json_schema` response format, Workers AI returns `content` as a parsed JSON object:
```json
{"content": {"intent": "interested", "confidence": 0.8}}
```

The OpenAI spec requires it as a JSON string:
```json
{"content": "{\"intent\":\"interested\",\"confidence\":0.8}"}
```

Prism's `Structured.php` does `new AssistantMessage($content)` which expects a string — receiving an object causes a TypeError crash. This breaks any agent using `HasStructuredOutput`.

### Issue 3: json_object mode not enforced

Workers AI does not enforce `response_format: {type: "json_object"}`. The model returns prose with markdown-embedded JSON instead of clean JSON, causing parse failures.

### Workarounds

1. **Direct HTTP calls** — bypass the SDK and use `Http::post()` with string content format:
   ```php
   $response = Http::withToken(config('services.cloudflare.ai_token'))
       ->post(config('services.cloudflare.workers_ai_url') . '/chat/completions', [
           'model' => 'workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast',
           'messages' => [
               ['role' => 'user', 'content' => 'What is the capital of France?']  // string, not array
           ],
       ]);
   ```
   Works for plain text, but loses SDK features (tools, structured output, middleware, streaming helpers).

2. **Wait for Cloudflare fix** — the `/compat` endpoint should handle array content format since it's valid per the OpenAI spec. This is the most likely resolution path.

3. **Wait for Prism fix** — Prism could add a string-only fallback when no images are present in the content array. This would fix Issue 1 but not Issues 2 or 3.

4. **Custom Prism provider** — write a Workers AI-specific Prism provider that serializes messages as strings and handles object content responses. Most effort, but gives full control.

### What still works

Gateway routing for **paid providers** (OpenAI, Anthropic, Gemini, Groq, etc.) works perfectly through the SDK — no issues. The incompatibilities are specific to Workers AI's `/compat` endpoint.

## Limitations

### Embeddings need separate config
The `xai` driver only supports text generation — not embeddings. If you need Workers AI embeddings, add a separate `workers-ai-embeddings` provider using `'driver' => 'openai'` (the OpenAI driver's embeddings handler posts to `/embeddings`, which `/compat` supports).

### Llama 4 Scout multimodal
`@cf/meta/llama-4-scout-17b-16e-instruct` is natively multimodal and categorized as "Text Generation" (not Image-to-Text), so it may work through `/compat` with multimodal content arrays. This is untested — and given Issue 1 above, unlikely to work through the SDK regardless.

## Cost Tiering

Use Workers AI for features where direct HTTP calls are acceptable (no SDK). For anything requiring the Laravel AI SDK or Prism PHP, use paid providers through the gateway until the SDK compatibility issues are resolved.

**Tier 1 — Workers AI via direct HTTP (free/credits):** Simple chatbots, content suggestions, internal tools — anywhere you can use `Http::post()` directly.

**Tier 2 — Paid providers via AI Gateway (through SDK):** Agents, structured output, tool calling, document generation, vision/OCR, quality-critical output — anything requiring the full SDK feature set.
