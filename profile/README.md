# Flit-PKM

Personal knowledge management: markdown notes, categories, and typed links between notes, backed by a shared sync API. Clients include a web app and an Android app; agents can use MCP against the same data.

## Repositories

| Repo | Role |
|------|------|
| **flit_core** | FastAPI backend (Python ≥3.13), PostgreSQL, Alembic migrations. JWT auth, OAuth-style device pairing, versioned sync, notes/categories/relationships REST API, vault ZIP/`.md` import-export, MCP server (JSON-RPC tools + OAuth), Dodo Payments billing, public feedback/subscriptions (Turnstile). |
| **flit_web** | SvelteKit 2 + TypeScript static SPA (`adapter-static`). Account, notes editor, categories, profile, connected apps, billing. Built output is also vendored under `flit_core/webapp_build/` for same-origin hosting. |
| **flit_android** | Kotlin + Jetpack Compose (minSdk 31). Local Room DB, markdown notes, categories, typed relationships and wikilinks, ranked text search, optional on-device speech-to-text (Sherpa ONNX), sync to Flit Core. Import/export ZIP and share targets. |

Documentation and conventions live in each repo’s README and `AGENTS.md` files (no separate **flit-docs** repo today).

## Architecture

**Deployment unit:** one **flit_core** process + PostgreSQL. Optional static **flit_landing** (no backend coupling).

```
Clients                    flit_core                         Data
─────────                  ─────────                         ────
flit_web (SPA)     ──┐     /           ← webapp_build        PostgreSQL
flit_android       ──┼──►  /api/*     ← REST, JWT / OAuth    (notes, categories,
MCP agents         ──┘     /api/sync/* ← versioned merge       relationships, users,
                           /mcp*       ← JSON-RPC + OAuth      billing, …)
```

### Layers

| Layer | Component | Responsibility |
|-------|-----------|------------------|
| Presentation | **flit_web** | Static SPA; prerendered shell + client routing (`200.html`). |
| Presentation | **flit_android** | Compose UI over local **Room**; on-device search, STT, import/export. |
| API | **flit_core** `/api/*` | CRUD, auth, profile, vault, billing; OpenAPI is contract for web. |
| Sync | **flit_core** `/api/sync/*` | Connected-app merge: versions, soft deletes, `core_id` mapping. |
| Agents | **flit_core** `/mcp` | PKM tools over JSON-RPC; OAuth and API keys, separate route tree. |
| Data | PostgreSQL | Single source of truth for all surfaces. |

### Client models

- **Web — thin client.** JWT session → direct `/api` calls. No local graph replica.
- **Android — thick client.** Authoritative UI reads **Room**; **SyncRepository** pushes/pulls against `/api/sync` when a connected-app token exists. Local-only work stays offline.

### Auth boundaries

| Credential | Issued via | Used for |
|------------|------------|----------|
| User JWT | `/api/auth` (web login) | SPA, connect-code creation, MCP API key management |
| Connected-app OAuth | `/api/connect/exchange` (after profile code) | Android sync |
| MCP OAuth / API key | `/mcp/oauth/*`, `/mcp/api-keys` | Agent tool calls |

### Release

`flit_web` `npm run build` → copy to `flit_core/webapp_build/` → core mounts `/` (same origin as `/api` in production). Dev: core `:8000`, web dev server `:5173` + `VITE_API_BASE_URL`.

## Data model

- **Notes** — title, markdown body, versioning and soft delete for sync.
- **Categories** — many-to-many with notes.
- **Relationships** — directed edges between two notes with a fixed type: `FOLLOWS_ON`, `SIMILAR_TO`, `CONTRADICTS`, `REFERENCES`, `RELATED_TO`. Created explicitly (UI, API, MCP, or structured import); not inferred automatically in the backend today.

## Status

Active development. The three repos above are working codebases with tests and CI patterns documented locally — not a design-only phase.

Issues and discussions are welcome. Match contribution style to the target repo’s README and `AGENTS.md`.
