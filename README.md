# laravel-cloudflare-ai-gateway

A Claude Code skill that configures Laravel and PHP projects to route AI traffic through [Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/) for observability, caching, and rate limiting. Also adds [Workers AI](https://developers.cloudflare.com/workers-ai/) as a free open-source model provider.

## What it does

- **Routes AI traffic through Cloudflare AI Gateway** — rewrites provider base URLs so all requests (OpenAI, Anthropic, Gemini, etc.) pass through your gateway for logging, caching, and rate limiting
- **Adds Workers AI as a free provider** — configures Cloudflare's free open-source models (Llama, Qwen, Mistral) as a provider in your Laravel app
- **Supports both frameworks** — works with [Laravel AI SDK](https://github.com/laravel/ai) (`laravel/ai`) and [Prism PHP](https://github.com/prism-php/prism) (`prism-php/prism`)

## Install

**Claude Code CLI:**
```bash
claude skill add --from meirdick/laravel-cloudflare-ai-gateway
```

**Laravel Cloud (Boost):**
```bash
php artisan boost:add-skill meirdick/laravel-cloudflare-ai-gateway
```

**Manual:**
Copy the `skills/laravel-cloudflare-ai-gateway/` directory into your Claude Code skills directory (`~/.claude/skills/` or `.claude/skills/` in your project).

## Prerequisites

- A Laravel or PHP project using `laravel/ai` or `prism-php/prism`
- A Cloudflare account with an [AI Gateway](https://dash.cloudflare.com/?to=/:account/ai/ai-gateway) created
- API keys for your existing AI providers (OpenAI, Anthropic, etc.)
- (Optional) A Cloudflare API token with `Workers AI: Read` permission for Workers AI

## Usage

Invoke the skill in any Laravel project:

```
/laravel-cloudflare-ai-gateway
```

The skill runs a guided 4-phase workflow: detect your project setup, ask configuration questions, present a plan for approval, then execute the changes.

## License

MIT
