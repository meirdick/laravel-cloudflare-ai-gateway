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

**Tips for reliable JSON mode:**
- Keep field names short and simple
- Avoid deeply nested objects
- Keep total fields under ~8
- Use short descriptions
- Avoid nullable schema fields — use empty strings instead

## Limitations

### No multimodal through `/compat`
The `/compat` endpoint only accepts `content` as a **string**. OpenAI-format multimodal content arrays (`[{type: 'text', ...}, {type: 'image_url', ...}]`) are rejected with: `"Type mismatch of '/messages/0/content', 'array' not in 'string'"`.

For vision/OCR, route to OpenAI (gpt-4o-mini) or Anthropic via AI Gateway instead.

### Embeddings need separate config
The `xai` driver only supports text generation — not embeddings. If you need Workers AI embeddings, add a separate `workers-ai-embeddings` provider using `'driver' => 'openai'` (the OpenAI driver's embeddings handler posts to `/embeddings`, which `/compat` supports).

### Llama 4 Scout multimodal
`@cf/meta/llama-4-scout-17b-16e-instruct` is natively multimodal and categorized as "Text Generation" (not Image-to-Text), so it may work through `/compat` with multimodal content arrays. This is untested.

## Cost Tiering

**Tier 1 — Workers AI (free/credits):** Chatbots, task generation, entity summaries, intent classification, content suggestions, internal tools.

**Tier 2 — Paid providers via AI Gateway:** Document generation, complex analysis, vision/OCR, quality-critical output, strict structured output compliance.
