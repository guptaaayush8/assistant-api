# BCM Copilot (bcm-ghcp)

AI-powered BCM/DR assessment tool using the Microsoft Agent Framework. Analyses Azure services, validates BCM requirements, and recommends Pilot Light DR strategies — guided step by step through a chat interface.

---

## Table of Contents

1. [Tech Stack](#tech-stack)
2. [Project Structure](#project-structure)
3. [Prerequisites](#prerequisites)
4. [Azure Resource Setup](#azure-resource-setup)
5. [App Registrations (Entra ID)](#app-registrations-entra-id)
6. [Backend Configuration](#backend-configuration)
7. [Frontend Configuration](#frontend-configuration)
8. [Azure MCP Server Configuration](#azure-mcp-server-configuration)
9. [Running Locally](#running-locally)
10. [Running with Docker](#running-with-docker)
11. [Deployment (Azure Container Apps)](#deployment-azure-container-apps)
12. [Testing](#testing)
13. [Workflow Stages](#workflow-stages)
14. [Adding a New Tool](#adding-a-new-tool)
15. [Adding a New Service](#adding-a-new-service)

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 19, TypeScript, Vite 6, Tailwind CSS 4, Zustand 5, MSAL React |
| Backend | Python 3.12+ (Docker uses 3.13), FastAPI, Pydantic, SSE streaming |
| AI | Microsoft Agent Framework (`agent-framework` >= 1.2.2), Azure OpenAI |
| Auth | Azure AD / Entra ID (MSAL SPA → JWT → OBO for downstream) |
| Data | Azure Cosmos DB (chat history + application configs) |
| Azure Tools | Azure MCP Server (Streamable HTTP, OBO auth) |
| Infra | Docker, nginx (frontend reverse proxy), uv (Python packaging) |

---

## Project Structure

```
bcm-ghcp/
├── package.json              # Root scripts (lint, typecheck)
├── backend/
│   ├── Dockerfile
│   ├── pyproject.toml        # Python deps (uv / pip compatible)
│   ├── uv.lock              # Lockfile for deterministic installs
│   ├── pytest.ini
│   ├── .env.example          # ← Copy to .env, fill in values
│   ├── app/
│   │   ├── config.py         # All settings (env-driven, Pydantic)
│   │   ├── main.py           # FastAPI app (/api/chat, /api/health, etc.)
│   │   ├── agent/
│   │   │   ├── client.py     # Agent Framework client + session mgmt
│   │   │   ├── tools.py      # BCM tools (@tool decorator)
│   │   │   └── prompts.py    # System prompt + stage appendices
│   │   ├── services/
│   │   │   ├── auth.py       # JWT verification (JWKS, audience, scope)
│   │   │   ├── azure_mcp.py  # MCP server HTTP client (JSON-RPC 2.0)
│   │   │   ├── azure_token.py # OBO token exchange
│   │   │   ├── azure_discovery.py # Subscription/resource resolution
│   │   │   ├── application.py # App config CRUD (Cosmos DB)
│   │   │   ├── cmdb.py       # CMDB/SMDB metadata
│   │   │   └── pilot_light.py # DR strategy engine
│   │   └── models/
│   │       └── schemas.py    # Pydantic models
│   ├── database/             # Static knowledgebase JSON files
│   └── tests/
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf            # Reverse proxy config (envsubst-based)
│   ├── package.json
│   ├── vite.config.ts        # Dev proxy → backend:8000
│   └── src/
│       ├── auth/
│       │   ├── msalConfig.ts # MSAL client/tenant/scope config
│       │   └── authProvider.ts # Token acquisition helpers
│       ├── store/            # Zustand state management
│       └── components/       # React UI components
├── mcp_servers/
│   └── azure_mcp/
│       └── Dockerfile        # Production MCP server container
└── docs/
    ├── DESIGN.md
    ├── UI_DESIGN.md
    └── BACKLOG.md
```

---

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Node.js | 18+ | Frontend build, MCP dev server |
| Python | 3.12+ | Backend runtime |
| uv | 0.5+ | Python dependency management (or use pip) |
| Azure CLI | Latest | `az login` for local credential chain |
| Docker | 20+ | Container builds (optional for local dev) |

---

## Azure Resource Setup

You need the following Azure resources provisioned before the app can function:

| Resource | Purpose | Required Settings |
|----------|---------|-------------------|
| **Azure OpenAI** | LLM for the agent | Endpoint URL + a deployed model (e.g. `gpt-4.1`) |
| **Azure Cosmos DB** (NoSQL API) | Chat history + application configs | Database: `chats`, Containers: `session_history`, `applications` |
| **Azure MCP Server** (Container App) | Azure resource discovery tools | Streamable HTTP transport with Entra auth |

### Cosmos DB Setup

1. Create an Azure Cosmos DB account (NoSQL API).
2. Create a database named `chats` (or your chosen name).
3. Create two containers:
   - `session_history` — partition key: `/session_id`
   - `applications` — partition key: `/application_name`
4. Grant the backend's managed identity (or your local `az login` identity) the **Cosmos DB Built-in Data Contributor** role.

### Azure OpenAI Setup

1. Create an Azure OpenAI resource.
2. Deploy a model (e.g. `gpt-4.1`) — note the **deployment name** (not the model name).
3. Grant the backend's identity the **Cognitive Services OpenAI User** role on the resource.

---

## App Registrations (Entra ID)

The system uses **three** Entra ID app registrations for end-to-end authenticated access:

### 1. `bcm-frontend` (SPA)

| Setting | Value |
|---------|-------|
| Platform | Single-page application |
| Redirect URIs | `http://localhost:5173`, `https://<your-frontend-url>` |
| Supported account types | Single tenant |
| API permissions | `bcm-api` → `Access` (delegated) |

Note the **Application (client) ID** — this becomes `VITE_MSAL_CLIENT_ID` in the frontend and `AUTH_FRONTEND_CLIENT_ID` in the backend.

### 2. `bcm-api` (Backend API)

| Setting | Value |
|---------|-------|
| Platform | Web (confidential client) |
| Expose an API | App ID URI: `api://<bcm-api-client-id>` |
| Scopes exposed | `Access` (delegated, admin consent not required) |
| Client secret | Create one for OBO (store securely) |
| API permissions | `azmcp-server` → `Mcp.Tools.ReadWrite` (delegated) |
| Token version | `requestedAccessTokenVersion: 2` in the manifest |

Note the **Application (client) ID** → `AUTH_API_CLIENT_ID`, and the **client secret** → `AUTH_API_CLIENT_SECRET`.

### 3. `azmcp-server` (Azure MCP Server)

| Setting | Value |
|---------|-------|
| Platform | Web (confidential client) |
| Expose an API | App ID URI: `api://<azmcp-server-client-id>` |
| Scopes exposed | `Mcp.Tools.ReadWrite` (delegated) |
| Client secret | (or federated credential for Container Apps) |

The full scope URI is: `api://<azmcp-server-client-id>/Mcp.Tools.ReadWrite` → this becomes `MCP_SCOPE` in the backend.

### Token Flow Diagram

```
┌─────────┐  Bearer token (aud=bcm-api)   ┌─────────┐  OBO token (aud=azmcp)  ┌───────────┐
│ Frontend │ ─────────────────────────────► │ Backend │ ──────────────────────► │ MCP Server│
│  (SPA)   │                                │  (API)  │                         │  (azmcp)  │
└─────────┘                                └─────────┘                         └───────────┘
     │                                          │
     │ MSAL acquires token with                 │ Validates JWT (aud, iss, scp, azp, tid)
     │ scope: api://<bcm-api>/Access            │ Then OBO exchanges for MCP-audience token
     │                                          │ using AUTH_API_CLIENT_SECRET
```

---

## Backend Configuration

Copy `backend/.env.example` → `backend/.env` and fill in all values.

### Complete Environment Variables Reference

#### Authentication (Required for production)

| Variable | Default | Description |
|----------|---------|-------------|
| `AUTH_TENANT_ID` | (empty) | Azure AD tenant GUID |
| `AUTH_API_CLIENT_ID` | (empty) | `bcm-api` app registration client ID |
| `AUTH_API_CLIENT_SECRET` | (empty) | `bcm-api` client secret (for OBO flow) |
| `AUTH_FRONTEND_CLIENT_ID` | (empty) | `bcm-frontend` app registration client ID (validated via `azp` claim) |
| `AUTH_REQUIRED_SCOPE` | `Access` | Required scope on incoming tokens (in the `scp` claim) |

#### Azure OpenAI (Required)

| Variable | Default | Description |
|----------|---------|-------------|
| `AZURE_OPENAI_ENDPOINT` | (empty) | Azure OpenAI resource URL, e.g. `https://myaoai.openai.azure.com/` |
| `AZURE_OPENAI_DEPLOYMENT` | (empty) | The **deployment name** (not model name) in your Azure OpenAI resource |
| `AZURE_OPENAI_API_VERSION` | `2024-10-21` | Azure OpenAI API version |

#### Azure MCP Server (Required for Azure resource tools)

| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_SERVER_URL` | (empty) | URL of the Azure MCP server, e.g. `https://mymcp.azurecontainerapps.io` |
| `MCP_SCOPE` | (empty) | OBO target scope: `api://<azmcp-server-client-id>/Mcp.Tools.ReadWrite` |

#### Cosmos DB (Required for persistence)

| Variable | Default | Description |
|----------|---------|-------------|
| `COSMOS_ENDPOINT` | (empty) | Cosmos DB account endpoint, e.g. `https://myaccount.documents.azure.com:443/` |
| `COSMOS_DATABASE` | `chats` | Database name |
| `COSMOS_CONTAINER` | `session_history` | Container for chat history |
| `COSMOS_APPLICATIONS_CONTAINER` | `applications` | Container for application configs |

#### Azure Resources

| Variable | Default | Description |
|----------|---------|-------------|
| `AZURE_RESOURCE_GROUP` | (empty) | Default Azure resource group for scoping assessments |

#### Server Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `HOST` | `0.0.0.0` | Server bind address |
| `PORT` | `8000` | Server port |
| `LOG_LEVEL` | `INFO` | Logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |
| `CORS_ORIGINS` | `["http://localhost:5173"]` | JSON array of allowed CORS origins |
| `RATE_LIMIT_WINDOW` | `60` | Rate limit window in seconds |
| `RATE_LIMIT_MAX` | `20` | Max requests per user per window |
| `USER_IDENTIFIER` | (hostname) | Override the machine/user identifier |

### Local Development (Auth Bypass)

When running locally (no `WEBSITE_SITE_NAME` or `CONTAINER_APP_NAME` env var detected), the backend logs warnings for missing config but **does not fail**. Auth validation still requires the variables above — if they're empty, every request will get a 401. For fully local testing without Azure AD, you need to set valid auth variables or temporarily bypass auth in development.

---

## Frontend Configuration

The frontend uses build-time environment variables prefixed with `VITE_`.

### Environment Variables

| Variable | Default (hardcoded fallback) | Description |
|----------|------|-------------|
| `VITE_MSAL_CLIENT_ID` | `8db16281-12a3-4fde-aa27-e91803f01a76` | `bcm-frontend` app registration client ID |
| `VITE_MSAL_TENANT_ID` | `d1d6dae2-2b71-414c-955c-230dc4440ef9` | Azure AD tenant ID |
| `VITE_MSAL_BACKEND_SCOPE` | `api://998c342f-2c85-4111-bdc7-5adbb50f021e/Access` | Backend API scope URI |

### Setting Frontend Env Vars

Create a `.env` file in the `frontend/` directory (Vite loads it automatically):

```env
VITE_MSAL_CLIENT_ID=<bcm-frontend-client-id>
VITE_MSAL_TENANT_ID=<your-tenant-id>
VITE_MSAL_BACKEND_SCOPE=api://<bcm-api-client-id>/Access
```

### Dev Proxy

The Vite dev server (`frontend/vite.config.ts`) proxies all `/api/*` requests to `http://localhost:8000`. No changes needed for local development.

---

## Azure MCP Server Configuration

The Azure MCP server provides tools for querying Azure resources (list subscriptions, resource groups, resources, pricing, etc.).

### Local Development (No Auth)

Run in a separate terminal:

```bash
npx @azure/mcp@latest server start --transport http --read-only --dangerously-disable-http-incoming-auth
```

This starts on `http://localhost:5000`. Set in `backend/.env`:

```env
MCP_SERVER_URL=http://localhost:5000
MCP_SCOPE=
```

> **Warning**: `--dangerously-disable-http-incoming-auth` disables all JWT validation. Never use this in production.

### Production (Container App with Entra Auth)

Use the Dockerfile at `mcp_servers/azure_mcp/Dockerfile`. The container expects these environment variables at runtime:

| Variable | Description |
|----------|-------------|
| `AzureAd__TenantId` | Entra ID tenant GUID |
| `AzureAd__ClientId` | `azmcp-server` app registration client ID |
| `AzureAd__ClientSecret` | `azmcp-server` client secret (or use federated credential) |
| `AzureAd__Audience` | `api://<azmcp-server-client-id>` |

The container runs:
```
azmcp server start --transport http --mode all --read-only --outgoing-auth-strategy UseOnBehalfOf
```

This means:
- **Incoming auth**: Validates Bearer JWTs against `AzureAd__*` config
- **Outgoing auth (UseOnBehalfOf)**: Exchanges the validated user token for downstream Azure ARM tokens, preserving per-user RBAC

Build and push:
```bash
cd mcp_servers/azure_mcp
docker build -t <your-acr>.azurecr.io/azure-mcp:latest .
docker push <your-acr>.azurecr.io/azure-mcp:latest
```

---

## Running Locally

### 1. Start the Azure MCP Server (optional, but needed for Azure tools)

```bash
npx @azure/mcp@latest server start --transport http --read-only --dangerously-disable-http-incoming-auth
```

### 2. Start the Backend

```bash
cd backend

# Option A: using uv (recommended, matches Docker build)
uv sync --dev
cp .env.example .env    # ← Edit with your values
uv run uvicorn app.main:app --reload --port 8000

# Option B: using pip
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -e ".[dev]"
cp .env.example .env    # ← Edit with your values
uvicorn app.main:app --reload --port 8000
```

### 3. Start the Frontend

```bash
cd frontend
npm install
npm run dev    # → http://localhost:5173
```

### 4. Verify

- Health check: `GET http://localhost:8000/api/health` → `{"status": "ok"}`
- Open `http://localhost:5173` → should show login page (MSAL redirect)

---

## Running with Docker

### Backend

```bash
cd backend
docker build -t bcm-ghcp-backend .
docker run -p 8000:8000 \
  -e AUTH_TENANT_ID=<tenant-id> \
  -e AUTH_API_CLIENT_ID=<bcm-api-client-id> \
  -e AUTH_API_CLIENT_SECRET=<secret> \
  -e AUTH_FRONTEND_CLIENT_ID=<bcm-frontend-client-id> \
  -e AZURE_OPENAI_ENDPOINT=https://<aoai>.openai.azure.com/ \
  -e AZURE_OPENAI_DEPLOYMENT=<deployment-name> \
  -e COSMOS_ENDPOINT=https://<cosmos>.documents.azure.com:443/ \
  -e MCP_SERVER_URL=https://<mcp-url> \
  -e MCP_SCOPE=api://<azmcp-client-id>/Mcp.Tools.ReadWrite \
  -e CORS_ORIGINS='["https://<frontend-url>"]' \
  bcm-ghcp-backend
```

### Frontend

```bash
cd frontend
docker build -t bcm-ghcp-frontend \
  --build-arg VITE_MSAL_CLIENT_ID=<bcm-frontend-client-id> \
  --build-arg VITE_MSAL_TENANT_ID=<tenant-id> \
  --build-arg VITE_MSAL_BACKEND_SCOPE=api://<bcm-api-client-id>/Access \
  .
docker run -p 80:80 \
  -e BACKEND_ORIGIN=http://backend:8000 \
  bcm-ghcp-frontend
```

> **Note**: The frontend nginx config uses `envsubst` to inject `BACKEND_ORIGIN` at runtime for the `/api/` reverse proxy.

### MCP Server

```bash
cd mcp_servers/azure_mcp
docker build -t azure-mcp .
docker run -p 8080:8080 \
  -e AzureAd__TenantId=<tenant-id> \
  -e AzureAd__ClientId=<azmcp-client-id> \
  -e AzureAd__ClientSecret=<azmcp-secret> \
  -e AzureAd__Audience=api://<azmcp-client-id> \
  azure-mcp
```

---

## Deployment (Azure Container Apps)

Recommended topology: 3 Container Apps in the same environment.

| Container App | Image | Ingress | Port |
|---------------|-------|---------|------|
| `bcm-backend` | `bcm-ghcp-backend` | Internal + External | 8000 |
| `bcm-frontend` | `bcm-ghcp-frontend` | External | 80 |
| `bcm-mcp` | `azure-mcp` | Internal | 8080 |

### Key deployment settings:

1. **Backend** needs a managed identity with:
   - Cosmos DB Built-in Data Contributor on the Cosmos account
   - Cognitive Services OpenAI User on the Azure OpenAI resource
   - Set all `AUTH_*`, `AZURE_OPENAI_*`, `COSMOS_*`, `MCP_*` env vars as secrets

2. **Frontend** is static — just set `BACKEND_ORIGIN` to the backend's internal FQDN (`https://bcm-backend.internal.<env>.azurecontainerapps.io`). MSAL vars are baked in at build time.

3. **MCP Server** needs:
   - `AzureAd__*` env vars for incoming JWT validation
   - The managed identity (or service principal) needs **Reader** on target subscriptions for Azure resource discovery

### ACR Push Example

```bash
# Tag and push all images
docker build -t bcmpilotlight.azurecr.io/bcm-ghcp-backend:latest ./backend
docker build -t bcmpilotlight.azurecr.io/bcm-ghcp-frontend:latest ./frontend
docker build -t bcmpilotlight.azurecr.io/azure-mcp:latest ./mcp_servers/azure_mcp

docker push bcmpilotlight.azurecr.io/bcm-ghcp-backend:latest
docker push bcmpilotlight.azurecr.io/bcm-ghcp-frontend:latest
docker push bcmpilotlight.azurecr.io/azure-mcp:latest
```

---

## Testing

### Backend Tests

```bash
cd backend
uv run pytest           # or: pytest (if venv activated)
uv run ruff check       # linting
```

Tests use `pytest` with `asyncio_mode = auto`. See `backend/tests/` for available tests.

### Frontend Type Check

```bash
cd frontend
npx tsc --noEmit
```

### Root-Level Check (both)

```bash
npm test    # Runs tsc on frontend + ruff on backend
```

---

## Workflow Stages

The app guides users through 6 stages (stages 4–6 are planned):

| # | Stage | Description |
|---|-------|-------------|
| 1 | **Analyse my service** | Azure resource discovery, cost overview |
| 2 | **Set RTO & RPO** | CMDB/SMDB validation, tier confirmation, RPO setting |
| 3 | **Adopt Pilot Light** | Fit-gap analysis and DR recommendations |
| 4 | **Cost & approval** | *(planned)* |
| 5 | **BCM test** | *(planned)* |
| 6 | **Go to production** | *(planned)* |

---

## Adding a New Tool

1. Define a Pydantic params model in `backend/app/agent/tools.py`
2. Create the tool function decorated with `@tool`
3. Use `_emit_progress()` for thinking steps and `_emit_component()` for UI (tables, badges, context)
4. Add the tool to `ALL_TOOLS` list at the bottom of the file
5. The frontend renders components automatically — no frontend changes needed

---

## Adding a New Service

1. Create `backend/app/services/your_service.py`
2. Implement the service logic (call Azure APIs, Cosmos DB, etc.)
3. Call your service from a tool in `tools.py`
4. If needed, add configuration variables to `backend/app/config.py` (the `Settings` class)
