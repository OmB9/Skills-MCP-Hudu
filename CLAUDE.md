# CLAUDE.md — MCP Hudu Server

You are Claude Code. Generate and maintain a production-ready **MCP server** that wraps the **Hudu** API from `hudu.json` (Swagger/OpenAPI).

## Objective
Ship an MVP MCP server that:
- Conforms to the **LATEST MCP specification** (always current version - NO legacy versions).
- **ALWAYS uses the LATEST official MCP TypeScript SDK**: @modelcontextprotocol/sdk (update immediately when new versions available)
- **EXCLUSIVELY uses Streamable HTTP Transport** on port 3100 - **NO STDIO, NO SSE, HTTP ONLY**.
- Implements the **resource+action** pattern (predictable verbs across domains).
- Runs reliably in **Docker**, with health/readiness endpoints.
- Handles **partial API access** gracefully (typed errors, warnings, next steps).
- Uses **.env** for all Hudu config (no secrets in code).
- **MANDATORY**: Always update to latest packages - security and compatibility requirement.

---

## Environment & Configuration

Create a `.env` file from `.env.example`:

```bash
HUDU_BASE_URL=https://your-hudu.example.com
HUDU_API_KEY=your-api-key
HUDU_TIMEOUT=30000
MCP_SERVER_PORT=3100
LOG_LEVEL=info
```

Fail fast if required vars are missing. No hard-coded secrets.

---

## Connection from Claude Code (HTTP ONLY)

**ONLY HTTP transport is supported - no STDIO configuration:**

Add MCP server to Claude Code:

```bash
claude mcp add --transport http hudu http://127.0.0.1:3100/mcp
```

Alternative Claude Desktop config:
```json
{
  "mcpServers": {
    "hudu": {
      "transport": "http",
      "url": "http://127.0.0.1:3100/mcp"
    }
  }
}
```

**IMPORTANT:** Never configure STDIO transport. This server only supports HTTP.

---

## Deliverables (what you should generate)

- `src/server.ts` — MCP HTTP server wiring (Streamable transport), health endpoints:
  - `GET /health` (liveness)
  - `GET /` (server info)
  - `ALL /mcp` (MCP endpoint)
- `src/hudu-client.ts` — Hudu API client with typed methods
- `src/tools/*` — 47 tools com prefixo `hudu_` e padrão `manage/search`:
  - **Core**: articles, companies, assets, passwords (manage + search = 8 tools)
  - **Procedures**: procedures, procedure tasks (manage + search = 4 tools)
  - **Folders**: kb article folders (manage + search = 2 tools)
  - **Networks**: networks, VLANs, VLAN zones, IP addresses (manage + search = 8 tools)
  - **Storage**: uploads, rack storages, rack items, public photos (manage + search = 8 tools)
  - **Extended**: expirations (1), websites (2), asset layouts (2), activity logs (1), relations (2), magic dash (2) = 10 tools
  - **MCPHub bridge**: list/get prompts + list/read resources (4 tools)
  - **Utility**: admin, search global, navigation (3 tools)
- `src/types.ts` — TypeScript types from OpenAPI
- `hudu.json` — OpenAPI/Swagger spec (reference only)
- `Dockerfile` — Multi-stage build, non-root runtime
- `.env.example` — Environment template
- `docker-compose.yml` — Quick local testing
- `package.json` — scripts: `dev`, `build`, `start`, `lint`, `type-check`

---

## Project Structure (current)

```
.
├─ hudu.json
├─ src/
│  ├─ server.ts                    # MCP HTTP server, OAuth proxy, JWT validation
│  ├─ index.ts                     # Entry point
│  ├─ hudu-client.ts               # Hudu API client com typed methods
│  ├─ filtered-hudu-client.ts      # Multi-tenant wrapper (HUDU_ALLOWED_COMPANY_IDS)
│  ├─ resources.ts                 # MCP Resources (6 hudu:// URIs)
│  ├─ prompts.ts                   # MCP Prompts definitions
│  ├─ types.ts                     # TypeScript types from OpenAPI
│  ├─ utils/
│  │  └─ html-stripper.ts          # HTML cleanup for API responses
│  ├─ formatters/
│  │  ├─ markdown.ts               # 29 Markdown table formatters
│  │  └─ response-formatter.ts     # Global response interceptor (Markdown output)
│  └─ tools/
│     ├─ working-index.ts          # Registro principal (47 tools)
│     ├─ index.ts                  # Registro secundário
│     ├─ base.ts                   # Helpers de resposta
│     ├─ schema-utils.ts           # Schemas compartilhados
│     ├─ articles.ts               # Artigos da base de conhecimento
│     ├─ companies.ts              # Empresas e organizações
│     ├─ assets.ts                 # Ativos de TI
│     ├─ passwords.ts              # Senhas e credenciais
│     ├─ procedures.ts             # Procedimentos e tarefas
│     ├─ folders.ts                # Pastas de organização
│     ├─ networks.ts               # Redes, VLANs, IPs
│     ├─ storage.ts                # Uploads, racks, fotos
│     ├─ expirations.ts            # Rastreamento de vencimentos
│     ├─ websites.ts               # Monitoramento de websites
│     ├─ asset-layouts.ts          # Templates de layouts de ativos
│     ├─ activity-logs.ts          # Logs de auditoria
│     ├─ relations.ts              # Relações entre entidades
│     ├─ magic-dash.ts             # Widgets do Magic Dash
│     ├─ admin.ts                  # Administração e auditoria
│     ├─ search.ts                 # Busca global unificada
│     └─ navigation.ts             # Navegação rápida por nome
├─ .env.example
├─ Caddyfile                       # Reverse proxy + security headers
├─ Dockerfile
├─ docker-compose.yml              # Caddy + MCP server (production)
├─ tsconfig.json
├─ package.json
└─ CLAUDE.md
```

---

## Coding Standards

- **Language:** TypeScript with `"strict": true` (latest stable TypeScript version).
- **SDK:** **ONLY** the latest official MCP TypeScript SDK - always update immediately when new versions release.
- **Transport:** **EXCLUSIVELY** Streamable HTTP Transport via `StreamableHTTPServerTransport` - **NEVER STDIO, NEVER SSE**.
- **Pattern:** `hudu_<verbo>_<domínio>` naming para tools (ex: `hudu_manage_company_information`).
- **Validation:** Zod schemas for inputs/outputs (latest version).
- **Dependencies:** **ALWAYS** use latest stable versions of all packages - no pinning to old versions.
- **Errors:** Map Hudu 401/403/404/429/5xx → typed MCP errors with:
  - `message` (user-readable)
  - `code` (stable)
  - `warnings[]` (if partial)
  - `nextSteps[]` (remediation hints)
- **Logging:** Winston with daily log rotation (combined, error, and API log files). Console output in development.
- **Resilience:** Graceful handling of API permission errors, continue with partial results.
- **Security:** Environment-based config, no secrets in code or images.

---

## Build & Run

**Local (Node):**
```bash
npm install
npm run build
npm start
# or dev:
npm run dev
```

**Docker (with Caddy, production):**
```bash
docker compose up --build -d
docker compose logs -f
```

**Docker (standalone, local testing only):**
```bash
docker build -t hudu-mcp-server .
docker run --rm --env-file .env -p 3100:3100 hudu-mcp-server
```

Health check:
- `GET http://localhost:3100/health` → 200 OK
- `GET http://localhost:3100/` → Server info

---

## Tool Contracts (47 tools registradas)

Todas as tools seguem as diretrizes de nomenclatura otimizadas para ToolRAG:
- **Prefixo**: `hudu_` (namespace do servidor)
- **Verbo**: `manage` (CRUD) ou `search` (consulta paginada)
- **Descrições**: pt-BR, substantivo-chave primeiro, 2-4 sinônimos de domínio
- **Parâmetros**: Todas as descrições em pt-BR

### Core Resources (8 tools)

| Tool Name | Tipo | Domínio |
|-----------|------|---------|
| `hudu_manage_knowledge_articles` | CRUD | Artigos, documentos, procedimentos |
| `hudu_search_knowledge_articles` | Query | Busca na base de conhecimento |
| `hudu_manage_company_information` | CRUD | Empresas, clientes, organizações |
| `hudu_search_company_information` | Query | Busca de empresas |
| `hudu_manage_it_asset_inventory` | CRUD | Ativos de TI, equipamentos, dispositivos |
| `hudu_search_it_asset_inventory` | Query | Busca no inventário de ativos |
| `hudu_manage_password_credentials` | CRUD | Senhas, credenciais, acessos |
| `hudu_search_password_credentials` | Query | Busca no cofre de senhas |

### Procedures (4 tools)

| Tool Name | Tipo | Domínio |
|-----------|------|---------|
| `hudu_manage_workflow_procedures` | CRUD | Procedimentos, rotinas, checklists |
| `hudu_search_workflow_procedures` | Query | Busca de procedimentos |
| `hudu_manage_procedure_task_items` | CRUD | Tarefas, etapas, passos |
| `hudu_search_procedure_task_items` | Query | Busca de tarefas |

### Folders (2 tools)

| Tool Name | Tipo | Domínio |
|-----------|------|---------|
| `hudu_manage_kb_article_folders` | CRUD | Pastas, diretórios, categorias |
| `hudu_search_kb_article_folders` | Query | Busca de pastas |

### Network Resources (8 tools)

| Tool Name | Tipo | Domínio |
|-----------|------|---------|
| `hudu_manage_network_documentation` | CRUD | Redes, sub-redes, segmentos |
| `hudu_search_network_documentation` | Query | Busca de redes |
| `hudu_manage_network_vlan_records` | CRUD | VLANs, segmentos virtuais |
| `hudu_search_network_vlan_records` | Query | Busca de VLANs |
| `hudu_manage_network_vlan_zones` | CRUD | Zonas de VLAN, perímetros |
| `hudu_search_network_vlan_zones` | Query | Busca de zonas |
| `hudu_manage_ip_address_records` | CRUD | Endereços IP, atribuições |
| `hudu_search_ip_address_records` | Query | Busca de IPs |

### Storage Resources (8 tools)

| Tool Name | Tipo | Domínio |
|-----------|------|---------|
| `hudu_manage_file_upload_records` | CRUD | Uploads, anexos, arquivos |
| `hudu_search_file_upload_records` | Query | Busca de uploads |
| `hudu_manage_rack_storage_locations` | CRUD | Racks, armários, datacenters |
| `hudu_search_rack_storage_locations` | Query | Busca de racks |
| `hudu_manage_rack_storage_items` | CRUD | Equipamentos montados em racks |
| `hudu_search_rack_storage_items` | Query | Busca de itens em racks |
| `hudu_manage_public_photo_gallery` | CRUD | Fotos públicas, imagens |
| `hudu_search_public_photo_gallery` | Query | Busca na galeria de fotos |

### Utility Tools (3 tools)

| Tool Name | Tipo | Domínio |
|-----------|------|---------|
| `hudu_admin_instance_operations` | Admin | Administração, auditoria, monitoramento |
| `hudu_search_all_resource_types` | Search | Busca global unificada |
| `hudu_navigate_to_resource_by_name` | Nav | Navegação rápida por nome |

### Extended Resources (10 tools)

| Tool Name | Tipo | Domínio |
|-----------|------|---------|
| `hudu_search_expiration_tracking` | Query | Vencimentos, expirations, renovações |
| `hudu_manage_website_monitoring` | CRUD | Websites, monitoramento, URLs |
| `hudu_search_website_monitoring` | Query | Busca de websites |
| `hudu_manage_asset_layout_templates` | CRUD | Templates de layouts, campos personalizados |
| `hudu_search_asset_layout_templates` | Query | Busca de templates |
| `hudu_search_activity_audit_logs` | Query | Logs de auditoria, atividades |
| `hudu_manage_entity_relations` | CRUD | Relações entre entidades |
| `hudu_search_entity_relations` | Query | Busca de relações |
| `hudu_manage_dashboard_widgets` | CRUD | Magic Dash widgets |
| `hudu_search_dashboard_widgets` | Query | Busca de widgets |

### MCPHub Bridge Tools (4 tools)

| Tool Name | Tipo | Domínio |
|-----------|------|---------|
| `hudu_list_prompts` | Bridge | Lista prompts MCP disponíveis |
| `hudu_get_prompt` | Bridge | Retorna texto de um prompt MCP |
| `hudu_list_resources` | Bridge | Lista recursos MCP (hudu:// URIs) |
| `hudu_read_resource` | Bridge | Lê conteúdo de um recurso MCP |

### Padrão de Input (tools manage)

```json
{
  "action": "create | get | update | delete",
  "id": 123,
  "fields": { "name": "...", "company_id": 1 }
}
```

### Padrão de Input (tools search)

```json
{
  "search": "texto de busca",
  "page": 1,
  "page_size": 25,
  "company_id": 1
}
```

---

## Error Model

```json
{
  "code": "UNAUTHORIZED | FORBIDDEN | NOT_FOUND | RATE_LIMITED | SERVER_ERROR",
  "message": "Human-readable summary",
  "warnings": ["Partial results due to permissions"],
  "nextSteps": ["Check HUDU_API_KEY", "Verify permissions"]
}
```

---

## Implementation Checklist

✅ **Completed:**
1. Project scaffolding com TypeScript strict
2. Environment configuration (.env)
3. MCP SDK integration com Streamable HTTP Transport
4. Hudu API client com typed methods
5. 47 tools com padrão `hudu_manage/search_*` otimizado para ToolRAG
6. Error handling para acesso parcial à API
7. Docker configuration com Caddy (reverse proxy + auto HTTPS)
8. Health endpoints (`/health`, `/`)
9. Nomenclatura de tools conforme diretrizes obrigatórias (substantivo-chave, sinônimos, pt-BR)
10. CORS, rate limiting e segurança de rede
11. OAuth 2.1 com Azure AD (proxy `/authorize`, `/token`, `/register` + JWT validation)
12. Static Bearer token como alternativa para Claude Code CLI
13. Winston logging com daily log rotation
14. MCP Resources (6 hudu:// URIs) e MCP Prompts
15. Multi-tenant support via `HUDU_ALLOWED_COMPANY_IDS`
16. Markdown-formatted responses (all tool outputs as tables)

⚠️ **Known Issues:**
- Password endpoints requerem permissões elevadas na API
- Alguns endpoints podem retornar 401 dependendo das permissões do usuário

---

## Testing Protocol

1. **Build**: `npm run build`
2. **Start**: `npm start` ou `docker compose up -d --build`
3. **Health check**: `curl http://localhost:3100/health`
4. **Configure Claude Code (HTTP ONLY)**: `claude mcp add --transport http hudu http://127.0.0.1:3100/mcp`
5. **Test tools**: Use `hudu_search_all_resource_types`, `hudu_search_company_information`, etc. no Claude Code
6. **NEVER configure STDIO** - Only HTTP transport is supported
7. **Verify logs**: `docker compose logs hudu-mcp-server` ou `cat logs/combined-*.log`

---

## Acceptance Criteria

- ✅ Starts with clear diagnostics if env is incomplete
- ✅ Health endpoint returns 200 OK
- ✅ Claude Code connects over HTTP transport
- ✅ Lists 47 tools via MCP protocol
- ✅ Successfully calls `hudu_search_all_resource_types` e demais tools
- ✅ Errors are typed and user-readable
- ✅ Handles partial API access gracefully
- ✅ Runs in Docker as non-root user

---

## Troubleshooting

- **401/403**: Verify `HUDU_API_KEY` and user permissions
- **Connection refused**: Verificar se Docker está rodando (`docker compose ps`)
- **No tools available**: Rebuild com `docker compose down && docker compose up -d --build`
- **Partial results**: Some endpoints require elevated permissions - this is expected

---

## Current Status

**Working:**
- 47 tools registradas e disponíveis (nomenclatura ToolRAG-optimized)
- Streamable HTTP transport na porta 3100
- OAuth 2.1 com Azure AD (Claude.ai web custom connectors)
- Static Bearer token (Claude Code CLI)
- Caddy reverse proxy com Let's Encrypt TLS automático
- Busca global unificada via `hudu_search_all_resource_types`
- Navegação rápida via `hudu_navigate_to_resource_by_name`
- Graceful error handling para permissões parciais
- MCP Resources (6 hudu:// URIs) e MCP Prompts
- Multi-tenant support via `HUDU_ALLOWED_COMPANY_IDS`
- Markdown-formatted responses em todas as tools
- Winston logging com daily rotation

**Diretrizes de Nomenclatura (2026-02):**
- Prefixo `hudu_` em todas as 47 tools
- Padrão `manage` (CRUD) e `search` (query paginada)
- Descrições em pt-BR com substantivo-chave primeiro
- 2-4 sinônimos de domínio nos primeiros 100 caracteres
- Todas as descrições de parâmetros em pt-BR
- Anti-boilerplate: frases compactas sem repetição
- Sem jargão interno da plataforma (ex: "card" substituído por "recurso")

**Latest requirements:**
- **ALWAYS** use latest MCP SDK version
- **EXCLUSIVELY** StreamableHTTPServerTransport - no STDIO support
- Single `/mcp` endpoint for all MCP communication
- Built-in session management with UUID
- **MANDATORY** latest package updates for security and compatibility
- Follow latest MCP specification from https://modelcontextprotocol.io/

---

**Build it, run the container, and test via Claude Code over HTTP. Keep SDK usage canonical, naming consistent, and errors helpful.**