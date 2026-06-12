---
name: laravel-cloudflare-ai-gateway
description: >-
  Configures Laravel and PHP projects to route AI traffic through Cloudflare AI
  Gateway for observability, caching, and rate limiting. Adds Workers AI as a
  free open-source model provider. Supports Laravel AI SDK (laravel/ai) and
  Prism PHP (prism-php/prism). Use when the user mentions Cloudflare AI Gateway,
  Workers AI, AI observability, AI cost monitoring, or wants to proxy AI provider
  traffic. Also trigger when someone asks about logging AI requests, caching AI
  responses, rate limiting AI calls, monitoring AI usage, using free AI models,
  reducing AI costs, or routing OpenAI/Anthropic/Gemini traffic through a gateway
  — even if they don't explicitly mention "Cloudflare" or "gateway".
license: MIT
user_invocable: true
compatibility: Requires a Laravel or PHP project with laravel/ai or prism-php/prism. Needs a Cloudflare account.
compatible_agents:
  - Claude Code
  - Cursor
  - Laravel Boost
  - Gemini CLI
  - OpenCode
  - Roo Code
tags:
  - laravel
  - php
  - cloudflare
  - ai-gateway
  - workers-ai
  - observability
  - prism
metadata:
  author: mdick85
  version: "1.2.0"
---

# Cloudflare AI Gateway for Laravel

Configure a Laravel project to route AI traffic through Cloudflare AI Gateway for observability, caching, and rate limiting. Optionally adds Workers AI (free open-source models) as a provider.

## Instructions

When invoked, follow these four phases in order. **Never skip Phase 3 (plan approval).**

---

### Phase 1 — Detect

Examine the current project to understand what's already configured.

**Step 1: Check `composer.json`** for:
- `laravel/ai` — Laravel AI SDK
- `prism-php/prism` or `echolabsdev/prism` — Prism PHP

**Step 2: Read the config file(s):**
- Laravel AI SDK: `config/ai.php`
- Prism PHP: `config/prism.php`

**Step 3: Note:**
- Which providers exist (openai, anthropic, gemini, groq, etc.)
- Whether providers already have `url` fields
- Whether a `workers-ai` provider already exists
- Whether `.env.example` already has gateway URL vars

**Framework differences:**

| Aspect | Prism PHP (`config/prism.php`) | Laravel AI SDK (`config/ai.php`) |
|--------|-------------------------------|----------------------------------|
| URL in default config? | YES — all providers have `url` with `env()` | NO — not in default config |
| Key field name | `api_key` | `key` |
| Gateway routing change | Env vars only | Add `url` field to config + set env vars |
| Adding Workers AI | Add entry to `config/prism.php` | Add entry to `config/ai.php` |

Report what you found before moving to Phase 2.

---

### Phase 2 — Ask the User

Ask these questions in order. For each, explain what and why before asking. **Never ask users to type secrets — instruct them where to add secrets to `.env` themselves.**

#### Question 1: Gateway Setup

Explain: "AI Gateway acts as a proxy between your app and AI providers (OpenAI, Anthropic, etc.), giving you logging, caching, and rate limiting without any code changes — you just change the base URL and your existing API keys pass through."

Then ask: Do you have a Cloudflare account with an AI Gateway already created?

- **If no:** Explain how to create one: Go to dash.cloudflare.com > AI > AI Gateway > Create Gateway. Choose a name (this becomes the URL slug). Then come back with:
  - Their **Account ID** (32-character hex string from the dashboard URL or Overview page)
  - Their **Gateway name** (the slug they chose)
- **If yes:** Ask for their Account ID and Gateway name.

#### Question 2: Gateway Mode

Explain: "AI Gateway supports four authentication modes. For Laravel projects, pass-through (open) is the only mode that works without custom HTTP middleware."

Briefly describe the modes:
1. **Pass-through (open)** — Your API keys pass through in headers. Gateway needs no auth. Works natively with Laravel.
2. **Pass-through (authenticated)** — Same, but gateway requires a `cf-aig-authorization` header. Needs custom middleware.
3. **BYOK** — Keys stored in Cloudflare dashboard. Needs custom middleware.
4. **Unified billing** — Cloudflare manages keys and billing. Needs custom middleware.

Recommend pass-through (open). Point to `references/gateway-modes.md` for details.

Ask which mode. If they choose anything other than pass-through (open), warn that it requires custom HTTP middleware not covered by this skill.

#### Question 3: Workers AI

Explain: "Workers AI runs free open-source models (Gemma 4, Kimi K2.6, Llama, Qwen, etc.) on Cloudflare's edge network. Free tier includes 10,000 neurons/day. Models like Gemma 4 and Kimi K2.6 support chain-of-thought reasoning, surfaced as native reasoning/thinking stream events. Laravel AI SDK projects get a native `workers-ai` driver via `meirdick/laravel-cf-workersai`; Prism PHP projects use `meirdick/prism-workers-ai`."

Ask if they want to add Workers AI as a provider.

If yes, instruct: "You'll need a Cloudflare API token with `Workers AI: Read` permission. Create one at dash.cloudflare.com/profile/api-tokens and add it as `CLOUDFLARE_AI_API_TOKEN=your-token` in your `.env` file."

#### Question 4: Default Provider (only if adding Workers AI)

Explain: "You can set Workers AI as your default provider so all AI calls use free models unless explicitly overridden. Paid providers (OpenAI, Anthropic) remain available by specifying them explicitly."

Ask if they want to change the default provider to Workers AI.

---

### Phase 3 — Present Plan

After gathering all answers, present a complete implementation plan. **The user must approve before any changes are made.**

Use this template:

```
## Implementation Plan

### Config changes
- [list each file and what will be added/modified]

### Environment variables (.env.example and .env)
- [list each variable with its value]

### Other changes
- [default provider change, if applicable]

Approve this plan? (yes/no)
```

**Provider URL mapping** (use to build gateway URLs):

| Provider | Gateway path suffix | Direct default URL |
|----------|--------------------|--------------------|
| OpenAI | `/openai` | `https://api.openai.com/v1` |
| Anthropic | `/anthropic` | `https://api.anthropic.com/v1` |
| Gemini | `/google-ai-studio/v1beta/models` | `https://generativelanguage.googleapis.com/v1beta/models` |
| Groq | `/groq/openai/v1` | `https://api.groq.com/openai/v1` |
| Mistral | `/mistral/v1` | `https://api.mistral.ai/v1` |
| xAI (Grok) | `/grok/v1` | `https://api.x.ai/v1` |
| DeepSeek | `/deepseek/v1` | `https://api.deepseek.com/v1` |
| Workers AI (Prism only) | `/compat` | N/A — Laravel AI SDK uses `meirdick/laravel-cf-workersai` (`account_id` + `gateway` config, no URL) |

The gateway base URL pattern is:
```
https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}
```

Only include providers that already exist in the project config. Do not add providers the project doesn't use.

---

### Phase 4 — Execute

**Only after the user approves the plan.**

#### Step 1: Config changes (Laravel AI SDK only)

Add a `url` field with env fallback to each existing provider in `config/ai.php`. Example:

```php
'openai' => [
    'driver' => 'openai',
    'key' => env('OPENAI_API_KEY'),
    'url' => env('OPENAI_URL', 'https://api.openai.com/v1'),
],
```

For Prism PHP: skip this step — providers already have `url` fields reading from env.

Use these env var names and default URLs:

| Provider | Env var | Default URL |
|----------|---------|-------------|
| OpenAI | `OPENAI_URL` | `https://api.openai.com/v1` |
| Anthropic | `ANTHROPIC_URL` | `https://api.anthropic.com/v1` |
| Gemini | `GEMINI_URL` | `https://generativelanguage.googleapis.com/v1beta/models` |
| Groq | `GROQ_URL` | `https://api.groq.com/openai/v1` |
| Mistral | `MISTRAL_URL` | `https://api.mistral.ai/v1` |
| xAI | `XAI_URL` | `https://api.x.ai/v1` |
| DeepSeek | `DEEPSEEK_URL` | `https://api.deepseek.com/v1` |

#### Step 2: Workers AI provider (if requested)

The package depends on the framework. **These are different packages — do not mix them up.**

**Laravel AI SDK** — install the native provider:
```bash
composer require "meirdick/laravel-cf-workersai:^0.3"
```

This is a native `laravel/ai` provider (no Prism dependency, no `config/prism.php`, no gateway URL to compose). It supports text, embeddings, structured output, tool calling, streaming, reasoning replay across tool turns, retries, and session affinity. Requires `laravel/ai ^0.7 || ^0.8`. Configure in `config/ai.php`:

```php
'workers-ai' => [
    'driver' => 'workers-ai',
    'key' => env('CLOUDFLARE_AI_API_TOKEN'),
    'account_id' => env('CLOUDFLARE_ACCOUNT_ID'),
    'gateway' => env('CLOUDFLARE_AI_GATEWAY'), // optional — omit to hit Workers AI directly
],
```

The package builds the endpoint from `account_id` (+ optional `gateway`); model names are plain `@cf/...` with **no `workers-ai/` prefix**. The credential key is `key` (laravel/ai convention) — not `api_key`.

**Prism PHP** — install the Prism provider:
```bash
composer require "meirdick/prism-workers-ai:^0.4"
```

This provides a first-class `workers-ai` driver for Prism that handles Workers AI's content format differences, structured output quirks, and reasoning model support (ThinkingStart/ThinkingEvent/ThinkingComplete events, session affinity for prefix caching, `reasoning_effort` forwarding). Configure in `config/prism.php`:

```php
'workers-ai' => [
    'api_key' => env('CLOUDFLARE_AI_API_TOKEN', ''),
    'url' => env('WORKERS_AI_URL'), // .../compat — model names need the workers-ai/ prefix
],
```

#### Step 3: `.env.example`

Append a gateway URL block. Only include providers that exist in the project config.

```env
# === Cloudflare AI Gateway ===
# Route AI traffic through Cloudflare for observability, caching, rate limiting.
# Remove/comment these to route directly to providers instead.
OPENAI_URL=https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/openai
ANTHROPIC_URL=https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/anthropic
GEMINI_URL=https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/google-ai-studio/v1beta/models
GROQ_URL=https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/groq/openai/v1
MISTRAL_URL=https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/mistral/v1
XAI_URL=https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/grok/v1
DEEPSEEK_URL=https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/deepseek/v1
```

If Workers AI was requested, also append. For **Laravel AI SDK** (native package — no URL needed):
```env

# === Workers AI (free open-source models) ===
CLOUDFLARE_ACCOUNT_ID={account-id}
CLOUDFLARE_AI_API_TOKEN=
CLOUDFLARE_AI_GATEWAY={gateway-name}
```

For **Prism PHP**:
```env

# === Workers AI (free open-source models) ===
CLOUDFLARE_AI_API_TOKEN=
WORKERS_AI_URL=https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/compat
```

Replace `{account-id}` and `{gateway-name}` with placeholders in `.env.example`.

#### Step 4: `.env`

Copy the same block to `.env`, but with the user's actual account ID and gateway name filled in.

#### Step 5: Default provider (if requested)

Laravel AI SDK: Change `'default'` in `config/ai.php` to `'workers-ai'`.

#### Step 6: Verify Workers AI (if configured)

After setup, verify Workers AI is working via `php artisan tinker`:

```php
// Laravel AI SDK (laravel-cf-workersai) — note: no workers-ai/ model prefix
use function Laravel\Ai\agent;

$response = agent(instructions: 'You are a helpful assistant.')
    ->prompt('Say hello in one word.', provider: 'workers-ai', model: '@cf/meta/llama-3.1-8b-instruct');
echo $response->text;

// Prism PHP (prism-workers-ai) — model prefix required through /compat
$response = Prism::text()
    ->using('workers-ai', 'workers-ai/@cf/google/gemma-4-26b-a4b-it')
    ->withPrompt('Say hello in one word.')
    ->asText();
echo $response->text;
```

Then confirm the requests appear in the gateway dashboard (AI > AI Gateway > [gateway-name] > Logs).

#### Step 7: Report

After completing all changes, report:

1. **What was detected** — framework, config file, existing providers
2. **What was changed** — list each file and modification
3. **Verification result** — whether Workers AI responded correctly via direct HTTP and SDK
4. **Manual steps remaining:**
   - If Workers AI: "Add your Cloudflare API token as `CLOUDFLARE_AI_API_TOKEN` in `.env`"
   - Run `php artisan config:clear` if config is cached
   - Verify in Cloudflare dashboard (AI > AI Gateway > [gateway-name] > Analytics)

---

## Gotchas

These are hard-won lessons from deploying AI Gateway across 8+ Laravel projects, plus live stress-testing against the production Workers AI API (2026-06-11). Read these before executing — they'll save you from the most common failures.

### Both frameworks

- **Do not use the `openai` driver for Workers AI.** The `openai` driver (as of Prism v4+ / laravel/ai v0.3+) sends requests to `/responses`, but Workers AI only speaks `/chat/completions`. Using `'driver' => 'openai'` causes a silent 500 error.
- **Gemini URL must include `/v1beta/models`.** Unlike other providers where the gateway path is just the provider name (e.g., `/openai`), Gemini requires `.../google-ai-studio/v1beta/models` because Gemini's API expects this prefix in every request path.
- **The env var is `CLOUDFLARE_AI_API_TOKEN`, not `CLOUDFLARE_AI_API_KEY`.** Cloudflare's convention is "API Token" (scoped) vs "API Key" (global). Workers AI needs a scoped token with `Workers AI: Read` permission.
- **Model choice matters for tool calling.** Verified live: `@cf/meta/llama-3.3-70b-instruct-fp8-fast` **never emits tool calls** — it answers in prose instead. Use `@cf/meta/llama-4-scout-17b-16e-instruct` or `@cf/openai/gpt-oss-120b` for tool-using agents. Even those only *choose* to call a tool some of the time under `tool_choice: auto` — pass `tool_choice: required` (via provider options) when the tool must run.
- **Reasoning models (Gemma 4, Kimi K2.6) emit thinking tokens that count against `max_tokens`.** If the budget is too low, the model spends it all on reasoning and returns empty content. Set the budget high enough (1000+) for reasoning models.
- **Reasoning models are slow.** Kimi K2.6 has been observed taking 45s+ on small prompts; structured 70B requests can exceed 60s under load. The default timeout in both SDKs is 60s — raise it (`#[Timeout(120)]` in laravel/ai) for big/reasoning models.

### Laravel AI SDK (`meirdick/laravel-cf-workersai`)

- **The credential key is `key`, not `api_key`** — matching every first-party laravel/ai provider. (`api_key` is accepted as a fallback since v0.3.0, but configs written against the v0.2.0 docs crashed.)
- **No `workers-ai/` model prefix.** The package routes via `account_id` + `gateway` config, not the `/compat` endpoint, so model names are plain `@cf/...`.
- **No `config/prism.php` needed.** Older versions of this skill described a Prism bridge; the native package (v0.3.0+, laravel/ai `^0.7 || ^0.8`) has no Prism dependency. If you find a stale `prism.php` workers-ai entry from a previous setup, it can be removed.
- **Truncation is surfaced, not silent.** Cloudflare misreports budget-exhausted completions as `finish_reason: "stop"`; the package normalizes them to `FinishReason::Length` and defaults `max_completion_tokens` to 4096 (Cloudflare's own default is 256 — far too small for structured output). Override with `default_max_tokens` in the provider config.
- **Timeouts fail fast.** Since v0.3.0 the retry policy does not retry transfer timeouts (previously a 60s timeout became ~3 minutes of wall time across 3 attempts). Connect failures and 502/503/504 are still retried with backoff.
- **A forced `tool_choice` relaxes to `auto` on follow-up turns automatically** — otherwise the model would be forced to call a tool again instead of answering, looping until max-steps with empty text.
- **Session affinity** is a config/provider-options key (`session_affinity`), sent as the `x-session-affinity` header for prefix caching on multi-turn conversations.

### Prism PHP (`meirdick/prism-workers-ai`)

- **Workers AI requires the package (`^0.4`).** The built-in xAI driver sends user content as arrays, crashes on object content from structured output, and has no reasoning model support. Install with `composer require "meirdick/prism-workers-ai:^0.4"`.
- **Workers AI URL is `.../compat` — no trailing `/v1`.** The SDK appends `/chat/completions` automatically. If you add `/v1`, the final URL becomes `.../compat/v1/chat/completions` which returns "No route for that URI".
- **Model names need the `workers-ai/` prefix through `/compat`.** The AI Gateway uses this prefix to route to the correct provider. Without it, the gateway doesn't know where to send the request. The response strips the prefix automatically.
- **Different models use different reasoning field names.** Kimi uses `delta.reasoning_content`, Gemma 4 uses `delta.reasoning`. The `ExtractsThinking` trait handles both transparently.
- **Session affinity:** pass `->withProviderOptions(['session_affinity' => 'ses_' . $conversationId])` to route consecutive requests to the same Workers AI instance for prefix caching.
- **Embeddings require v0.4.1+.** Workers AI's `/compat/embeddings` endpoint omits `usage.total_tokens`; earlier versions crashed with a TypeError.
- **Use `saveQuietly()` not `updateQuietly()` for pgvector columns.** `$model->updateQuietly(['embedding' => $array])` silently fails with pgvector — the array doesn't get cast to the vector format. Use `$model->embedding = $array; $model->saveQuietly();` instead.

## Usage Examples

### Workers AI in Laravel AI SDK (`meirdick/laravel-cf-workersai`)

Model names are plain `@cf/...` — **no `workers-ai/` prefix** (the native package routes via config, not the `/compat` endpoint).

```php
// Using default provider (if set to workers-ai)
$response = agent(instructions: 'You are helpful.')->prompt('Hello!');

// Explicit provider and model
$response = agent(instructions: 'You are helpful.')
    ->prompt('Hello!', provider: 'workers-ai', model: '@cf/google/gemma-4-26b-a4b-it');

// Via attributes
#[Provider('workers-ai')]
#[Model('@cf/google/gemma-4-26b-a4b-it')]

// Tool-using agent: pick a tool-capable model and force the call
// (llama-3.3-70b never emits tool calls; auto is flaky on the rest)
#[Provider('workers-ai')]
#[Model('@cf/meta/llama-4-scout-17b-16e-instruct')]
class NumberAgent implements Agent, HasProviderOptions, HasTools
{
    public function providerOptions(Lab|string $provider): array
    {
        // Custom drivers arrive as a string, not a Lab enum case.
        // The package relaxes this to 'auto' on follow-up turns automatically.
        return $provider === 'workers-ai' ? ['tool_choice' => 'required'] : [];
    }
    // ...
}

// Slow/reasoning models need a higher timeout (laravel/ai defaults to 60s)
#[Timeout(120)]
#[Model('@cf/moonshotai/kimi-k2.6')]
```

### Workers AI in Prism PHP

```php
use Prism\Prism\Facades\Prism;

// Text generation (Gemma 4 — reasoning model, emits thinking + content)
Prism::text()
    ->using('workers-ai', 'workers-ai/@cf/google/gemma-4-26b-a4b-it')
    ->withPrompt('Hello!')
    ->asText();
// $response->steps[0]->additionalContent['thinking'] contains the reasoning chain

// Streaming with reasoning events
$stream = Prism::text()
    ->using('workers-ai', 'workers-ai/@cf/google/gemma-4-26b-a4b-it')
    ->withMaxTokens(2000)
    ->withPrompt('Explain quantum entanglement.')
    ->asStream();
// Yields: ThinkingStartEvent → ThinkingEvent deltas → ThinkingCompleteEvent
//         → TextStartEvent → TextDeltaEvent deltas → TextCompleteEvent → StreamEndEvent
// StreamEndEvent.additionalContent['thinking'] has the full accumulated reasoning

// With reasoning_effort control
Prism::text()
    ->using('workers-ai', 'workers-ai/@cf/moonshotai/kimi-k2.6')
    ->withProviderOptions(['reasoning_effort' => 'high'])
    ->withPrompt('Solve this step by step.')
    ->asText();

// With session affinity for multi-turn prefix caching
Prism::text()
    ->using('workers-ai', 'workers-ai/@cf/google/gemma-4-26b-a4b-it')
    ->withProviderOptions(['session_affinity' => 'ses_' . $conversationId])
    ->withPrompt($followUpMessage)
    ->asText();

// Non-reasoning model (no thinking overhead)
Prism::text()
    ->using('workers-ai', 'workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast')
    ->withPrompt('Hello!')
    ->asText();

// Structured output (works natively — no TypeError)
Prism::structured()
    ->using('workers-ai', 'workers-ai/@cf/google/gemma-4-26b-a4b-it')
    ->withSchema($schema)
    ->withPrompt('Classify this intent.')
    ->generate();

// Embeddings
Prism::embeddings()
    ->using('workers-ai', 'workers-ai/@cf/baai/bge-large-en-v1.5')
    ->fromInput('Hello world')
    ->generate();
```

### Recommended Models

Names below are the plain Workers AI model IDs (Laravel AI SDK usage). For Prism through `/compat`, prefix each with `workers-ai/`.

| Model | Use Case | Reasoning? | Tool calls? |
|-------|----------|------------|-------------|
| `@cf/google/gemma-4-26b-a4b-it` | General purpose, chat, structured output | Yes | — |
| `@cf/moonshotai/kimi-k2.6` | Complex reasoning, math, code (slow — raise timeout) | Yes | Yes |
| `@cf/meta/llama-4-scout-17b-16e-instruct` | Tool-using agents, multimodal | No | Yes (use `tool_choice: required`) |
| `@cf/openai/gpt-oss-120b` | Tool-using agents, reasoning | Yes | Yes |
| `@cf/meta/llama-3.3-70b-instruct-fp8-fast` | Fast general purpose, no thinking overhead | No | **Never** (verified live) |
| `@cf/meta/llama-3.1-8b-instruct` | Cheapest, simple tasks | No | — |
| `@cf/baai/bge-large-en-v1.5` | Embeddings (1024 dims) | N/A | N/A |

### Embeddings in Laravel AI SDK

```php
use Illuminate\Support\Str;
use Laravel\Ai\Embeddings;

// Set default_for_embeddings to workers-ai in config/ai.php, then:
$embedding = Str::of('custody schedule modification')->toEmbeddings();
// Returns 1024-dim float array (bge-large-en-v1.5 default)

// Batch embeddings:
$response = Embeddings::for(['text one', 'text two'])->generate('workers-ai');

// Store in pgvector (use saveQuietly, NOT updateQuietly):
$model->embedding = $embedding;
$model->saveQuietly();

// Semantic search (auto-embeds the query string):
$results = Model::query()
    ->whereVectorSimilarTo('embedding', 'search query here')
    ->limit(10)
    ->get();
```

### Rollback

Remove or comment out the `*_URL` env vars in `.env`. Requests fall back to provider default URLs. No code changes needed. For the native Workers AI provider, remove `gateway` from the config (or the `CLOUDFLARE_AI_GATEWAY` env var) to bypass the gateway and hit Workers AI directly.

---

## Reference Files

- `references/gateway-modes.md` — Detailed comparison of the four gateway authentication modes
- `references/workers-ai.md` — Workers AI endpoint details, driver selection, model list, JSON mode, limitations
- `references/troubleshooting.md` — Common errors and fixes
