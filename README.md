# 9Router v2

> A self-hosted AI gateway — one endpoint, many providers, auto-fallback.

9Router v2 is a decoupled rewrite of [9Router](https://github.com/decolua/9router) with a clean separation between a dedicated **Express backend** and a **Vite + React frontend**. It exposes an OpenAI-compatible REST API that proxies requests across dozens of AI providers with automatic load balancing, fallback, and key rotation.

---

## Screenshots

| Login Page | Quota Tracker |
|:---:|:---:|
| ![Login Page](docs/screnshoot/login-page.png) | ![Quota Tracker](docs/screnshoot/quota-tracker.png) |

---

## Features

- **OpenAI-compatible API** — works with any client that supports `/v1/chat/completions`, `/v1/images/generations`, `/v1/audio/speech`, `/v1/embeddings`, etc.
- **Multi-provider routing** — Cloudflare Workers AI, OpenAI, Anthropic, Gemini, Groq, and many more
- **Cloudflare Workers AI Automation** — automates account registration and API key extraction with Playwright + 2Captcha + Ammail temp mail
- **Dashboard UI** — manage providers, connections, proxy pools, CLI tools, and automation from a modern dark-mode interface
- **OIDC / Password authentication** — single sign-on or local credentials
- **Agent Skills** — ready-to-use SKILL.md files for Claude, Gemini, Codex, and other AI coding agents
- **SQLite backend** — zero-dependency local database, no external services required

---

## Architecture

```
9router-v2/
├── backend/          # Express server (port 3001)
│   └── src/
│       ├── routes/   # Auto-routed endpoints (/v1, /api, /auth, ...)
│       ├── db/       # SQLite via better-sqlite3
│       └── automation/ # Playwright automation scripts
├── frontend/         # Vite + React SPA (port 5177)
│   └── src/
│       ├── pages/    # Dashboard pages
│       └── shared/   # Components, hooks, constants
└── skills/           # Agent SKILL.md files
```

---

## Quick Start

### Requirements

- Node.js 20+
- Python 3.10+ (for automation features)
- Chromium (for Playwright automation)

### Install

```bash
git clone https://github.com/ahwanulm/9router-v2.git
cd 9router-v2
npm install
```

### Development

```bash
npm run dev          # Start both backend + frontend concurrently
npm run backend      # Backend only (port 3001)
npm run frontend     # Frontend only (port 5177)
```

### Environment Variables

Copy and configure the backend environment:

```bash
cp backend/.env.template backend/.env
```

Key variables:

| Variable | Description |
|---|---|
| `PORT` | Backend server port (default: `3001`) |
| `REQUIRE_LOGIN` | Enable authentication (`true`/`false`) |
| `JWT_SECRET` | Secret for JWT signing |
| `ADMIN_PASSWORD` | Dashboard admin password |

---

## Production Deployment

### 1. Build Frontend

```bash
cd frontend && npm run build
```

### 2. Start Backend

```bash
cd backend && npm start
```

### 3. Nginx Reverse Proxy

Use the included `nginx.conf` to proxy `/api` and `/v1` to the backend while serving the built frontend statically. See `docs/deployment-linux.md` for a full Linux deployment guide.

---

## Cloudflare Workers AI Automation

The automation module automatically registers Cloudflare Workers AI accounts:

1. Configure **Ammail** temp mail credentials in Dashboard → Automation → Settings
2. Add a **2Captcha** API key for Turnstile solving
3. Add email accounts in the Automation tab and click **Run**
4. API keys are extracted and added to 9Router automatically

---

## Agent Skills

Skills are SKILL.md files for AI coding agents. Paste the entry skill URL into your AI:

```
Read this skill and use it:
https://raw.githubusercontent.com/ahwanulm/9router-v2/refs/heads/master/skills/9router/SKILL.md
```

Browse all skills in the Dashboard → Skills page or in the [`skills/`](./skills/) directory.

---

## Custom Changes

This version contains local patches for the VPS deployment, maintained by Nareida. The following changes were made on top of the 9Router v2 source and are backed up to this public snapshot.

### Upstream context metadata

- `/v1/models` now adds `context_length` for available LLM models.
- Metadata is fetched from provider model catalogs: OpenRouter, Groq, and custom OpenAI-compatible nodes.
- Provider catalogs are cached in SQLite for 24 hours; stale cache is served when an upstream refresh fails.
- Combo-reported context uses the largest value among its member models. This is not a guarantee that every fallback can accept a prompt of that size; models with smaller windows may still reject or be skipped.
- Providers that do not publish context metadata get no `context_length` field; no hardcoded fallback is used.

### Soul mode

- OpenAI-compatible providers with `soulMode: true` move the client's system message to the start of the first user message as an identity override.
- The goal is to prevent upstream default personas from replacing the SYSTEM/AGENTS prompt sent by the client.
- New connections inherit `soulMode` from the provider node. Existing connections need the flag enabled on their connection/provider-specific data.

### Provider validation and runtime fixes

- Validation endpoints use explicit timeout/abort and no longer leave requests hanging.
- Relative import fixes in several Open-SSE/backend modules for `tsx` compatibility; theme/shared imports remain a known FIXME.
- The backend serves the static frontend from `/opt/data/9router-dist` with SPA fallback and appropriate cache policies.
- The auth middleware only applies to API/LLM routes; frontend routes are not redirected to auth.

### Local verification

- `GET /v1/models` verified to return `context_length` for combos and models with available catalogs.
- `POST /v1/chat/completions` verified working through a combo.
- `git diff --check` passes.
- Theme/runtime imports and the full TypeScript check are not claimed as passing; the upstream snapshot still contains several import/runtime and type errors that need a separate audit.

### Repository and privacy

- This repository is public. Upstream OAuth client credentials (Gemini CLI, iFlow, and Antigravity) remain part of the upstream source; they are not Boss's runtime API keys, OAuth tokens, or database credentials.
- This README contains no tokens or secrets.

---

## API Reference

| Endpoint | Description |
|---|---|
| `GET /api/health` | Health check |
| `GET /v1/models` | List available chat/LLM models |
| `POST /v1/chat/completions` | Chat completions (streaming supported) |
| `POST /v1/images/generations` | Image generation |
| `POST /v1/audio/speech` | Text-to-speech |
| `POST /v1/audio/transcriptions` | Speech-to-text |
| `POST /v1/embeddings` | Text embeddings |
| `GET /v1/search` | Web search |

---

## License

MIT — see [LICENSE](./LICENSE)

---

## Acknowledgements

> 🙏 **Thanks to [9Router](https://github.com/decolua/9router)** — this project is a fork and architectural rewrite of the original 9Router monolith. The core routing logic, provider integrations, and many features were inspired by and built upon the excellent work of the 9Router project.
