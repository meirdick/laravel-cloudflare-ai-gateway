---
name: laravel-cloudflare-ai-gateway
description: >-
  Configures Laravel and PHP projects to route AI traffic through Cloudflare AI
  Gateway for observability, caching, and rate limiting. Adds Workers AI as a
  free open-source model provider. Supports Laravel AI SDK (laravel/ai) and
  Prism PHP (prism-php/prism). Use when the user mentions Cloudflare AI Gateway,
  Workers AI, AI observability, or wants to proxy AI provider traffic.
license: MIT
user_invocable: true
compatibility: Requires a Laravel or PHP project with laravel/ai or prism-php/prism. Needs a Cloudflare account.
metadata:
  author: mdick85
  version: "1.0.0"
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

Explain: "Workers AI runs free open-source models (Llama, Qwen, Mistral, etc.) on Cloudflare's edge network. Free tier includes 10,000 neurons/day. It's great for chatbots, summaries, and internal tools where you don't need GPT-4/Claude-level quality."

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
| Workers AI | `/compat` | N/A (Cloudflare-native) |

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

**CRITICAL: Use `xai` driver, NOT `openai`.** The `openai` driver uses `/responses` which Workers AI doesn't support — it causes 500 errors. See `references/workers-ai.md` for details.

**Laravel AI SDK** (`config/ai.php`):
```php
'workers-ai' => [
    'driver' => 'xai',
    'key' => env('CLOUDFLARE_AI_API_TOKEN'),
    'url' => env('WORKERS_AI_URL'),
],
```

**Prism PHP** (`config/prism.php`):
```php
'workers-ai' => [
    'api_key' => env('CLOUDFLARE_AI_API_TOKEN', ''),
    'url' => env('WORKERS_AI_URL'),
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

If Workers AI was requested, also append:
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

#### Step 6: Report

After completing all changes, report:

1. **What was detected** — framework, config file, existing providers
2. **What was changed** — list each file and modification
3. **Manual steps remaining:**
   - If Workers AI: "Add your Cloudflare API token as `CLOUDFLARE_AI_API_TOKEN` in `.env`"
   - Run `php artisan config:clear` if config is cached
   - Verify in Cloudflare dashboard (AI > AI Gateway > [gateway-name] > Analytics)

---

## Usage Examples

### Workers AI in Laravel AI SDK

```php
// Using default provider (if set to workers-ai)
$response = agent(instructions: 'You are helpful.')->prompt('Hello!');

// Explicit provider and model
$response = agent(instructions: 'You are helpful.')
    ->prompt('Hello!', provider: 'workers-ai', model: 'workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast');

// Via attributes
#[Provider('workers-ai')]
#[Model('workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast')]
```

### Workers AI in Prism PHP

```php
use Prism\Prism\Facades\Prism;

Prism::text()
    ->using('xai', 'workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast')
    ->usingProviderConfig('xai', config('prism.providers.workers-ai'))
    ->withPrompt('Hello!')
    ->asText();
```

### Rollback

Remove or comment out the `*_URL` env vars in `.env`. Requests fall back to provider default URLs. No code changes needed.

---

## Reference Files

- `references/gateway-modes.md` — Detailed comparison of the four gateway authentication modes
- `references/workers-ai.md` — Workers AI endpoint details, driver selection, model list, JSON mode, limitations
- `references/troubleshooting.md` — Common errors and fixes
