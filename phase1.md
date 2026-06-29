# Enterprise Agentic AI Platform — Phase 1 Plan of Action

**Organisation:** Fichtner GmbH & Co. KG  
**Scope:** Phase 1 — Email Reading, Summarisation & Calendar View  
**Author:** Architecture & Implementation Plan  
**Status:** Ready for Review

-----

## 1. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER (Your Browser)                         │
│              Corporate Microsoft Account Login                   │
└───────────────────────────┬─────────────────────────────────────┘
                            │  HTTPS
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              LibreChat UI  (Team's Docker Infra)                │
│                                                                  │
│  • Agent Builder — selects MCP tools                            │
│  • MCP Settings Panel — registers your MCP server URL           │
│  • OAuth flow — Microsoft login popup                           │
│  • Sends requests to Requesty.AI for LLM reasoning              │
│  • Injects headers: X-User-Email, X-User-ID into MCP calls      │
└────────────┬──────────────────────────────┬─────────────────────┘
             │                              │
             │  Streamable HTTP             │  HTTPS
             │  (POST /mcp)                 │  (LLM API calls)
             ▼                              ▼
┌────────────────────────┐    ┌─────────────────────────────────┐
│  YOUR VM               │    │  Requesty.AI Gateway            │
│  FastMCP Python Server │    │                                 │
│  HOST: 0.0.0.0         │    │  Primary:  azure/gpt-4.1-nano   │
│  PORT: 8000            │    │           @swedencentral        │
│                        │    │  Fallback: mistral-small-2603   │
│  /mcp  ← endpoint      │    │                                 │
│                        │    │  OpenAI-compatible API          │
│  Tools exposed:        │    └─────────────────────────────────┘
│  • list_emails         │
│  • read_email          │
│  • search_emails       │
│  • summarise_email     │
│  • list_calendar_events│
│  • get_my_profile      │
└────────────┬───────────┘
             │
             │  msgraph-sdk-python
             │  Delegated OAuth2 token
             │  (per-user, from LibreChat)
             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Microsoft Graph API                                 │
│         https://graph.microsoft.com/v1.0/                       │
│                                                                  │
│  /me                    → User profile                          │
│  /me/messages           → Read emails                           │
│  /me/messages/{id}      → Read single email                     │
│  /me/calendarView       → Calendar events                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              Microsoft 365                                       │
│         Outlook (your mailbox) · Calendar (your events)         │
│              Fichtner GmbH & Co. KG Tenant                      │
└─────────────────────────────────────────────────────────────────┘
```

-----

## 2. Technology Framework

|Layer        |Technology             |Purpose                                         |
|-------------|-----------------------|------------------------------------------------|
|UI           |LibreChat              |Chat interface, Agent builder, MCP tool selector|
|LLM Gateway  |Requesty.AI            |Routes to Azure GPT-4.1-nano or Mistral-small   |
|MCP Framework|FastMCP (Python)       |Exposes tools over Streamable HTTP              |
|Auth         |MSAL Python + OAuth2   |Delegated token from LibreChat → Graph API      |
|Graph Client |msgraph-sdk-python     |Calls Microsoft Graph API                       |
|Language     |Python 3.11+           |All server-side code                            |
|Transport    |Streamable HTTP        |MCP protocol between LibreChat and your VM      |
|Infra        |Windows 11 VM + VS Code|Development and runtime host                    |

-----

## 3. Folder Structure

```
agentic-ai-platform/
│
├── README.md
├── .env                          ← secrets (never commit this)
├── .env.example                  ← template for .env
├── requirements.txt              ← Python dependencies
│
├── config/
│   └── settings.py               ← central config loader (reads .env)
│
├── auth/
│   ├── __init__.py
│   └── graph_auth.py             ← MSAL token acquisition & refresh
│
├── graph/
│   ├── __init__.py
│   ├── mail_client.py            ← Graph API calls for email
│   └── calendar_client.py        ← Graph API calls for calendar
│
├── tools/
│   ├── __init__.py
│   ├── email_tools.py            ← MCP tools: list, read, search emails
│   ├── calendar_tools.py         ← MCP tools: list calendar events
│   └── profile_tools.py          ← MCP tools: get user profile
│
├── server.py                     ← FastMCP server entry point
│
└── tests/
    ├── test_auth.py
    ├── test_mail.py
    └── test_calendar.py
```

-----

## 4. Phase 1 — MCP Tools to Build

|#|Tool Name             |Description                                  |Graph API Endpoint           |Permission      |
|-|----------------------|---------------------------------------------|-----------------------------|----------------|
|1|`get_my_profile`      |Get logged-in user’s name and email          |`GET /me`                    |User.Read ✅     |
|2|`list_emails`         |List recent emails with subject, sender, date|`GET /me/messages`           |Mail.Read ✅     |
|3|`read_email`          |Read full body of a specific email by ID     |`GET /me/messages/{id}`      |Mail.Read ✅     |
|4|`search_emails`       |Search emails by keyword or sender           |`GET /me/messages?$search=`  |Mail.Read ✅     |
|5|`summarise_email`     |Summarise email thread using LLM             |Uses read_email + Requesty.AI|Mail.Read ✅     |
|6|`list_calendar_events`|List upcoming calendar events                |`GET /me/calendarView`       |Calendars.Read ✅|


> All 6 tools use already-granted permissions. No admin action needed for Phase 1.

-----

## 5. Auth Flow — How Delegated OAuth Works (First Principles)

```
Step 1: You open LibreChat in your browser and log in.

Step 2: LibreChat detects your MCP server requires OAuth.
        It shows an "Authenticate" button in MCP Settings.

Step 3: You click Authenticate. LibreChat opens a Microsoft
        login popup using your Azure App Registration's
        Client ID and Tenant ID.

Step 4: You log in with your corporate Microsoft account.
        Microsoft checks you are authorised for the app.

Step 5: Microsoft redirects back to LibreChat's callback URL:
        https://<librechat-domain>/api/mcp/outlook-ai/oauth/callback

Step 6: LibreChat exchanges the auth code for an access token
        and a refresh token. Stores them securely per-user.

Step 7: For every tool call, LibreChat passes the access token
        to your FastMCP server in the Authorization header.

Step 8: Your FastMCP server reads the token from the header
        and passes it directly to msgraph-sdk-python.
        No credentials stored on your VM.

Step 9: Graph API validates the token, confirms permissions,
        and returns the requested data.

Step 10: FastMCP formats the response and returns it to
         LibreChat → LLM reasons over it → shown to you.
```

**Key point:** Your VM never stores any credentials. The OAuth token lives in LibreChat and is sent fresh with every request.

-----

## 6. Sequential Implementation Steps

### Step 6.1 — Environment Setup (Your VM)

- [ ] Install Python 3.11+ from python.org (check “Add to PATH”)
- [ ] Install Git from git-scm.com
- [ ] Open VS Code, open terminal
- [ ] Create project folder: `mkdir agentic-ai-platform && cd agentic-ai-platform`
- [ ] Create virtual environment: `python -m venv venv`
- [ ] Activate it: `venv\Scripts\activate` (Windows)
- [ ] Install dependencies: `pip install fastmcp msal msgraph-sdk python-dotenv`
- [ ] Create `.env` file with Azure App Registration credentials

### Step 6.2 — Azure App Registration Configuration

- [ ] Confirm Client ID, Tenant ID, Client Secret from Azure Portal
- [ ] Add LibreChat OAuth callback URL to App Registration:
  `https://<librechat-domain>/api/mcp/outlook-ai/oauth/callback`
- [ ] Verify redirect URI is saved in Azure Portal → Authentication tab
- [ ] Request admin to grant consent for `Mail.ReadWrite` and `Mail.Send`

### Step 6.3 — Build Auth Module (`auth/graph_auth.py`)

- [ ] Implement token extraction from incoming request headers
- [ ] Implement MSAL confidential client for token validation
- [ ] Test: can you get a valid token and call `/me`?

### Step 6.4 — Build Graph Clients

- [ ] `graph/mail_client.py` — functions: `get_messages()`, `get_message_by_id()`, `search_messages()`
- [ ] `graph/calendar_client.py` — functions: `get_calendar_events()`
- [ ] Test each function independently before wrapping in MCP tools

### Step 6.5 — Build FastMCP Tools

- [ ] `tools/profile_tools.py` — `get_my_profile` tool
- [ ] `tools/email_tools.py` — `list_emails`, `read_email`, `search_emails`, `summarise_email` tools
- [ ] `tools/calendar_tools.py` — `list_calendar_events` tool
- [ ] Each tool: input validation, error handling, clean JSON output

### Step 6.6 — Build FastMCP Server (`server.py`)

- [ ] Initialise FastMCP app with `transport="streamable-http"`
- [ ] Register all tools from tools/ modules
- [ ] Set host to `0.0.0.0` and port `8000`
- [ ] Add startup health-check endpoint `/health`
- [ ] Test locally: `python server.py`

### Step 6.7 — Local Testing

- [ ] Run server: `python server.py`
- [ ] Test health: `curl http://localhost:8000/health`
- [ ] Test MCP endpoint: `curl -X POST http://localhost:8000/mcp`
- [ ] Verify tool list is returned correctly

### Step 6.8 — Connect to LibreChat

- [ ] Get your VM’s internal IP address: `ipconfig` in terminal
- [ ] Go to LibreChat → MCP Settings → + Add Server
- [ ] Fill in:
  - Name: `outlook-ai`
  - URL: `http://<your-vm-ip>:8000/mcp`
  - Transport: Streamable HTTP
  - Auth: OAuth
- [ ] Click Create → Click Authenticate → Complete Microsoft login
- [ ] Verify green “Connected” status in LibreChat

### Step 6.9 — End-to-End Test in LibreChat

- [ ] Open chat, select your MCP server from dropdown
- [ ] Test prompt: “List my last 5 emails”
- [ ] Test prompt: “Summarise the latest email from [sender]”
- [ ] Test prompt: “What meetings do I have this week?”
- [ ] Verify responses are accurate and well-formatted

-----

## 7. .env File Structure

```env
# Azure App Registration
AZURE_CLIENT_ID=your-client-id-here
AZURE_CLIENT_SECRET=your-client-secret-here
AZURE_TENANT_ID=your-tenant-id-here

# MCP Server
MCP_HOST=0.0.0.0
MCP_PORT=8000

# Requesty.AI
REQUESTY_API_KEY=your-requesty-key-here
REQUESTY_BASE_URL=https://router.requesty.ai/v1
PRIMARY_MODEL=azure/gpt-4.1-nano@swedencentral
```

-----

## 8. LibreChat MCP Config (for your team’s librechat.yaml — later)

```yaml
mcpServers:
  outlook-ai:
    type: streamable-http
    url: http://<your-vm-ip>:8000/mcp
    headers:
      X-User-Email: '{{LIBRECHAT_USER_EMAIL}}'
      X-User-ID: '{{LIBRECHAT_USER_ID}}'
    timeout: 30000
    initTimeout: 15000
    serverInstructions: true
```

> Note: For now you can add this via the LibreChat UI directly without touching librechat.yaml.

-----

## 9. Phase Roadmap (High Level)

|Phase      |Scope                                                                  |Status       |
|-----------|-----------------------------------------------------------------------|-------------|
|**Phase 1**|Email read + search + summarise + calendar view                        |← We are here|
|Phase 2    |Draft email replies + send + create calendar events + meeting scheduler|Pending      |
|Phase 3    |Task extraction + MOM generation + follow-up tracking                  |Pending      |
|Phase 4    |Semantic search over email history + MongoDB persistence               |Pending      |
|Phase 5    |Teams integration + OneDrive document retrieval                        |Pending      |

-----

## 10. Risks & Mitigations

|Risk                                      |Impact                       |Mitigation                                                             |
|------------------------------------------|-----------------------------|-----------------------------------------------------------------------|
|VM IP changes on restart                  |LibreChat loses connection   |Use a static internal IP or hostname                                   |
|OAuth token expiry                        |Tool calls fail silently     |LibreChat handles refresh automatically; implement 401 retry in tools  |
|Graph API rate limits                     |Bulk email reads throttled   |Add retry with exponential backoff in graph clients                    |
|`Mail.ReadWrite` / `Mail.Send` not granted|Phase 2 draft/send blocked   |Raise with admin now — no code impact on Phase 1                       |
|No HTTPS yet                              |Fine for internal network dev|Switch to HTTPS when domain is assigned before any multi-user expansion|

-----

## 11. Definition of Done — Phase 1

Phase 1 is complete when all of the following are true:

- [ ] FastMCP server starts cleanly with `python server.py`
- [ ] All 6 tools appear in LibreChat MCP tool selector
- [ ] `get_my_profile` returns your name and email correctly
- [ ] `list_emails` returns your last N emails with subject, sender, date
- [ ] `read_email` returns full email body when given an ID
- [ ] `search_emails` returns relevant results for a keyword
- [ ] `summarise_email` produces a clean LLM-generated summary
- [ ] `list_calendar_events` returns your upcoming meetings
- [ ] OAuth authentication flow completes without errors
- [ ] All tools handle errors gracefully (no raw stack traces to UI)

-----

*Review this plan, modify if required, and confirm. Once confirmed I will ask: **Shall I proceed to implementation?***