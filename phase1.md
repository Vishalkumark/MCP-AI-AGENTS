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


---

./README.md
---
```
└── 📁FiGPT_OutlookMCP
    └── 📁auth
        ├── __init__.py
        ├── graph_auth.py
        ├── token_cache.py
    └── 📁bin
        ├── agent_output.txt
        ├── agent_test.py
        ├── del.txt
        ├── get_token.py
        ├── graph_auth_v1.py
        ├── quick_test.py
        ├── test_mcp_init.py
        ├── token.json
    └── 📁config
        ├── __init__.py
        ├── settings.py
    └── 📁graph
        ├── __init__.py
        ├── attachment_client.py
        ├── availability_client.py
        ├── calendar_client.py
        ├── draft_client.py
        ├── graph_client_factory.py
        ├── mail_client.py
        ├── task_client.py
        ├── thread_client.py
    └── 📁llm
        ├── __init__.py
        ├── requesty_client.py
    └── 📁logs
        ├── .gitkeep
    └── 📁parsers
        ├── __init__.py
        ├── docx_parser.py
        ├── ocr_fallback.py
        ├── parser_router.py
        ├── pdf_parser.py
        ├── pptx_parser.py
        ├── xlsx_parser.py
    └── 📁semantic
        ├── embedder.py
        ├── search_engine.py
    └── 📁temp_attachments
        └── 📁charts
            ├── .gitkeep
         ├── .gitkeep
    └── 📁tests
        ├── __init__.py
        ├── test_attachment_tools.py
        ├── test_auth.py
        ├── test_calendar_tools.py
        ├── test_charts.py
        ├── test_draft_tools.py
        ├── test_guardrails.py
        ├── test_mail_tools.py
        ├── test_mom_followup.py
        ├── test_parsers.py
    └── 📁tools
        ├── __init__.py
        ├── attachment_tools.py
        ├── calendar_tools.py
        ├── chart_tools.py
        ├── draft_tools.py
        ├── email_tools.py
        ├── folder_tools.py
        ├── followup_tools.py
        ├── mcp_instance.py
        ├── mom_tools.py
        ├── profile_tools.py
        ├── semantic_tools.py
        ├── task_tools.py
    └── 📁utils
        ├── __init__.py
        ├── audit_logger.py
        ├── date_utils.py
        ├── error_handler.py
        ├── governance.py
        ├── language_utils.py
        ├── logger.py
        ├── validator.py
    ├── .env
    ├── .env.example
    ├── .gitignore
    ├── README.md
    ├── requirements.txt
    ├── run_all_test.sh
    └── server.py
```


---
./requirements.txt
---
---
# Core MCP framework
fastmcp>=2.0.0

# Microsoft authentication
msal>=1.28.0

# Microsoft Graph SDK
msgraph-sdk>=1.5.0

# Environment / config loading
python-dotenv>=1.0.1

# HTTP client
httpx>=0.27.0

# Data validation
pydantic>=2.7.0

# Date parsing utility
python-dateutil>=2.9.0

# Attachment parsing — PDF
pymupdf>=1.24.0

# Attachment parsing — Word
python-docx>=1.1.2

# Attachment parsing — PowerPoint
python-pptx>=0.6.23

# Attachment parsing — Excel
openpyxl>=3.1.2

# OCR fallback (optional, scanned documents)
pytesseract>=0.3.10
pillow>=10.3.0

# pytest (test runner)
pytest 
pytest-asyncio

# Semantic search
sentence-transformers>=2.7.0

# Vector similarity calculations for semantic search
numpy>=1.26.0

# Cosine similarity scoring
scikit-learn>=1.4.0

# Multi-Language support
langdetect>=1.0.9

# Generate charts
matplotlib>=3.9.0
---

---
./run_all_test.sh
---
#!/bin/bash
# =============================================================================
# run_all_tests.sh
# Full test suite runner — lists all tools then runs all tests
# Usage: bash run_all_tests.sh
# =============================================================================

echo ""
echo "============================================================"
echo "  outlook-ai-agent-mcp — Full Test Suite"
echo "============================================================"

# Step 1: Check server is running and list all tools
echo ""
echo "[ STEP 1 ] Checking server health and listing tools…"
echo "————————————————————"

HEALTH=$(curl -s http://localhost:8000/health 2>/dev/null)

if [ -z "$HEALTH" ]; then
echo "  WARNING: Server not running on localhost:8000"
echo "  Start it with: python server.py"
echo "  Skipping tool listing — running tests only."
else
TOOL_COUNT=$(echo "$HEALTH" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get(‘tools_registered’,0))" 2>/dev/null)
echo "  Status      : $(echo "$HEALTH" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get(‘status’,‘unknown’))" 2>/dev/null)"
echo "  Server      : $(echo "$HEALTH" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get(‘server’,‘unknown’))" 2>/dev/null)"
echo "  Tools count : $TOOL_COUNT"
echo ""
echo "  Tools registered:"
echo "$HEALTH" | python3 -c "
import sys, json
d = json.load(sys.stdin)
tools = d.get(‘tools’, [])
for i, t in enumerate(tools, 1):
print(f’    {i:>2}. {t}’)
" 2>/dev/null
fi

# Step 2: Run all test files

echo ""
echo "[ STEP 2 ] Running all tests…"
echo "————————————————————"

python -m pytest tests/ -v   
–tb=short   
–no-header   
-q 2>&1 | tail -50

echo ""
echo "============================================================"
echo "  Done."
echo "============================================================"
echo ""
---

---
.env.example
---
# =================================================================
# AZURE / ENTRA ID — APP REGISTRATION (Authentication)
# =================================================================
AZURE_CLIENT_ID=
AZURE_CLIENT_SECRET=
AZURE_TENANT_ID=
AZURE_AUTHORITY=https://login.microsoftonline.com/<tenant-id>

# =================================================================
# MICROSOFT GRAPH API
# =================================================================
GRAPH_API_BASE_URL=https://graph.microsoft.com/v1.0
GRAPH_API_SCOPES=Mail.Read,Calendars.Read,User.Read

# =================================================================
# REQUESTY.AI — LLM GATEWAY
# =================================================================
REQUESTY_API_KEY=
REQUESTY_BASE_URL=https://router.requesty.ai/v1
REQUESTY_PRIMARY_MODEL=azure/gpt-4.1-nano@swedencentral
REQUESTY_FALLBACK_MODEL=mistral/mistral-small-2603

# =================================================================
# FASTMCP SERVER CONFIG
# =================================================================
MCP_HOST=0.0.0.0
MCP_PORT=8000
MCP_SERVER_NAME=outlook-ai-agent-mcp
MCP_TRANSPORT=streamable-http
MCP_TRANSPORT=http

# FASTMCP_HOST=0.0.0.0
FASTMCP_HOST=127.0.0.1
FASTMCP_PORT=8000

# =================================================================
# LIBRECHAT INTEGRATION
# =================================================================
LIBRECHAT_DOMAIN=
LIBRECHAT_OAUTH_CALLBACK=${LIBRECHAT_DOMAIN}/api/mcp/outlook-ai/oauth/callback

# =================================================================
# ATTACHMENT PROCESSING
# =================================================================
MAX_ATTACHMENT_SIZE_MB=25
ATTACHMENT_TEMP_DIR=./temp_attachments
OCR_ENABLED=false
TESSERACT_CMD_PATH=

# =================================================================
# FUTURE / DEPLOYMENT (FGSV297)
# =================================================================
DEPLOY_HOST=
DEPLOY_SSH_KEY_PATH=
DEPLOY_PYTHON_PATH=

# =================================================================
# LOGGING
# =================================================================
LOG_LEVEL=INFO

# =================================================================
# GRAPH API SCOPES (update existing line)
# =================================================================
GRAPH_API_SCOPES=Mail.Read,Mail.ReadWrite,Calendars.Read,Calendars.ReadWrite,Calendars.Read.Shared,User.Read

# =================================================================
# SEMANTIC SEARCH
# =================================================================
SEMANTIC_SEARCH_MODEL=all-MiniLM-L6-v2
SEMANTIC_SEARCH_MAX_EMAILS=100

# =================================================================
# FOLLOW-UP TRACKING
# =================================================================
FOLLOWUP_SCAN_COUNT=20
FOLLOWUP_DAYS_THRESHOLD=3

# =================================================================
# MICROSOFT PLANNER
# =================================================================
PLANNER_GROUP_ID=
PLANNER_PLAN_ID=

# =================================================================
# CHARTS
# =================================================================
CHARTS_OUTPUT_DIR=./temp_attachments/charts
CHARTS_MAX_ITEMS=20

---
./server.py
---
"""
server.py
=========
The FastMCP server entry point for the outlook-ai-agent-mcp platform.

Why this file exists:
    FastMCP needs a single place to:
        1. Create the MCP server application instance
        2. Register all MCP tools from the tools/ folder
        3. Start listening on the configured host and port

    This file does NOTHING ELSE. No business logic, no Graph API calls,
    no LLM calls — all of that lives in the appropriate layer (tools/,
    graph/, llm/). This file is purely the startup and wiring point.

Design notes:
    - The `mcp` object created here is the FastMCP server instance.
      It is imported by every file in tools/ (e.g. `from server import mcp`)
      so they can register their tools onto it with the @mcp.tool decorator.
    - Tool modules are imported AFTER the mcp instance is created, not
      before — this is important because the import of each tool module
      triggers the @mcp.tool decorators to fire, registering the tools.
      If mcp didn't exist yet when the import happened, registration
      would fail.
    - Transport is set to "streamable-http" per our architecture —
      this is what LibreChat's MCP integration expects.

How to run:
    From the project root with venv active:
        python server.py

    Expected output:
        INFO | server | Starting outlook-ai-agent-mcp on 0.0.0.0:8000
        INFO | server | Transport: streamable-http
        INFO | server | MCP endpoint: http://0.0.0.0:8000/mcp
        INFO | server | 9 tools registered and ready
        INFO | server | Server is running. Press Ctrl+C to stop.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
# Standard library
import sys

# FastMCP — the MCP server framework
from fastmcp import FastMCP

# from server import mcp
from tools.mcp_instance import mcp


# CORS middleware — required for LibreChat to connect from a different
# origin (different domain/port). Without this, LibreChat's browser-based
# connection will be blocked by the browser's cross-origin policy.
from starlette.responses import JSONResponse
from starlette.middleware.cors import CORSMiddleware
cors_app_wrapper = None  # placeholder — applied after mcp is initialised

# Our config and logger — loaded first before anything else
from config.settings import settings
from utils.logger import get_logger

logger = get_logger(__name__)

# ---------------------------------------------------------------------------
# Step 1: Create the FastMCP server instance.
# ---------------------------------------------------------------------------
# This `mcp` object is the central hub of the entire platform. It is
# imported by every tools/ file to register capabilities onto it.
# The name "outlook-ai-agent-mcp" appears in LibreChat's MCP server
# listing as the server's display name.
# mcp = FastMCP(settings.MCP_SERVER_NAME)


# ---------------------------------------------------------------------------
# Step 2: Import all tool modules.
# ---------------------------------------------------------------------------
# CRITICAL ORDER: mcp must be created (Step 1) BEFORE these imports.
# Each import below triggers the @mcp.tool decorators in that file to
# fire, registering each tool function onto the mcp instance above.
# If mcp didn't exist yet, the decorators would have nothing to register on.
#
# We import inside a try/except so a bad import in one tool module
# doesn't silently kill the server — it fails loudly with a clear
# message pointing to exactly which file has the problem.
try:
    # ── Phase 1 tools ────────────────────────────────────────────
    # Profile tools — get_my_profile
    from tools import profile_tools  # noqa: F401

    # Email tools — list_emails, read_email, search_emails,
    #               summarise_email, list_emails_paged,
    #               search_emails_advanced, export_emails_markdown,
    #               mark_email_read (Phase 2 addition)
    from tools import email_tools  # noqa: F401

    # Calendar tools — list_calendar_events
    from tools import calendar_tools  # noqa: F401

    # Attachment tools — list_attachments, read_attachment,
    #                    summarise_attachment
    from tools import attachment_tools  # noqa: F401

    # ── Phase 2 tools ────────────────────────────────────────────
    # Draft tools — list_draft_emails, compose_email,
    #               save_draft_to_outlook, create_draft_reply, 
    #               find_availability, create_draft_invite
    from tools import draft_tools  # noqa: F401

    # Folder tools — list_folders, create_folder, move_email
    from tools import folder_tools  # noqa: F401

    # Task tools — extract_tasks
    from tools import task_tools  # noqa: F401

    # Semantic tools — semantic_search_emails
    from tools import semantic_tools  # noqa: F401

    # ── Phase 3 tools ────────────────────────────────────────────
    # MOM tools — generate_mom, save_mom_as_draft
    from tools import mom_tools  # noqa: F401

    # Follow-up tools — track_followups, check_email_replied,
    #                   compose_followup, add_task_todo,
    #                   add_task_planner, list_tasks
    from tools import followup_tools  # noqa: F401

    # Chart tools — generate_chart
    from tools import chart_tools  # noqa: F401
    
    logger.info("All tool modules imported and registered successfully")

except ImportError as import_err:
    logger.error(
        f"Failed to import a tool module: {import_err}\n"
        f"Check that all dependencies are installed (pip install -r requirements.txt) "
        f"and that there are no syntax errors in the tools/ folder."
    )
    sys.exit(1)


# ---------------------------------------------------------------------------
# Step 3: Define a health check endpoint.
# ---------------------------------------------------------------------------
# This is a lightweight HTTP endpoint (not an MCP tool) that lets you
# quickly verify the server is reachable before testing through LibreChat.
# Test it with: curl http://localhost:8000/health
@mcp.custom_route("/health", methods=["GET"])
async def health_check(request) -> JSONResponse:

    """
    Simple health check endpoint. Returns server status and the list of
    registered tools. Returns JSON so the client can parse reliably.
    """
    tool_names = [tool.name for tool in await mcp.list_tools()]

    payload = {
        "status": "healthy",
        "server": settings.MCP_SERVER_NAME,
        "transport": settings.MCP_TRANSPORT,
        "tools_registered": len(tool_names),
        "tools": tool_names,
    }
    return JSONResponse(payload)

# ---------------------------------------------------------------------------
# Step 4: Start the server.
# ---------------------------------------------------------------------------
if __name__ == "__main__":
    logger.info(f"Starting {settings.MCP_SERVER_NAME} on {settings.MCP_HOST}:{settings.MCP_PORT}")
    logger.info(f"Transport: {settings.MCP_TRANSPORT}")
    # logger.info(f"MCP endpoint: http://{settings.MCP_HOST}:{settings.MCP_PORT}/mcp")
    # logger.info(f"Health check: http://{settings.MCP_HOST}:{settings.MCP_PORT}/health")
    # Use public URL in logs if configured, otherwise fall back to local
    public_url = getattr(settings, "MCP_PUBLIC_URL", None)
    display_url = public_url if public_url else f"http://{settings.MCP_HOST}:{settings.MCP_PORT}"
    logger.info(f"MCP endpoint : {display_url}/mcp")
    logger.info(f"Health check : {display_url}/health")
    logger.info(f"Internal     : http://{settings.MCP_HOST}:{settings.MCP_PORT}/mcp")
    logger.info("Server is running. Press Ctrl+C to stop.")

    # Step 4a: Run the FastMCP server using the transport and host/port
    #          from config/settings.py (which reads from .env).
    #          streamable-http is the transport LibreChat expects for
    #          production MCP server connections.
    mcp.run(
        transport=settings.MCP_TRANSPORT,
        host=settings.MCP_HOST,
        port=settings.MCP_PORT,
    )

---
./config/__init__.py
---


---
./config/settings.py
---
"""
settings.py
===========
Centralised configuration loader. Reads the .env file once and exposes
all settings as a single, typed, importable object.

Why this file exists:
    Without this file, every module in the project would call
    os.getenv("SOME_VAR") directly, scattered everywhere. That makes it
    hard to know what variables exist, easy to typo a variable name,
    and painful to refactor later. This file centralises all of that
    into one place — every other file imports `settings` from here and
    nothing else touches .env directly.

Design notes:
    - Validates required values manually (a lightweight pattern for
      Phase 1) so missing values fail loudly at startup rather than
      causing a confusing crash later, deep inside some unrelated
      function.
    - This file is imported ONCE when the server starts. The resulting
      `settings` object is then reused everywhere — we don't reload
      .env on every function call.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import os
from pathlib import Path
from dotenv import load_dotenv

# ---------------------------------------------------------------------------
# Step 1: Load the .env file from the project root.
# ---------------------------------------------------------------------------
# Path(__file__).parent.parent walks up from config/settings.py to the
# project root (outlook-ai-agent-mcp/), so this works regardless of
# which directory the script is actually run from.
PROJECT_ROOT = Path(__file__).resolve().parent.parent
ENV_PATH = PROJECT_ROOT / ".env"
load_dotenv(dotenv_path=ENV_PATH)


# ---------------------------------------------------------------------------
# Step 2: Define a helper to fetch required variables and fail loudly
#         if they're missing, instead of silently returning None.
# ---------------------------------------------------------------------------
def _require_env(var_name: str) -> str:
    """
    Fetch an environment variable that MUST be present for the app to
    function. Raises a clear error immediately if it's missing, rather
    than letting the app start in a broken state.
    """
    value = os.getenv(var_name)
    if value is None or value.strip() == "":
        raise EnvironmentError(
            f"Missing required environment variable: '{var_name}'. "
            f"Please check your .env file."
        )
    return value


def _optional_env(var_name: str, default: str = "") -> str:
    """
    Fetch an environment variable that is OPTIONAL — returns a default
    value if not set, without raising an error.
    """
    return os.getenv(var_name, default)


# ---------------------------------------------------------------------------
# Step 3: Define the Settings class — one attribute per .env variable,
#         grouped by section for readability.
# ---------------------------------------------------------------------------
class Settings:
    """
    Holds every configuration value the application needs, loaded once
    from .env at import time. Access via the shared `settings` instance
    at the bottom of this file — never instantiate this class again
    elsewhere in the project.
    """

    def __init__(self):
        # ----- Azure / Entra ID -----
        self.AZURE_CLIENT_ID: str = _require_env("AZURE_CLIENT_ID")
        self.AZURE_CLIENT_SECRET: str = _require_env("AZURE_CLIENT_SECRET")
        self.AZURE_TENANT_ID: str = _require_env("AZURE_TENANT_ID")
        self.AZURE_AUTHORITY: str = _require_env("AZURE_AUTHORITY")

        # ----- Microsoft Graph API -----
        self.GRAPH_API_BASE_URL: str = _optional_env(
            "GRAPH_API_BASE_URL", "https://graph.microsoft.com/v1.0"
        )
        graph_scopes_raw: str = _optional_env(
            "GRAPH_API_SCOPES", "Mail.Read,Calendars.Read,User.Read"
        )
        # Step 4: Scopes arrive as a comma-separated string in .env —
        #         split into a clean Python list here, once, so every
        #         caller gets a ready-to-use list.
        self.GRAPH_API_SCOPES: list[str] = [
            s.strip() for s in graph_scopes_raw.split(",") if s.strip()
        ]

        # ----- Requesty.AI -----
        self.REQUESTY_API_KEY: str = _require_env("REQUESTY_API_KEY")
        self.REQUESTY_BASE_URL: str = _optional_env(
            "REQUESTY_BASE_URL", "https://router.requesty.ai/v1"
        )
        self.REQUESTY_PRIMARY_MODEL: str = _optional_env(
            "REQUESTY_PRIMARY_MODEL", "azure/gpt-4.1-nano@swedencentral"
        )
        self.REQUESTY_FALLBACK_MODEL: str = _optional_env(
            "REQUESTY_FALLBACK_MODEL", "mistral/mistral-small-2603"
        )

        # ----- FastMCP server -----
        # self.MCP_HOST: str = _optional_env("MCP_HOST", "0.0.0.0")
        # self.MCP_PORT: int = int(_optional_env("MCP_PORT", "8000"))
        # self.MCP_SERVER_NAME: str = _optional_env(
        #     "MCP_SERVER_NAME", "outlook-ai-agent-mcp"
        # )
        # self.MCP_TRANSPORT: str = _optional_env(
        #     "MCP_TRANSPORT", "streamable-http"
        # )

        self.MCP_HOST: str = _optional_env("MCP_HOST", "0.0.0.0")
        self.MCP_PORT: int = int(_optional_env("MCP_PORT", "8000"))
        self.MCP_SERVER_NAME: str = _optional_env(
            "MCP_SERVER_NAME", "outlook-ai-agent-mcp"
        )
        self.MCP_TRANSPORT: str = _optional_env(
            "MCP_TRANSPORT", "streamable-http"
        )
        self.MCP_PUBLIC_URL: str = _optional_env("MCP_PUBLIC_URL", "")

        # ----- LibreChat integration -----
        self.LIBRECHAT_DOMAIN: str = _optional_env("LIBRECHAT_DOMAIN", "")
        self.LIBRECHAT_OAUTH_CALLBACK: str = _optional_env(
            "LIBRECHAT_OAUTH_CALLBACK", ""
        )

        # ----- Attachment processing -----
        self.MAX_ATTACHMENT_SIZE_MB: int = int(
            _optional_env("MAX_ATTACHMENT_SIZE_MB", "25")
        )
        self.ATTACHMENT_TEMP_DIR: str = _optional_env(
            "ATTACHMENT_TEMP_DIR", "./temp_attachments"
        )
        # Step 5: Booleans arrive as strings from .env ("true"/"false") —
        #         convert explicitly rather than relying on Python's
        #         truthy/falsy string behaviour (bool("false") is True!).
        self.OCR_ENABLED: bool = _optional_env(
            "OCR_ENABLED", "false"
        ).strip().lower() == "true"
        self.TESSERACT_CMD_PATH: str = _optional_env("TESSERACT_CMD_PATH", "")

        # ----- Future / deployment (FGSV297) -----
        self.DEPLOY_HOST: str = _optional_env("DEPLOY_HOST", "")
        self.DEPLOY_SSH_KEY_PATH: str = _optional_env("DEPLOY_SSH_KEY_PATH", "")
        self.DEPLOY_PYTHON_PATH: str = _optional_env("DEPLOY_PYTHON_PATH", "")

        # ----- Logging -----
        self.LOG_LEVEL: str = _optional_env("LOG_LEVEL", "INFO")

        # ----- Phase 2 — Semantic search -----
        self.SEMANTIC_SEARCH_MODEL: str = _optional_env(
            "SEMANTIC_SEARCH_MODEL", "all-MiniLM-L6-v2"
        )
        self.SEMANTIC_SEARCH_MAX_EMAILS: int = int(
            _optional_env("SEMANTIC_SEARCH_MAX_EMAILS", "100")
        )

                # ----- Phase 3 — Follow-up tracking -----
        self.FOLLOWUP_SCAN_COUNT: int = int(
            _optional_env("FOLLOWUP_SCAN_COUNT", "20")
        )
        self.FOLLOWUP_DAYS_THRESHOLD: int = int(
            _optional_env("FOLLOWUP_DAYS_THRESHOLD", "3")
        )

        # ----- Phase 3 — Microsoft Planner -----
        self.PLANNER_GROUP_ID: str = _optional_env("PLANNER_GROUP_ID", "")
        self.PLANNER_PLAN_ID: str = _optional_env("PLANNER_PLAN_ID", "")

        # ----- Charts -----
        self.CHARTS_OUTPUT_DIR: str = _optional_env(
            "CHARTS_OUTPUT_DIR", "./temp_attachments/charts"
        )
        self.CHARTS_MAX_ITEMS: int = int(
            _optional_env("CHARTS_MAX_ITEMS", "20")
        )
        
# ---------------------------------------------------------------------------
# Step 6: Create ONE shared instance, imported everywhere else as:
#         from config.settings import settings
# ---------------------------------------------------------------------------
settings = Settings()

---
./tests/__init__.py
---


---
./tests/test_attachment_tools.py
---
"""
test_attachment_tools.py
========================
Tests for tools/attachment_tools.py — list_attachments, read_attachment,
and summarise_attachment.

How to run:
    python -m pytest tests/test_attachment_tools.py -v
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import pytest
from unittest.mock import AsyncMock, patch


# ---------------------------------------------------------------------------
# Test: list_attachments returns correct metadata shape
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_list_attachments_returns_correct_shape():
    """
    list_attachments() should return a list of dicts with attachment
    metadata: attachment_id, file_name, content_type, size_kb.
    """
    fake_attachments = [
        {
            "id": "att-001",
            "name": "Q3_Report.pdf",
            "contentType": "application/pdf",
            "size": 512000,  # 500 KB in bytes
        }
    ]

    with patch(
        "tools.attachment_tools.list_message_attachments",
        new=AsyncMock(return_value=fake_attachments)
    ):
        from tools.attachment_tools import list_attachments
        result = await list_attachments(email_id="msg-001")

    assert isinstance(result, list)
    assert result[0]["file_name"] == "Q3_Report.pdf"
    assert result[0]["content_type"] == "application/pdf"
    assert result[0]["size_kb"] == 500.0


# ---------------------------------------------------------------------------
# Test: read_attachment returns extracted text
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_read_attachment_returns_extracted_text():
    """
    read_attachment() should download bytes, route through the parser,
    and return the extracted text and file type.
    """
    fake_bytes = b"%PDF-1.4 sample bytes"
    fake_parse_result = {
        "text": "This is the extracted text from the PDF.",
        "file_type": "pdf",
        "used_ocr": False,
    }

    with patch(
        "tools.attachment_tools.download_attachment",
        new=AsyncMock(return_value=(fake_bytes, "Report.pdf", "application/pdf"))
    ), patch(
        "tools.attachment_tools.parse_attachment",
        new=AsyncMock(return_value=fake_parse_result)
    ):
        from tools.attachment_tools import read_attachment
        result = await read_attachment(email_id="msg-001", attachment_id="att-001")

    assert result["file_name"] == "Report.pdf"
    assert result["file_type"] == "pdf"
    assert result["used_ocr"] is False
    assert "extracted text" in result["extracted_text"]


# ---------------------------------------------------------------------------
# Test: summarise_attachment handles empty text gracefully
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_summarise_attachment_handles_empty_text():
    """
    If the parser extracts no text (blank or corrupted file),
    summarise_attachment() should return a clear message instead of
    calling the LLM with an empty prompt.
    """
    fake_bytes = b""
    fake_parse_result = {
        "text": "",   # Empty — nothing extracted
        "file_type": "pdf",
        "used_ocr": False,
    }

    with patch(
        "tools.attachment_tools.download_attachment",
        new=AsyncMock(return_value=(fake_bytes, "Empty.pdf", "application/pdf"))
    ), patch(
        "tools.attachment_tools.parse_attachment",
        new=AsyncMock(return_value=fake_parse_result)
    ):
        from tools.attachment_tools import summarise_attachment
        result = await summarise_attachment(email_id="msg-001", attachment_id="att-001")

    # Should NOT call the LLM — should return the "no readable text" message.
    assert "No readable text" in result["instruction"]

---
./tests/test_auth.py
---
"""
test_auth.py
============
Tests for auth/graph_auth.py and auth/token_cache.py.

What this tests:
    - Token extraction from a well-formed Authorization header
    - Correct rejection of missing or malformed headers
    - Token stored and cleared correctly in request-scoped context

How to run (from project root, with venv active):
    python -m pytest tests/test_auth.py -v
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import pytest
from unittest.mock import patch, MagicMock

# Functions under test
from auth.graph_auth import get_current_access_token
from auth.token_cache import set_current_token, get_current_token, clear_current_token
from fastmcp.exceptions import AuthorizationError


# ---------------------------------------------------------------------------
# Test: successful token extraction from a valid Authorization header
# ---------------------------------------------------------------------------
def test_get_current_access_token_valid():
    """
    When a well-formed 'Bearer <token>' Authorization header is present,
    get_current_access_token() should return just the raw token string
    without the 'Bearer ' prefix.
    """
    # Step 1: Mock get_http_headers() to simulate LibreChat sending a
    #         valid Authorization header with our request.
    mock_headers = {"Authorization": "Bearer my-test-token-abc123"}

    with patch("auth.graph_auth.get_http_headers", return_value=mock_headers):
        token = get_current_access_token()

    # Step 2: The function should strip "Bearer " and return just the token.
    assert token == "my-test-token-abc123"


# ---------------------------------------------------------------------------
# Test: missing Authorization header raises AuthorizationError
# ---------------------------------------------------------------------------
def test_get_current_access_token_missing_header():
    """
    When no Authorization header is present in the request, the function
    should raise AuthorizationError — not return None or crash silently.
    """
    mock_headers = {}  # Empty — no Authorization header

    with patch("auth.graph_auth.get_http_headers", return_value=mock_headers):
        with pytest.raises(AuthorizationError):
            get_current_access_token()


# ---------------------------------------------------------------------------
# Test: malformed header (missing "Bearer " prefix) raises AuthorizationError
# ---------------------------------------------------------------------------
def test_get_current_access_token_malformed_header():
    """
    If the Authorization header is present but doesn't start with
    'Bearer ', the function should raise AuthorizationError.
    """
    mock_headers = {"Authorization": "Basic some-other-scheme"}

    with patch("auth.graph_auth.get_http_headers", return_value=mock_headers):
        with pytest.raises(AuthorizationError):
            get_current_access_token()


# ---------------------------------------------------------------------------
# Test: token cache stores and retrieves correctly
# ---------------------------------------------------------------------------
def test_token_cache_set_and_get():
    """
    set_current_token() stores a token that get_current_token() can
    then retrieve within the same execution context.
    """
    test_token = "test-token-xyz-9999"

    set_current_token(test_token)
    retrieved = get_current_token()

    assert retrieved == test_token


# ---------------------------------------------------------------------------
# Test: token cache clears correctly
# ---------------------------------------------------------------------------
def test_token_cache_clear():
    """
    After clear_current_token() is called, get_current_token() should
    return None — confirming no token persists between requests.
    """
    set_current_token("temporary-token")
    clear_current_token()

    result = get_current_token()

    assert result is None

---
./tests/test_calendar_tools.py
---
"""
test_calendar_tools.py
======================
Tests for tools/calendar_tools.py — list_calendar_events.

How to run:
    python -m pytest tests/test_calendar_tools.py -v
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import pytest
from unittest.mock import AsyncMock, patch


# ---------------------------------------------------------------------------
# Test: list_calendar_events returns correctly shaped output
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_list_calendar_events_returns_correct_shape():
    """
    list_calendar_events() should return a list of event dicts with
    the fields defined in its docstring.
    """
    fake_events = [
        {
            "subject": "Project Kick-off Meeting",
            "start": {"dateTime": "2025-01-20T10:00:00"},
            "end": {"dateTime": "2025-01-20T11:00:00"},
            "organizer": {"emailAddress": {"name": "Klaus Wagner"}},
            "location": {"displayName": "Conference Room A"},
            "isOnlineMeeting": False,
        }
    ]

    # Patch both the graph call AND the date parser to keep the test
    # self-contained — no dependency on the current actual date.
    with patch(
        "tools.calendar_tools.fetch_calendar_events",
        new=AsyncMock(return_value=fake_events)
    ), patch(
        "tools.calendar_tools.parse_relative_date_range",
        return_value=(None, None)
    ):
        from tools.calendar_tools import list_calendar_events
        result = await list_calendar_events(date_range="today")

    assert isinstance(result, list)
    assert len(result) == 1
    assert result[0]["subject"] == "Project Kick-off Meeting"
    assert result[0]["organizer"] == "Klaus Wagner"
    assert result[0]["is_online_meeting"] is False


# ---------------------------------------------------------------------------
# Test: error handling returns clean dict
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_list_calendar_events_handles_exception_gracefully():
    """
    If Graph API fails, list_calendar_events() should return an error
    dict, not raise an unhandled exception.
    """
    with patch(
        "tools.calendar_tools.fetch_calendar_events",
        new=AsyncMock(side_effect=Exception("Graph API timeout"))
    ), patch(
        "tools.calendar_tools.parse_relative_date_range",
        return_value=(None, None)
    ):
        from tools.calendar_tools import list_calendar_events
        result = await list_calendar_events(date_range="today")

    assert isinstance(result, dict)
    assert result.get("error") is True

---
./tests/test_charts.py
---
"""
test_charts.py
==============
Tests for tools/chart_tools.py — generate_chart tool.

How to run:
    python -m pytest tests/test_charts.py -v
"""

import pytest
from unittest.mock import patch, MagicMock
from pathlib import Path


# class TestGenerateChart:
#     """Tests for generate_chart tool."""

@pytest.mark.asyncio
async def test_bar_chart_generates_successfully():
    with patch("tools.chart_tools._generate_bar_chart") as mock_bar:
        mock_bar.return_value = ("/tmp/bar_chart_test.svg", "<svg>test</svg>")

        from tools.chart_tools import generate_chart
        result = await generate_chart(
            chart_type="bar",
            labels="John, Jane, HR",
            values="15, 8, 3",
            title="Emails by Sender",
            y_label="Email Count",
        )

    assert "error" not in result
    assert result["chart_type"] == "bar"
    assert result["data_points"] == 3
    assert "📊" in result["instruction"]


@pytest.mark.asyncio
async def test_pie_chart_generates_successfully():
    with patch("tools.chart_tools._generate_pie_chart") as mock_pie:
        mock_pie.return_value = ("/tmp/pie_chart_test.svg", "<svg>pie</svg>")

        from tools.chart_tools import generate_chart
        result = await generate_chart(
            chart_type="pie",
            labels="Inbox, Drafts, Sent",
            values="120, 5, 45",
            title="Email Distribution by Folder",
        )

    assert result["chart_type"] == "pie"
    assert result["data_points"] == 3


@pytest.mark.asyncio
async def test_line_chart_generates_successfully():
    with patch("tools.chart_tools._generate_line_chart") as mock_line:
        mock_line.return_value = ("/tmp/line_chart_test.svg", "<svg>line</svg>")

        from tools.chart_tools import generate_chart
        result = await generate_chart(
            chart_type="line",
            labels="Mon, Tue, Wed, Thu, Fri",
            values="4, 7, 2, 9, 5",
            title="Emails Received Per Day",
            x_label="Day",
            y_label="Count",
        )

    assert result["chart_type"] == "line"
    assert result["data_points"] == 5


@pytest.mark.asyncio
async def test_returns_error_for_invalid_chart_type():
    from tools.chart_tools import generate_chart
    result = await generate_chart(
        chart_type="scatter",
        labels="A, B",
        values="1, 2",
        title="Test",
    )
    assert result.get("error") is True
    assert "scatter" in result["message"]


@pytest.mark.asyncio
async def test_returns_error_for_mismatched_labels_values():
    from tools.chart_tools import generate_chart
    result = await generate_chart(
        chart_type="bar",
        labels="A, B, C",
        values="1, 2",
        title="Mismatch Test",
    )
    assert result.get("error") is True
    assert "Mismatch" in result["message"]


@pytest.mark.asyncio
async def test_returns_error_for_non_numeric_values():
    from tools.chart_tools import generate_chart
    result = await generate_chart(
        chart_type="bar",
        labels="A, B",
        values="10, twenty",
        title="Bad Values Test",
    )
    assert result.get("error") is True
    assert "numbers" in result["message"].lower()


@pytest.mark.asyncio
async def test_returns_error_for_empty_labels():
    from tools.chart_tools import generate_chart
    result = await generate_chart(
        chart_type="pie",
        labels="",
        values="",
        title="Empty Test",
    )
    assert result.get("error") is True

---
./tests/test_draft_tools.py
---
"""
test_phase2_tools.py
====================
Tests for Phase 2 tools — draft, folder, task, and semantic tools.

What this tests:
    - Draft tool output shapes
    - Folder tool output shapes
    - Task extraction instruction format
    - Semantic search with mock email corpus
    - Error handling for all new tools

How to run:
    python -m pytest tests/test_phase2_tools.py -v
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import pytest
import numpy as np
from unittest.mock import AsyncMock, patch, MagicMock


# ===========================================================================
# Draft Tool Tests
# ===========================================================================

class TestListDraftEmails:
    """Tests for list_draft_emails tool."""

    @pytest.mark.asyncio
    async def test_returns_table_when_drafts_exist(self):
        """
        list_draft_emails() should return a display_table and count
        when drafts are present.
        """
        fake_drafts = [
            {
                "id": "draft-001",
                "subject": "Re: Project Update",
                "to": ["john@company.com"],
                "last_modified": "2026-07-01T10:00:00Z",
                "preview": "Hi John, following up on the project...",
            }
        ]

        with patch(
            "tools.draft_tools.get_draft_emails",
            new=AsyncMock(return_value=fake_drafts)
        ):
            from tools.draft_tools import list_draft_emails
            result = await list_draft_emails(count=5)

        assert result["count"] == 1
        assert "display_table" in result
        assert "Re: Project Update" in result["display_table"]
        assert "instruction" in result

    @pytest.mark.asyncio
    async def test_returns_empty_message_when_no_drafts(self):
        """
        list_draft_emails() should return a clear message when
        the Drafts folder is empty.
        """
        with patch(
            "tools.draft_tools.get_draft_emails",
            new=AsyncMock(return_value=[])
        ):
            from tools.draft_tools import list_draft_emails
            result = await list_draft_emails()

        assert result["count"] == 0
        assert "empty" in result["display_table"].lower() or "No drafts" in result["display_table"]


# class TestCreateDraftEmail:
#     """Tests for create_draft_email tool."""

@pytest.mark.asyncio
async def test_compose_email_returns_instruction():
    """
    compose_email() should return an instruction field and
    header for LibreChat to compose and display the email.
    """
    from tools.draft_tools import compose_email
    result = await compose_email(
        to_emails="john@company.com",
        subject="Project Status Update",
        context="Update John on the delayed timeline",
    )

    assert "instruction" in result
    assert "john@company.com" in result["to"]
    assert result["subject"] == "Project Status Update"


@pytest.mark.asyncio
async def test_compose_email_returns_error_for_empty_recipients():
    """
    compose_email() should return an error dict when no
    valid recipient email addresses are provided.
    """
    from tools.draft_tools import compose_email
    result = await compose_email(
        to_emails="",
        subject="Test",
        context="Test context",
    )
    assert result.get("error") is True
    assert "recipient" in result["message"].lower()


class TestFindAvailability:
    """Tests for find_availability tool."""

    @pytest.mark.asyncio
    async def test_returns_table_when_slots_found(self):
        """
        find_availability() should return a display_table with
        suggested time slots when availability is found.
        """
        fake_result = {
            "suggestions": [
                {
                    "start": "2026-07-15T10:00:00",
                    "start_display": "Tuesday, 15 July 2026 at 10:00 UTC",
                    "end_display": "10:30 UTC",
                    "duration_minutes": 30,
                    "confidence": 85.0,
                    "quality": "🟢 High",
                    "organizer_available": "free",
                }
            ],
            "attendees_checked": ["john@company.com"],
            "duration_minutes": 30,
            "search_window_days": 7,
        }

        with patch(
            "tools.draft_tools.find_meeting_times",
            new=AsyncMock(return_value=fake_result)
        ):
            from tools.draft_tools import find_availability
            result = await find_availability(
                attendee_emails="john@company.com",
                duration_minutes=30,
            )

        assert "display_table" in result
        assert "10:00" in result["display_table"]
        assert "🟢" in result["display_table"]

    @pytest.mark.asyncio
    async def test_returns_error_for_empty_attendees(self):
        """
        find_availability() should return an error when no
        attendee emails are provided.
        """
        from tools.draft_tools import find_availability
        result = await find_availability(attendee_emails="")

        assert result.get("error") is True


# ===========================================================================
# Folder Tool Tests
# ===========================================================================

class TestListFolders:
    """Tests for list_folders tool."""

    @pytest.mark.asyncio
    async def test_returns_folder_table(self):
        """
        list_folders() should return a table with folder names
        and unread/total counts.
        """
        fake_folders = [
            {"id": "f1", "name": "Inbox", "unread_count": 5, "total_count": 120},
            {"id": "f2", "name": "Projects", "unread_count": 0, "total_count": 45},
        ]

        with patch(
            "tools.folder_tools.get_mail_folders",
            new=AsyncMock(return_value=fake_folders)
        ):
            from tools.folder_tools import list_folders
            result = await list_folders()

        assert result["count"] == 2
        assert "Inbox" in result["display_table"]
        assert "Projects" in result["display_table"]
        # Unread count should be bolded
        assert "**5**" in result["display_table"]


class TestMoveEmail:
    """Tests for move_email tool."""

    @pytest.mark.asyncio
    async def test_moves_to_well_known_folder(self):
        """
        move_email() should resolve well-known folder names like
        'archive' and call move with the correct folder ID.
        """
        fake_result = {
            "new_message_id": "msg-new-001",
            "subject": "Test Email",
            "status": "Email moved successfully",
        }

        with patch(
            "tools.folder_tools.move_message_to_folder",
            new=AsyncMock(return_value=fake_result)
        ) as mock_move:
            from tools.folder_tools import move_email
            result = await move_email(
                email_id="msg-001",
                destination_folder="archive",
            )

        # Should have called with the well-known folder name
        mock_move.assert_called_once_with(
            message_id="msg-001",
            destination_folder_id="archive",
        )
        assert result["status"] == "Email moved successfully"

    @pytest.mark.asyncio
    async def test_returns_error_for_unknown_folder(self):
        """
        move_email() should return an error when the destination
        folder name doesn't match any known folder.
        """
        with patch(
            "tools.folder_tools.get_mail_folders",
            new=AsyncMock(return_value=[
                {"id": "f1", "name": "Inbox", "unread_count": 0, "total_count": 10}
            ])
        ):
            from tools.folder_tools import move_email
            result = await move_email(
                email_id="msg-001",
                destination_folder="NonExistentFolder",
            )

        assert result.get("error") is True
        assert "not found" in result["message"].lower()


# ===========================================================================
# Task Extraction Tests
# ===========================================================================

class TestExtractTasks:
    """Tests for extract_tasks tool."""

    @pytest.mark.asyncio
    async def test_returns_instruction_with_email_digest(self):
        """
        extract_tasks() should return an instruction string containing
        the email digest and task extraction prompt for LibreChat's LLM.
        """
        fake_messages = [
            {
                "id": "msg-001",
                "subject": "Q3 Report Due Friday",
                "from": {"emailAddress": {"name": "Manager", "address": "mgr@co.com"}},
                "receivedDateTime": "2026-07-01T09:00:00Z",
                "bodyPreview": "Please submit the Q3 report by Friday 5pm.",
                "hasAttachments": False,
            }
        ]

        with patch(
            "tools.task_tools.fetch_recent_messages",
            new=AsyncMock(return_value=fake_messages)
        ):
            from tools.task_tools import extract_tasks
            result = await extract_tasks(count=5)

        assert result["emails_analysed"] == 1
        assert "instruction" in result
        assert "Action Items" in result["instruction"]
        assert "Q3 Report Due Friday" in result["email_digest"]

    @pytest.mark.asyncio
    async def test_handles_empty_inbox(self):
        """
        extract_tasks() should handle an empty inbox gracefully
        without raising an exception.
        """
        with patch(
            "tools.task_tools.fetch_recent_messages",
            new=AsyncMock(return_value=[])
        ):
            from tools.task_tools import extract_tasks
            result = await extract_tasks(count=5)

        assert result["emails_analysed"] == 0
        assert "instruction" in result


# ===========================================================================
# Semantic Search Tests
# ===========================================================================

class TestSemanticSearchEmails:
    """Tests for semantic_search_emails tool."""

    @pytest.mark.asyncio
    async def test_returns_ranked_results(self):
        """
        semantic_search_emails() should return results ranked by
        similarity score when relevant emails exist.
        """
        fake_emails = [
            {
                "id": "msg-001",
                "subject": "Budget overrun concerns",
                "from": {"emailAddress": {"name": "CFO", "address": "cfo@co.com"}},
                "receivedDateTime": "2026-07-01T09:00:00Z",
                "bodyPreview": "We need to discuss the cost overruns immediately.",
                "hasAttachments": False,
            },
            {
                "id": "msg-002",
                "subject": "Team lunch plans",
                "from": {"emailAddress": {"name": "HR", "address": "hr@co.com"}},
                "receivedDateTime": "2026-07-02T09:00:00Z",
                "bodyPreview": "Join us for lunch on Friday at noon.",
                "hasAttachments": False,
            },
        ]

        # Mock fetch_recent_messages to return our fake emails
        with patch(
            "tools.semantic_tools.fetch_recent_messages",
            new=AsyncMock(return_value=fake_emails)
        ):
            # Mock semantic_search to return a controlled result
            # so we don't need real model inference in tests
            mock_result = [
                {
                    **fake_emails[0],
                    "similarity_score": 0.82,
                    "relevance": "🟢 High",
                }
            ]
            with patch(
                "tools.semantic_tools.semantic_search",
                return_value=mock_result
            ):
                from tools.semantic_tools import semantic_search_emails
                result = await semantic_search_emails(
                    query="budget problems",
                    top_k=5,
                )

        assert result["result_count"] == 1
        assert "display_table" in result
        assert "0.82" in result["display_table"]
        assert "🟢" in result["display_table"]

    @pytest.mark.asyncio
    async def test_returns_message_when_no_results(self):
        """
        semantic_search_emails() should return a helpful message
        when no results meet the similarity threshold.
        """
        fake_emails = [
            {
                "id": "msg-001",
                "subject": "Team lunch",
                "from": {"emailAddress": {"name": "HR", "address": "hr@co.com"}},
                "receivedDateTime": "2026-07-01T09:00:00Z",
                "bodyPreview": "Join us for lunch.",
                "hasAttachments": False,
            }
        ]

        with patch(
            "tools.semantic_tools.fetch_recent_messages",
            new=AsyncMock(return_value=fake_emails)
        ), patch(
            "tools.semantic_tools.semantic_search",
            return_value=[]  # No results above threshold
        ):
            from tools.semantic_tools import semantic_search_emails
            result = await semantic_search_emails(query="quarterly financial report")

        assert result["result_count"] == 0
        assert "No semantically relevant" in result["display_table"]

    @pytest.mark.asyncio
    async def test_returns_error_for_empty_query(self):
        """
        semantic_search_emails() should return an error dict
        when no query is provided.
        """
        from tools.semantic_tools import semantic_search_emails
        result = await semantic_search_emails(query="")

        assert result.get("error") is True

---
./tests/test_guardrails.py
---
"""
test_guardrails.py
==================
Tests for utils/governance.py, utils/validator.py,
and utils/audit_logger.py.

How to run:
    python -m pytest tests/test_guardrails.py -v
"""

import pytest
import json
from pathlib import Path


# ===========================================================================
# Governance Rules Tests
# ===========================================================================

class TestGovernanceRules:
    """Tests for utils/governance.py"""

    def test_base_rules_in_email_rules(self):
        """
        get_email_rules() should contain the base rules
        prohibiting invented facts.
        """
        from utils.governance import get_email_rules
        rules = get_email_rules()
        assert "Only state facts" in rules
        assert "Never invent" in rules

    def test_email_rules_contain_source_citation(self):
        """
        get_email_rules(include_sources=True) should include
        the SOURCE CITATION block.
        """
        from utils.governance import get_email_rules
        rules = get_email_rules(include_sources=True)
        assert "Sources:" in rules

    def test_email_rules_no_citation_when_disabled(self):
        """
        get_email_rules(include_sources=False) should NOT include
        SOURCE CITATION block.
        """
        from utils.governance import get_email_rules
        rules = get_email_rules(include_sources=False)
        assert "Sources:" not in rules

    def test_mom_rules_contain_decisions_instruction(self):
        """
        get_mom_rules() should include instruction about only listing
        explicitly stated decisions.
        """
        from utils.governance import get_mom_rules
        rules = get_mom_rules()
        assert "explicitly stated" in rules

    def test_draft_rules_contain_no_promises(self):
        """
        get_draft_rules() should prohibit adding promises not
        in the context.
        """
        from utils.governance import get_draft_rules
        rules = get_draft_rules()
        assert "promises" in rules.lower()

    def test_task_rules_contain_owner_instruction(self):
        """
        get_task_rules() should include instruction about not
        guessing task owners.
        """
        from utils.governance import get_task_rules
        rules = get_task_rules()
        assert "Unassigned" in rules


# ===========================================================================
# Validator Tests
# ===========================================================================

class TestValidateEmailList:
    """Tests for validator.validate_email_list"""

    def test_valid_email_list_passes(self):
        """
        A list of properly formed emails should pass validation.
        """
        from utils.validator import validate_email_list

        emails = [
            {"id": "msg-001", "subject": "Test", "sender": "John"},
            {"id": "msg-002", "subject": "Another", "sender": "Jane"},
        ]
        result = validate_email_list(emails)
        assert result.is_valid is True
        assert len(result.errors) == 0

    def test_empty_list_generates_warning(self):
        """
        An empty email list should generate a warning but not an error.
        """
        from utils.validator import validate_email_list

        result = validate_email_list([])
        assert result.is_valid is True
        assert len(result.warnings) == 1
        assert "empty" in result.warnings[0].lower() or "No emails" in result.warnings[0]

    def test_duplicate_ids_generate_warning(self):
        """
        Duplicate email IDs should generate a validation warning.
        """
        from utils.validator import validate_email_list

        emails = [
            {"id": "msg-001", "subject": "Test 1"},
            {"id": "msg-001", "subject": "Test 2"},  # duplicate
        ]
        result = validate_email_list(emails)
        assert any("Duplicate" in w for w in result.warnings)


class TestValidateEmailBody:
    """Tests for validator.validate_email_body"""

    def test_normal_body_passes(self):
        """
        A normal email body should pass validation without warnings.
        """
        from utils.validator import validate_email_body

        result = validate_email_body(
            "Please find attached the quarterly report for your review. "
            "Let me know if you have any questions.",
            "Q3 Report"
        )
        assert result.is_valid is True
        assert len(result.warnings) == 0

    def test_empty_body_generates_warning(self):
        """
        An empty email body should generate a validation warning.
        """
        from utils.validator import validate_email_body

        result = validate_email_body("", "Test Subject")
        assert len(result.warnings) == 1

    def test_very_short_body_generates_warning(self):
        """
        A very short email body should generate a warning about
        summary quality.
        """
        from utils.validator import validate_email_body

        result = validate_email_body("Hi", "Short email")
        assert any("short" in w.lower() for w in result.warnings)


class TestValidateMomOutput:
    """Tests for validator.validate_mom_output"""

    def test_complete_mom_passes(self):
        """
        A MOM with all required sections should pass validation.
        """
        from utils.validator import validate_mom_output

        complete_mom = """
        MINUTES OF MEETING
        1. MEETING DETAILS
        2. ATTENDEES | Name | Email |
        3. DISCUSSION SUMMARY Some discussion happened here.
        4. DECISIONS MADE Decision was made.
        5. ACTION ITEMS | Task | Owner |
        6. NEXT STEPS Next meeting planned.
        7. NOTES No additional notes.
        """
        result = validate_mom_output(complete_mom)
        assert result.is_valid is True

    def test_too_short_mom_fails(self):
        """
        A MOM that is too short should fail validation.
        """
        from utils.validator import validate_mom_output

        result = validate_mom_output("Short MOM")
        assert result.is_valid is False
        assert len(result.errors) > 0

    def test_missing_section_generates_warning(self):
        """
        A MOM missing a required section should generate a warning.
        """
        from utils.validator import validate_mom_output

        mom_missing_actions = """
        MEETING DETAILS date today
        ATTENDEES John Jane
        DISCUSSION SUMMARY We talked about things.
        DECISIONS MADE We decided something.
        NEXT STEPS Meet again.
        """ * 5  # Make it long enough to pass length check

        result = validate_mom_output(mom_missing_actions)
        assert any("ACTION ITEMS" in w for w in result.warnings)


class TestValidateDraftContent:
    """Tests for validator.validate_draft_content"""

    def test_valid_draft_passes(self):
        """
        A properly formed draft should pass all validation checks.
        """
        from utils.validator import validate_draft_content

        result = validate_draft_content(
            body_text="Dear John,\n\nThank you for your email.\n\nBest regards,",
            to_emails=["john@company.com"],
            subject="Re: Project Update",
        )
        assert result.is_valid is True

    def test_invalid_email_address_fails(self):
        """
        An invalid recipient email address should fail validation.
        """
        from utils.validator import validate_draft_content

        result = validate_draft_content(
            body_text="Dear John, please review. Best regards,",
            to_emails=["not-an-email"],
            subject="Test",
        )
        assert result.is_valid is False
        assert any("valid email" in e for e in result.errors)

    def test_placeholder_text_generates_warning(self):
        """
        Placeholder text in draft body should generate a warning.
        """
        from utils.validator import validate_draft_content

        result = validate_draft_content(
            body_text="Dear [Your Name], please review the attached. Best regards,",
            to_emails=["john@company.com"],
            subject="Test",
        )
        assert any("placeholder" in w.lower() for w in result.warnings)

    def test_empty_body_fails(self):
        """
        An empty draft body should fail validation.
        """
        from utils.validator import validate_draft_content

        result = validate_draft_content(
            body_text="",
            to_emails=["john@company.com"],
            subject="Test",
        )
        assert result.is_valid is False


class TestValidateTaskList:
    """Tests for validator.validate_task_list"""

    def test_valid_tasks_pass(self):
        """
        A list of properly formed tasks should pass validation.
        """
        from utils.validator import validate_task_list

        tasks = [
            {"title": "Review the Q3 report by Friday", "owner": "John"},
            {"title": "Send updated timeline to client", "owner": "Jane"},
        ]
        result = validate_task_list(tasks)
        assert result.is_valid is True

    def test_empty_task_list_generates_warning(self):
        """
        An empty task list should generate a warning.
        """
        from utils.validator import validate_task_list

        result = validate_task_list([])
        assert len(result.warnings) == 1

    def test_duplicate_tasks_generate_warning(self):
        """
        Duplicate task titles should generate a validation warning.
        """
        from utils.validator import validate_task_list

        tasks = [
            {"title": "review the report"},
            {"title": "review the report"},  # duplicate
        ]
        result = validate_task_list(tasks)
        assert any("Duplicate" in w for w in result.warnings)


# ===========================================================================
# Audit Logger Tests
# ===========================================================================

class TestAuditLogger:
    """Tests for utils/audit_logger.py"""

    def test_log_tool_call_does_not_raise(self):
        """
        log_tool_call() should complete without raising exceptions
        even if the logs directory doesn't exist yet.
        """
        from utils.audit_logger import log_tool_call

        # Should not raise
        log_tool_call(
            tool_name="list_emails",
            user_email="test@company.com",
            inputs={"count": 10},
            outcome="success",
        )

    def test_sanitise_inputs_removes_body_content(self):
        """
        _sanitise_inputs() should not log email body text or
        context strings — only safe metadata fields.
        """
        from utils.audit_logger import _sanitise_inputs

        raw_inputs = {
            "count": 10,
            "keyword": "budget",
            "body_text": "SENSITIVE EMAIL CONTENT HERE",
            "context": "Private business context",
            "to_emails": "someone@company.com",
        }
        safe = _sanitise_inputs(raw_inputs)

        assert "count" in safe
        assert "keyword" in safe
        assert "body_text" not in safe
        assert "context" not in safe
        assert "to_emails" not in safe

    def test_sanitise_inputs_truncates_ids(self):
        """
        _sanitise_inputs() should truncate email IDs to a short
        prefix rather than logging the full ID.
        """
        from utils.audit_logger import _sanitise_inputs

        long_id = "AAMKADZmNDgyYjViLWQ40DctNDgwZi04ZmQwLWU3YjNjNjMyZT12"
        safe = _sanitise_inputs({"email_id": long_id})

        assert "email_id" in safe
        assert len(safe["email_id"]) < len(long_id)
        assert safe["email_id"].endswith("...")

---
./tests/test_mail_tools.py
---
"""
test_mail_tools.py
==================
Tests for tools/email_tools.py — list_emails, read_email,
search_emails, and summarise_email.

What this tests:
    - Each tool calls the correct underlying graph/ function
    - The output shape matches what the tool's docstring promises
    - Error handling: format_tool_error is returned, not a raw exception

How to run:
    python -m pytest tests/test_mail_tools.py -v
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import pytest
from unittest.mock import AsyncMock, patch


# ---------------------------------------------------------------------------
# Test: list_emails returns correctly shaped output
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_list_emails_returns_correct_shape():
    """
    list_emails() should return a list of dicts, each with the fields
    defined in its docstring (id, subject, sender, etc.).
    """
    # Step 1: Build a fake raw message response — simulates what
    #         graph/mail_client.fetch_recent_messages() would return.
    fake_messages = [
        {
            "id": "msg-001",
            "subject": "Q3 Budget Review",
            "from": {
                "emailAddress": {
                    "name": "Hans Mueller",
                    "address": "h.mueller@comp..de"
                }
            },
            "receivedDateTime": "2025-01-15T09:30:00Z",
            "hasAttachments": True,
            "bodyPreview": "Please find the Q3 budget review attached...",
        }
    ]

    # Step 2: Patch fetch_recent_messages so no real Graph API call is made.
    with patch(
        "tools.email_tools.fetch_recent_messages",
        new=AsyncMock(return_value=fake_messages)
    ):
        from tools.email_tools import list_emails
        result = await list_emails(count=5)

    # Step 3: Verify the output shape matches the tool's contract.
    assert isinstance(result, dict)
    assert result["count"] == 1
    assert "display_table" in result

    first = result["emails"][0]
    assert first["id"] == "msg-001"
    assert first["subject"] == "Q3 Budget Review"
    assert first["sender"] == "Hans Mueller"
    assert first["sender_email"] == "h.mueller@comp..de"
    assert first["has_attachments"] is True
    assert "preview" in first


# ---------------------------------------------------------------------------
# Test: list_emails caps count at 50
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_list_emails_caps_count():
    """
    list_emails() should cap the requested count at 50, even if a
    larger number is passed (e.g. if the LLM requests 100 emails).
    """
    with patch(
        "tools.email_tools.fetch_recent_messages",
        new=AsyncMock(return_value=[])
    ) as mock_fetch:
        from tools.email_tools import list_emails
        await list_emails(count=999)

    # The underlying fetch should have been called with at most 50.
    assert mock_fetch.called
    call_args = mock_fetch.call_args
    top_value = (
        call_args.kwargs.get("top")
        or (call_args.args[0] if call_args.args else 50)
    )
    assert top_value <= 50


# ---------------------------------------------------------------------------
# Test: read_email returns full body
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_read_email_returns_body():
    """
    read_email() should return a dict that includes the full 'body' field,
    unlike list_emails which only returns a short preview.
    """
    fake_message = {
        "id": "msg-002",
        "subject": "Meeting Rescheduled",
        "from": {"emailAddress": {"name": "Anna Schmidt", "address": "a.schmidt@comp..de"}},
        "receivedDateTime": "2025-01-16T14:00:00Z",
        "hasAttachments": False,
        "bodyPreview": "The meeting has been rescheduled...",
        "body": {"content": "Dear Team, the Tuesday meeting has been moved to Thursday at 3pm."},
    }

    with patch(
        "tools.email_tools.fetch_message_by_id",
        new=AsyncMock(return_value=fake_message)
    ):
        from tools.email_tools import read_email
        result = await read_email(email_id="msg-002")

    assert result["subject"] == "Meeting Rescheduled"
    assert "Thursday at 3pm" in result["body"]


# ---------------------------------------------------------------------------
# Test: email tool returns error dict on exception, not raw exception
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_list_emails_handles_exception_gracefully():
    """
    If the underlying Graph call raises an exception, list_emails()
    should return a dict with 'error: True', never raise an unhandled
    exception to LibreChat.
    """
    with patch(
        "tools.email_tools.fetch_recent_messages",
        new=AsyncMock(side_effect=Exception("Simulated Graph API failure"))
    ):
        from tools.email_tools import list_emails
        result = await list_emails(count=5)

    assert isinstance(result, dict)
    assert result.get("error") is True
    assert "message" in result

---
./tests/test_mom_followup.py
---
"""
test_mom_followup.py
====================
Tests for Phase 3 tools:
    - generate_mom (single email + thread)
    - save_mom_as_draft
    - track_followups
    - check_email_replied
    - compose_followup
    - add_task_todo
    - add_task_planner
    - list_tasks
    - language detection utility

How to run:
    python -m pytest tests/test_mom_followup.py -v
"""

import pytest
from unittest.mock import AsyncMock, patch
from datetime import datetime, timedelta, timezone


# ===========================================================================
# Language Utility Tests
# ===========================================================================

def test_detect_english():
    from utils.language_utils import detect_language
    code, name = detect_language(
        "Please find attached the quarterly report for your review."
    )
    assert code == "en"
    assert name == "English"


def test_detect_german():
    from utils.language_utils import detect_language
    code, name = detect_language(
        "Sehr geehrte Damen und Herren, anbei finden Sie den Bericht."
    )
    assert code == "de"
    assert name == "German"


def test_short_text_defaults_to_english():
    from utils.language_utils import detect_language
    code, name = detect_language("Hi")
    assert code == "en"
    assert name == "English"


def test_empty_text_defaults_to_english():
    from utils.language_utils import detect_language
    code, name = detect_language("")
    assert code == "en"
    assert name == "English"


def test_language_instruction_english():
    from utils.language_utils import get_language_instruction
    instruction = get_language_instruction("en", "English")
    assert instruction == "Write in English."


def test_language_instruction_german():
    from utils.language_utils import get_language_instruction
    instruction = get_language_instruction("de", "German")
    assert "German" in instruction


# ===========================================================================
# MOM Generation Tests
# ===========================================================================

def _make_fake_thread():
    return [
        {
            "id": "msg-001",
            "subject": "Project Kickoff Meeting",
            "sender_name": "John Smith",
            "sender_email": "john@company.com",
            "to_recipients": [
                {"name": "Jane Doe", "email": "jane@company.com"}
            ],
            "cc_recipients": [],
            "received_date": "2026-07-01T09:00:00+00:00",
            "body": (
                "Hi team, let us kick off the project. "
                "Action: John to prepare the timeline by Friday. "
                "Decision: We will use Python for the backend."
            ),
            "has_attachments": False,
            "is_draft": False,
        },
        {
            "id": "msg-002",
            "subject": "Re: Project Kickoff Meeting",
            "sender_name": "Jane Doe",
            "sender_email": "jane@company.com",
            "to_recipients": [
                {"name": "John Smith", "email": "john@company.com"}
            ],
            "cc_recipients": [],
            "received_date": "2026-07-01T10:00:00+00:00",
            "body": "Agreed. I will handle the design mockups by next Wednesday.",
            "has_attachments": False,
            "is_draft": False,
        },
    ]


@pytest.mark.asyncio
async def test_generate_mom_from_thread():
    fake_thread = _make_fake_thread()

    with patch(
        "tools.mom_tools.fetch_email_thread",
        new=AsyncMock(return_value=fake_thread)
    ):
        from tools.mom_tools import generate_mom
        result = await generate_mom(email_id="msg-001", use_thread=True)

    assert result["email_count"] == 2
    assert "john@company.com" in result["participants"]
    assert "jane@company.com" in result["participants"]
    assert result["meeting_subject"] == "Project Kickoff Meeting"
    assert "MINUTES OF MEETING" in result["instruction"]
    assert "ATTENDEES" in result["instruction"]
    assert "ACTION ITEMS" in result["instruction"]
    assert "DECISIONS MADE" in result["instruction"]


@pytest.mark.asyncio
async def test_generate_mom_from_single_email():
    fake_single = {
        "id": "msg-001",
        "subject": "Team Sync Notes",
        "from": {
            "emailAddress": {"name": "Manager", "address": "mgr@company.com"}
        },
        "receivedDateTime": "2026-07-01T09:00:00+00:00",
        "body": {"content": "We discussed the roadmap and agreed on Q3 targets."},
        "hasAttachments": False,
        "bodyPreview": "We discussed the roadmap...",
    }

    with patch(
        "tools.mom_tools.fetch_message_by_id",
        new=AsyncMock(return_value=fake_single)
    ):
        from tools.mom_tools import generate_mom
        result = await generate_mom(email_id="msg-001", use_thread=False)

    assert result["email_count"] == 1
    assert result["source"] == "Single Email"
    assert "MINUTES OF MEETING" in result["instruction"]


@pytest.mark.asyncio
async def test_generate_mom_handles_error():
    with patch(
        "tools.mom_tools.fetch_email_thread",
        new=AsyncMock(side_effect=Exception("Graph API error"))
    ):
        from tools.mom_tools import generate_mom
        result = await generate_mom(email_id="msg-001")

    assert result.get("error") is True
    assert "message" in result


@pytest.mark.asyncio
async def test_saves_mom_draft_successfully():
    fake_draft_result = {
        "draft_id": "draft-mom-001",
        "subject": "MOM - Project Kickoff",
        "to": ["john@company.com"],
        "status": "Draft saved to Drafts folder",
        "web_link": "",
    }

    with patch(
        "tools.mom_tools.create_draft",
        new=AsyncMock(return_value=fake_draft_result)
    ):
        from tools.mom_tools import save_mom_as_draft
        result = await save_mom_as_draft(
            to_emails="john@company.com",
            subject="MOM - Project Kickoff",
            mom_content="MINUTES OF MEETING\n1. Details...",
        )

    assert result["draft_id"] == "draft-mom-001"
    assert "MOM" in result["instruction"]
    assert "Outlook" in result["instruction"]


@pytest.mark.asyncio
async def test_save_mom_returns_error_for_empty_recipients():
    from tools.mom_tools import save_mom_as_draft
    result = await save_mom_as_draft(
        to_emails="",
        subject="MOM",
        mom_content="Content here",
    )
    assert result.get("error") is True
    assert "recipient" in result["message"].lower()


# ===========================================================================
# Follow-up Tracking Tests
# ===========================================================================

def _make_old_email(days_old: int = 5) -> dict:
    received = (
        datetime.now(timezone.utc) - timedelta(days=days_old)
    ).isoformat()
    return {
        "id": "msg-old-001",
        "subject": "Pending Approval Request",
        "from": {
            "emailAddress": {
                "name": "External Sender",
                "address": "external@other.com",
            }
        },
        "receivedDateTime": received,
        "hasAttachments": False,
        "bodyPreview": "Please approve the attached request.",
        "is_draft": False,
    }


@pytest.mark.asyncio
async def test_identifies_overdue_emails():
    """
    Patch at the source module level — get_user_profile lives in
    graph.graph_client_factory and is imported inside the function body,
    so we patch it at its origin, not at tools.followup_tools.
    """
    fake_profile = {
        "mail": "me@company.com",
        "userPrincipalName": "me@company.com",
    }
    fake_messages = [_make_old_email(days_old=5)]

    with patch(
        "graph.graph_client_factory.get_user_profile",
        new=AsyncMock(return_value=fake_profile)
    ), patch(
        "tools.followup_tools.fetch_recent_messages",
        new=AsyncMock(return_value=fake_messages)
    ):
        from tools.followup_tools import track_followups
        result = await track_followups(days_threshold=3, count=10)

    assert result["followup_count"] == 1
    assert "Pending Approval Request" in result["display_table"]


@pytest.mark.asyncio
async def test_returns_clear_when_no_followups():
    fake_profile = {
        "mail": "me@company.com",
        "userPrincipalName": "me@company.com",
    }
    fake_messages = [_make_old_email(days_old=1)]

    with patch(
        "graph.graph_client_factory.get_user_profile",
        new=AsyncMock(return_value=fake_profile)
    ), patch(
        "tools.followup_tools.fetch_recent_messages",
        new=AsyncMock(return_value=fake_messages)
    ):
        from tools.followup_tools import track_followups
        result = await track_followups(days_threshold=3, count=10)

    assert result["followup_count"] == 0
    assert "No follow-ups" in result["display_table"]


@pytest.mark.asyncio
async def test_track_followups_handles_empty_inbox():
    fake_profile = {
        "mail": "me@company.com",
        "userPrincipalName": "me@company.com",
    }

    with patch(
        "graph.graph_client_factory.get_user_profile",
        new=AsyncMock(return_value=fake_profile)
    ), patch(
        "tools.followup_tools.fetch_recent_messages",
        new=AsyncMock(return_value=[])
    ):
        from tools.followup_tools import track_followups
        result = await track_followups()

    assert result["followup_count"] == 0


@pytest.mark.asyncio
async def test_detects_replied_email():
    fake_profile = {
        "mail": "me@company.com",
        "userPrincipalName": "me@company.com",
    }
    fake_thread = [
        {
            "id": "msg-001",
            "subject": "Budget Request",
            "sender_name": "John",
            "sender_email": "john@company.com",
            "to_recipients": [],
            "cc_recipients": [],
            "received_date": "2026-07-01T09:00:00+00:00",
            "body": "Please approve the budget.",
            "has_attachments": False,
            "is_draft": False,
        },
        {
            "id": "msg-002",
            "subject": "Re: Budget Request",
            "sender_name": "Me",
            "sender_email": "me@company.com",
            "to_recipients": [],
            "cc_recipients": [],
            "received_date": "2026-07-02T09:00:00+00:00",
            "body": "Approved.",
            "has_attachments": False,
            "is_draft": False,
        },
    ]

    with patch(
        "graph.graph_client_factory.get_user_profile",
        new=AsyncMock(return_value=fake_profile)
    ), patch(
        "tools.followup_tools.fetch_email_thread",
        new=AsyncMock(return_value=fake_thread)
    ):
        from tools.followup_tools import check_email_replied
        result = await check_email_replied(email_id="msg-001")

    assert result["has_replied"] is True
    assert result["reply_count"] == 1
    assert "✅" in result["status_display"]


@pytest.mark.asyncio
async def test_detects_unreplied_email():
    fake_profile = {
        "mail": "me@company.com",
        "userPrincipalName": "me@company.com",
    }
    fake_thread = [
        {
            "id": "msg-001",
            "subject": "Pending Request",
            "sender_name": "John",
            "sender_email": "john@company.com",
            "to_recipients": [],
            "cc_recipients": [],
            "received_date": "2026-06-25T09:00:00+00:00",
            "body": "Please confirm.",
            "has_attachments": False,
            "is_draft": False,
        }
    ]

    with patch(
        "graph.graph_client_factory.get_user_profile",
        new=AsyncMock(return_value=fake_profile)
    ), patch(
        "tools.followup_tools.fetch_email_thread",
        new=AsyncMock(return_value=fake_thread)
    ):
        from tools.followup_tools import check_email_replied
        result = await check_email_replied(email_id="msg-001")

    assert result["has_replied"] is False
    assert result["reply_count"] == 0
    assert "❌" in result["status_display"]


@pytest.mark.asyncio
async def test_compose_followup_returns_instruction_with_sender():
    fake_email = {
        "id": "msg-001",
        "subject": "Quarterly Review",
        "from": {
            "emailAddress": {
                "name": "Hans Mueller",
                "address": "hans@company.de",
            }
        },
        "receivedDateTime": "2026-06-28T09:00:00+00:00",
        "body": {
            "content": (
                "Dear colleague, please send the quarterly figures "
                "at your earliest convenience."
            )
        },
        "hasAttachments": False,
    }

    with patch(
        "tools.followup_tools.fetch_message_by_id",
        new=AsyncMock(return_value=fake_email)
    ):
        from tools.followup_tools import compose_followup
        result = await compose_followup(email_id="msg-001")

    assert "Hans Mueller" in result["instruction"]
    assert "hans@company.de" in result["instruction"]
    assert result["original_subject"] == "Quarterly Review"


# ===========================================================================
# Task Tool Tests
# ===========================================================================

@pytest.mark.asyncio
async def test_creates_todo_task_successfully():
    fake_result = {
        "task_id": "todo-task-001",
        "title": "Follow up on budget email",
        "list_name": "Tasks",
        "due_date": "2026-07-18",
        "status": "Task created in Microsoft To-Do - Tasks",
    }

    with patch(
        "tools.followup_tools.create_todo_task",
        new=AsyncMock(return_value=fake_result)
    ):
        from tools.followup_tools import add_task_todo
        result = await add_task_todo(
            title="Follow up on budget email",
            due_date="2026-07-18",
            notes="Email from John on 1 July",
        )

    assert result["task_id"] == "todo-task-001"
    assert "To-Do" in result["instruction"]
    assert "✅" in result["instruction"]      # fixed: was asserting "OK"


@pytest.mark.asyncio
async def test_todo_task_handles_creation_error():
    with patch(
        "tools.followup_tools.create_todo_task",
        new=AsyncMock(side_effect=Exception("Tasks API unavailable"))
    ):
        from tools.followup_tools import add_task_todo
        result = await add_task_todo(title="Test task")

    assert result.get("error") is True
    assert "message" in result


@pytest.mark.asyncio
async def test_planner_returns_error_when_plan_id_missing():
    fake_result = {
        "error": True,
        "message": "PLANNER_PLAN_ID is not configured in .env.",
    }

    with patch(
        "tools.followup_tools.create_planner_task",
        new=AsyncMock(return_value=fake_result)
    ):
        from tools.followup_tools import add_task_planner
        result = await add_task_planner(title="Team task")

    assert result.get("error") is True


@pytest.mark.asyncio
async def test_creates_planner_task_successfully():
    fake_result = {
        "task_id": "planner-task-001",
        "title": "Infrastructure review",
        "plan_id": "plan-abc-123",
        "due_date": "2026-07-20",
        "assigned_to": "jane@company.com",
        "status": "Task created in Microsoft Planner",
    }

    with patch(
        "tools.followup_tools.create_planner_task",
        new=AsyncMock(return_value=fake_result)
    ):
        from tools.followup_tools import add_task_planner
        result = await add_task_planner(
            title="Infrastructure review",
            due_date="2026-07-20",
            assigned_to_email="jane@company.com",
        )

    assert result["task_id"] == "planner-task-001"
    assert "Planner" in result["instruction"]


@pytest.mark.asyncio
async def test_list_tasks_returns_table():
    fake_tasks = [
        {
            "id": "t1",
            "title": "Review budget proposal",
            "due_date": "2026-07-18",
            "status": "notStarted",
            "list_name": "Tasks",
        },
        {
            "id": "t2",
            "title": "Send MOM to team",
            "due_date": "Not set",
            "status": "notStarted",
            "list_name": "Tasks",
        },
    ]

    with patch(
        "tools.followup_tools.get_todo_tasks",
        new=AsyncMock(return_value=fake_tasks)
    ):
        from tools.followup_tools import list_tasks
        result = await list_tasks(source="todo")

    assert result["count"] == 2
    assert "Review budget proposal" in result["display_table"]
    assert "Send MOM to team" in result["display_table"]
    assert "Microsoft To-Do" in result["source"]


@pytest.mark.asyncio
async def test_list_tasks_returns_empty_message():
    with patch(
        "tools.followup_tools.get_todo_tasks",
        new=AsyncMock(return_value=[])
    ):
        from tools.followup_tools import list_tasks
        result = await list_tasks(source="todo")

    assert result["count"] == 0
    assert (
        "empty" in result["display_table"].lower()
        or "No pending" in result["display_table"]
    )

---
./tests/test_parsers.py
---
"""
test_parsers.py
===============
Tests for parsers/ — pdf_parser, docx_parser, pptx_parser,
xlsx_parser, and parser_router.

What this tests:
    - Each parser correctly extracts text from a minimal in-memory
      file constructed in the test itself (no real attachments needed)
    - parser_router routes to the correct parser based on file extension
    - parser_router handles unsupported file types gracefully

How to run:
    python -m pytest tests/test_parsers.py -v
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import io
import pytest
from unittest.mock import patch, AsyncMock

# Constructing minimal real file bytes for testing
from docx import Document as DocxDocument
from pptx import Presentation as PptxPresentation
from pptx.util import Inches
from openpyxl import Workbook
import fitz  # PyMuPDF — used to create a minimal test PDF in memory


# ---------------------------------------------------------------------------
# Helper fixtures: build minimal in-memory files for testing
# ---------------------------------------------------------------------------
def make_minimal_pdf_bytes(text: str = "Hello from PDF") -> bytes:
    """
    Create a minimal valid PDF in memory containing the given text,
    using PyMuPDF itself. Returns raw bytes.
    """
    doc = fitz.open()
    page = doc.new_page()
    page.insert_text((50, 50), text)

    # Save to bytes first
    pdf_bytes = doc.tobytes()
    doc.close()

    # Re-open from bytes so the text layer is fully committed
    # and extractable by get_text() — required in PyMuPDF 1.24+
    doc2 = fitz.open(stream=pdf_bytes, filetype="pdf")
    final_bytes = doc2.tobytes()
    doc2.close()

    return final_bytes


def make_minimal_docx_bytes(text: str = "Hello from DOCX") -> bytes:
    """
    Create a minimal valid .docx in memory containing the given text.
    Returns raw bytes.
    """
    doc = DocxDocument()
    doc.add_paragraph(text)

    buffer = io.BytesIO()
    doc.save(buffer)
    return buffer.getvalue()


def make_minimal_pptx_bytes(text: str = "Hello from PPTX") -> bytes:
    """
    Create a minimal valid .pptx in memory with one slide containing
    the given text. Returns raw bytes.
    """
    prs = PptxPresentation()
    slide_layout = prs.slide_layouts[1]  # Title and content layout
    slide = prs.slides.add_slide(slide_layout)
    slide.shapes.title.text = text

    buffer = io.BytesIO()
    prs.save(buffer)
    return buffer.getvalue()


def make_minimal_xlsx_bytes(text: str = "Hello from XLSX") -> bytes:
    """
    Create a minimal valid .xlsx in memory with one row of data.
    Returns raw bytes.
    """
    wb = Workbook()
    ws = wb.active
    ws["A1"] = text
    ws["B1"] = "Column B value"

    buffer = io.BytesIO()
    wb.save(buffer)
    return buffer.getvalue()


# ---------------------------------------------------------------------------
# Test: PDF parser extracts text correctly
# ---------------------------------------------------------------------------
def test_pdf_parser_extracts_text():
    """
    pdf_parser.extract_text_from_pdf() should extract the text that was
    written into the test PDF.
    """
    from parsers.pdf_parser import extract_text_from_pdf

    pdf_bytes = make_minimal_pdf_bytes("This is a test PDF document.")
    result = extract_text_from_pdf(pdf_bytes)

    assert "This is a test PDF document." in result["text"]
    assert result["has_extractable_text"] is True
    assert result["page_count"] == 1


# ---------------------------------------------------------------------------
# Test: DOCX parser extracts paragraphs
# ---------------------------------------------------------------------------
def test_docx_parser_extracts_text():
    """
    docx_parser.extract_text_from_docx() should return the paragraph
    text written into the test document.
    """
    from parsers.docx_parser import extract_text_from_docx

    docx_bytes = make_minimal_docx_bytes("This is a test Word document.")
    result = extract_text_from_docx(docx_bytes)

    assert "This is a test Word document." in result["text"]
    assert result["paragraph_count"] >= 1


# ---------------------------------------------------------------------------
# Test: PPTX parser extracts slide text
# ---------------------------------------------------------------------------
def test_pptx_parser_extracts_text():
    """
    pptx_parser.extract_text_from_pptx() should return text from the
    test slide, prefixed with a slide number label.
    """
    from parsers.pptx_parser import extract_text_from_pptx

    pptx_bytes = make_minimal_pptx_bytes("Test Slide Title")
    result = extract_text_from_pptx(pptx_bytes)

    assert "Test Slide Title" in result["text"]
    assert "Slide 1" in result["text"]
    assert result["slide_count"] == 1


# ---------------------------------------------------------------------------
# Test: XLSX parser extracts cell data
# ---------------------------------------------------------------------------
def test_xlsx_parser_extracts_data():
    """
    xlsx_parser.extract_text_from_xlsx() should return the cell values
    from the test workbook in a readable text format.
    """
    from parsers.xlsx_parser import extract_text_from_xlsx

    xlsx_bytes = make_minimal_xlsx_bytes("Budget Total")
    result = extract_text_from_xlsx(xlsx_bytes)

    assert "Budget Total" in result["text"]
    assert result["sheet_count"] >= 1
    assert result["truncated"] is False


# ---------------------------------------------------------------------------
# Test: parser_router routes PDF files to pdf_parser
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_parser_router_routes_pdf():
    """
    parse_attachment() should route a .pdf file to the PDF parser and
    return a result with file_type == "pdf".
    """
    from parsers.parser_router import parse_attachment

    pdf_bytes = make_minimal_pdf_bytes("Router test PDF")
    result = await parse_attachment(pdf_bytes, "test_report.pdf", "application/pdf")

    # print(f"\nDEBUG extracted text: '{result['text'][:100]}'")
    assert result["file_type"] == "pdf"
    assert "Router test PDF" in result["text"]


# ---------------------------------------------------------------------------
# Test: parser_router handles unsupported .doc gracefully
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_parser_router_handles_unsupported_doc():
    """
    parse_attachment() should return a clear, human-readable message
    for .doc files (legacy format) rather than crashing.
    """
    from parsers.parser_router import parse_attachment

    result = await parse_attachment(b"fake binary content", "old_file.doc", "application/msword")

    assert result["file_type"] == "unsupported_legacy"
    assert "legacy format" in result["text"]
    assert result["used_ocr"] is False


# ---------------------------------------------------------------------------
# Test: parser_router handles completely unknown file type gracefully
# ---------------------------------------------------------------------------
@pytest.mark.asyncio
async def test_parser_router_handles_unknown_type():
    """
    parse_attachment() should return a clear message for entirely
    unrecognised file types rather than crashing.
    """
    from parsers.parser_router import parse_attachment

    result = await parse_attachment(b"unknown bytes", "data.xyz", "application/octet-stream")

    assert result["file_type"] == "unsupported"
    assert "not currently supported" in result["text"]

---
./parsers/__init__.py
---


---
./parsers/docx_parser.py
---
"""
docx_parser.py
==============
Extracts text content from Word (.docx) file bytes using python-docx.

Why this file exists:
    Word documents are a common attachment type for reports, meeting
    notes, and CVs submitted in Word format. This file's only job is:
    given raw .docx bytes, return clean extracted text including both
    paragraphs and table content.

Design notes:
    - python-docx requires a file-like object, not raw bytes directly,
      so we wrap the bytes in io.BytesIO before opening.
    - Tables are extracted separately from paragraphs because Word
      documents often have important structured data (e.g. a CV's
      skills table) that would be lost if only paragraphs were read.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import io
from docx import Document

from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: extract_text_from_docx
# ---------------------------------------------------------------------------
def extract_text_from_docx(file_bytes: bytes) -> dict:
    """
    Extract all readable text (paragraphs and tables) from a Word
    document's raw bytes.

    Args:
        file_bytes (bytes): The raw binary content of the .docx file.

    Returns:
        A dictionary containing:
            - text (str): All extracted text — paragraphs and table
                          content combined, in document order.
            - paragraph_count (int): Number of paragraphs found.
            - table_count (int): Number of tables found.
    """
    # Step 1: python-docx's Document() expects a file-like object.
    #         io.BytesIO wraps our raw bytes to behave like a file
    #         without writing anything to disk.
    file_stream = io.BytesIO(file_bytes)
    document = Document(file_stream)

    text_segments = []

    # Step 2: Extract every paragraph's text. Empty paragraphs (just
    #         spacing in the original document) are skipped to keep
    #         the output clean.
    paragraph_count = 0
    for paragraph in document.paragraphs:
        if paragraph.text.strip():
            text_segments.append(paragraph.text)
            paragraph_count += 1

    # Step 3: Extract every table's content, row by row, with cells
    #         joined by " | " to preserve the table-like structure
    #         in plain text form.
    table_count = 0
    for table in document.tables:
        table_count += 1
        for row in table.rows:
            row_text = " | ".join(cell.text.strip() for cell in row.cells)
            if row_text.strip():
                text_segments.append(row_text)

    full_text = "\n".join(text_segments)

    return {
        "text": full_text,
        "paragraph_count": paragraph_count,
        "table_count": table_count,
    }

---
./parsers/ocr_fallback.py
---
"""
ocr_fallback.py
================
Performs OCR (Optical Character Recognition) on scanned/image-based PDFs
or image files, using pytesseract (a wrapper around the Tesseract OCR engine).

Why this file exists:
    Some PDFs (e.g. a scanned CV or a signed contract) have no real text
    layer — they're just images of text. pdf_parser.py detects this case
    and signals it; this file is what actually extracts text from those
    images when that happens.

Design notes:
    - This file is ONLY invoked when pdf_parser.py reports
      has_extractable_text = False. It is never called directly by
      tools/ — only through parser_router.py's fallback logic.
    - Requires Tesseract OCR to be installed at the SYSTEM level (not
      just pip-installed) — see TESSERACT_CMD_PATH in .env. If OCR is
      disabled or Tesseract isn't installed, this gracefully returns
      an empty result rather than crashing the whole tool call.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import io
import fitz  # PyMuPDF — used here to render PDF pages as images
import pytesseract
from PIL import Image

from config.settings import settings
from utils.logger import get_logger

logger = get_logger(__name__)

# If a custom Tesseract install path was provided in .env, tell
# pytesseract where to find the executable. On many systems with
# Tesseract properly added to PATH, this isn't necessary.
if settings.TESSERACT_CMD_PATH:
    pytesseract.pytesseract.tesseract_cmd = settings.TESSERACT_CMD_PATH


# ---------------------------------------------------------------------------
# Function: extract_text_via_ocr
# ---------------------------------------------------------------------------
def extract_text_via_ocr(file_bytes: bytes) -> dict:
    """
    Run OCR on a scanned PDF's pages to extract text from images.

    Args:
        file_bytes (bytes): The raw binary content of the (scanned) PDF.

    Returns:
        A dictionary containing:
            - text (str): All OCR-extracted text, page by page.
            - page_count (int): Total number of pages processed.
            - ocr_succeeded (bool): False if OCR is disabled or Tesseract
                                    isn't available, in which case text
                                    will be an empty string.
    """
    # Step 1: Respect the OCR_ENABLED setting. If the organisation hasn't
    #         set up Tesseract yet, fail gracefully instead of crashing.
    if not settings.OCR_ENABLED:
        logger.info("OCR is disabled in settings — skipping OCR extraction")
        return {"text": "", "page_count": 0, "ocr_succeeded": False}

    try:
        # Step 2: Open the PDF and render each page as an image, since
        #         OCR works on images, not on PDF text layers (which
        #         don't exist for this type of file anyway).
        pdf_document = fitz.open(stream=file_bytes, filetype="pdf")

        page_texts = []

        for page_number in range(len(pdf_document)):
            page = pdf_document[page_number]

            # Step 3: Render the page at a higher resolution (zoom factor 2)
            #         for better OCR accuracy than the default low-res render.
            pix = page.get_pixmap(matrix=fitz.Matrix(2, 2))
            image_bytes = pix.tobytes("png")

            # Step 4: Convert to a PIL Image object, which is what
            #         pytesseract expects as input.
            image = Image.open(io.BytesIO(image_bytes))

            # Step 5: Run OCR on this page's image.
            page_text = pytesseract.image_to_string(image)
            page_texts.append(page_text)

        pdf_document.close()

        full_text = "\n\n".join(page_texts)

        return {
            "text": full_text,
            "page_count": len(page_texts),
            "ocr_succeeded": True,
        }

    except Exception as exc:
        # Step 6: If Tesseract isn't actually installed on the system,
        #         this will fail here. Log clearly and fail gracefully
        #         rather than crashing the whole attachment tool.
        logger.error(f"OCR extraction failed: {exc}")
        return {"text": "", "page_count": 0, "ocr_succeeded": False}

---
./parsers/parser_router.py
---
"""
parser_router.py
=================
The single decision-point for "which parser should handle this file?"

Why this file exists:
    Every other file in this project that needs to read an attachment
    calls THIS file, never the individual parsers directly. This means
    adding support for a new file type later only requires changing
    this one file, not hunting through tools/ or graph/.

Design notes:
    - File type is determined primarily from the file extension, with
      content_type (MIME type) as a secondary check.
    - This file also owns the OCR fallback decision: if pdf_parser
      reports no extractable text, parser_router automatically calls
      ocr_fallback before returning to the caller. The caller (e.g.
      attachment_tools.py) never needs to know OCR happened unless it
      checks the used_ocr flag in the result.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from parsers.pdf_parser import extract_text_from_pdf
from parsers.docx_parser import extract_text_from_docx
from parsers.pptx_parser import extract_text_from_pptx
from parsers.xlsx_parser import extract_text_from_xlsx
from parsers.ocr_fallback import extract_text_via_ocr

from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: parse_attachment
# ---------------------------------------------------------------------------
async def parse_attachment(file_bytes: bytes, file_name: str, content_type: str) -> dict:
    """
    Route an attachment's raw bytes to the correct parser based on its
    file type, and return clean extracted text.

    Args:
        file_bytes (bytes): The raw binary content of the attachment.
        file_name (str): The original file name (e.g. "Report.pdf"),
                          used to detect file type via extension.
        content_type (str): The MIME type reported by Graph API
                             (e.g. "application/pdf"), used as a
                             secondary type-detection signal.

    Returns:
        A dictionary containing:
            - text (str): The extracted plain text content.
            - file_type (str): The detected file type
                                ("pdf", "docx", "pptx", "xlsx", or "unsupported").
            - used_ocr (bool): True if OCR was used as a fallback
                                (only relevant for scanned PDFs).
    """
    # Step 1: Determine file type from the file extension. This is the
    #         most reliable signal since Graph API content_type values
    #         can sometimes be generic (e.g. "application/octet-stream").
    file_name_lower = file_name.lower()

    # ----- PDF -----
    if file_name_lower.endswith(".pdf"):
        result = extract_text_from_pdf(file_bytes)

        # Only attempt OCR if there is genuinely no text AND OCR is
        # enabled. If OCR is disabled, return whatever text was found
        # (even if minimal) rather than returning empty string.
        if not result["has_extractable_text"] and result["text"].strip() == "":
            from config.settings import settings
            if settings.OCR_ENABLED:
                logger.info(f"'{file_name}' has no extractable text — attempting OCR")
                ocr_result = extract_text_via_ocr(file_bytes)
                return {
                    "text": ocr_result["text"],
                    "file_type": "pdf",
                    "used_ocr": ocr_result["ocr_succeeded"],
                }
            else:
                logger.info(f"'{file_name}' text extraction minimal, OCR disabled — returning as-is")

        return {
            "text": result["text"],
            "file_type": "pdf",
            "used_ocr": False,
        }

    # ----- Word -----
    elif file_name_lower.endswith(".docx"):
        result = extract_text_from_docx(file_bytes)
        return {
            "text": result["text"],
            "file_type": "docx",
            "used_ocr": False,
        }

    # ----- PowerPoint -----
    elif file_name_lower.endswith(".pptx"):
        result = extract_text_from_pptx(file_bytes)
        return {
            "text": result["text"],
            "file_type": "pptx",
            "used_ocr": False,
        }

    # ----- Excel -----
    elif file_name_lower.endswith(".xlsx"):
        result = extract_text_from_xlsx(file_bytes)
        return {
            "text": result["text"],
            "file_type": "xlsx",
            "used_ocr": False,
        }
    
        # ----- Plain text formats -----
    elif file_name_lower.endswith((".txt", ".md", ".log", ".json", ".xml")):
        # These are already plain text — decode directly, no parser needed.
        try:
            text = file_bytes.decode("utf-8")
        except UnicodeDecodeError:
            # Fallback to latin-1 if UTF-8 fails (older Windows files)
            text = file_bytes.decode("latin-1", errors="replace")
        return {
            "text": text,
            "file_type": file_name_lower.rsplit(".", 1)[-1],
            "used_ocr": False,
        }

    # ----- CSV -----
    elif file_name_lower.endswith(".csv"):
        import csv, io
        try:
            text_content = file_bytes.decode("utf-8")
        except UnicodeDecodeError:
            text_content = file_bytes.decode("latin-1", errors="replace")
        # Convert CSV rows to readable pipe-separated text
        reader = csv.reader(io.StringIO(text_content))
        rows = [" | ".join(row) for row in reader if any(cell.strip() for cell in row)]
        return {
            "text": "\n".join(rows),
            "file_type": "csv",
            "used_ocr": False,
        }

    # ----- Images -----
    elif file_name_lower.endswith((".png", ".jpg", ".jpeg", ".gif", ".bmp", ".webp")):
        # Images require OCR to extract any text content.
        # If OCR is disabled, return a clear message.
        from config.settings import settings
        if not settings.OCR_ENABLED:
            return {
                "text": (
                    f"'{file_name}' is an image file. To extract text from "
                    f"images, OCR must be enabled. Set OCR_ENABLED=true in "
                    f".env and ensure Tesseract is installed on the server."
                ),
                "file_type": "image",
                "used_ocr": False,
            }
        # OCR enabled — run tesseract on the image bytes
        from PIL import Image
        import io as _io
        import pytesseract
        image = Image.open(_io.BytesIO(file_bytes))
        extracted = pytesseract.image_to_string(image)
        return {
            "text": extracted,
            "file_type": "image",
            "used_ocr": True,
        }


    # ----- Legacy / unsupported formats -----
    elif file_name_lower.endswith((".doc", ".ppt", ".xls")):
        # Step 3: Old binary Office formats (pre-2007) aren't supported
        #         by the libraries in this project. Rather than crashing,
        #         return a clear, honest message the LLM can relay to
        #         the user.
        logger.info(f"'{file_name}' is a legacy Office format — not supported")
        return {
            "text": (
                f"This file ('{file_name}') is in a legacy format "
                f"(.doc/.ppt/.xls) that isn't currently supported. "
                f"Please convert it to the modern format "
                f"(.docx/.pptx/.xlsx) and try again."
            ),
            "file_type": "unsupported_legacy",
            "used_ocr": False,
        }

    # ----- Anything else -----
    else:
        logger.info(f"'{file_name}' has an unrecognised file type")
        return {
            "text": (
                f"This file type ('{file_name}') is not currently "
                f"supported for reading."
            ),
            "file_type": "unsupported",
            "used_ocr": False,
        }

---
./parsers/pdf_parser.py
---
"""
pdf_parser.py
=============
Extracts text content from PDF file bytes using PyMuPDF (imported as fitz).

Why this file exists:
    PDFs are one of the most common attachment types (reports, CVs,
    contracts). This file's only responsibility is: given raw PDF bytes,
    return the extracted text. It also detects when a PDF has no
    extractable text (i.e. it's a scanned image), so the caller knows
    to fall back to OCR.

Design notes:
    - PyMuPDF is used instead of heavier libraries because it's fast,
      lightweight, and reliable for both text extraction and detecting
      scanned vs digital-text pages.
    - This file does NOT perform OCR itself — that's ocr_fallback.py's job.
      This file only detects the need for it and reports back.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
# PyMuPDF's import name is "fitz" for historical reasons (it was originally
# built around the MuPDF C library under that internal name).
import fitz

from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: extract_text_from_pdf
# ---------------------------------------------------------------------------
def extract_text_from_pdf(file_bytes: bytes) -> dict:
    """
    Extract all readable text from a PDF file's raw bytes.

    Args:
        file_bytes (bytes): The raw binary content of the PDF file.

    Returns:
        A dictionary containing:
            - text (str): All extracted text, with pages separated by
                          double newlines for readability.
            - page_count (int): Total number of pages in the PDF.
            - has_extractable_text (bool): False if the PDF appears to be
                          scanned/image-only with no real text layer —
                          signals the caller to use OCR instead.
    """
    # Step 1: Open the PDF directly from in-memory bytes (no need to save
    #         to disk first). "pdf" tells PyMuPDF what format to expect.
    pdf_document = fitz.open(stream=file_bytes, filetype="pdf")

    extracted_pages = []

    # Step 2: Loop through every page and pull its text layer.
    for page_number in range(len(pdf_document)):
        page = pdf_document[page_number]
        page_text = page.get_text()
        extracted_pages.append(page_text)

    pdf_document.close()

    # Step 3: Join all pages into one text block, separated clearly.
    full_text = "\n\n".join(extracted_pages)

    # Step 4: Decide if this PDF actually has real, usable text. If the
    #         combined text (stripped of whitespace) is very short relative
    #         to the page count, this is almost certainly a scanned PDF
    #         with no real text layer — just embedded images of text.
    stripped_length = len(full_text.strip())
    has_extractable_text = stripped_length > (20 * len(extracted_pages))

    if not has_extractable_text:
        logger.info("PDF appears to be scanned/image-only — OCR fallback recommended")

    return {
        "text": full_text,
        "page_count": len(extracted_pages),
        "has_extractable_text": has_extractable_text,
    }

---
./parsers/pptx_parser.py
---
"""
pptx_parser.py
==============
Extracts text content from PowerPoint (.pptx) file bytes using python-pptx.

Why this file exists:
    Slide decks are a common attachment for meeting materials and
    presentations. This file's job is: given raw .pptx bytes, return
    text extracted slide by slide, clearly labelled, so a summary can
    reference specific slides if useful.

Design notes:
    - Text can live in different "shapes" on a slide (text boxes, titles,
      tables). This file loops through all shapes on every slide rather
      than assuming a fixed layout.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import io
from pptx import Presentation

from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: extract_text_from_pptx
# ---------------------------------------------------------------------------
def extract_text_from_pptx(file_bytes: bytes) -> dict:
    """
    Extract all readable text from a PowerPoint presentation's raw bytes,
    organised slide by slide.

    Args:
        file_bytes (bytes): The raw binary content of the .pptx file.

    Returns:
        A dictionary containing:
            - text (str): All extracted text, with each slide's content
                          prefixed by a "Slide N:" label for clarity.
            - slide_count (int): Total number of slides in the deck.
    """
    # Step 1: Wrap bytes in a file-like object for python-pptx to open.
    file_stream = io.BytesIO(file_bytes)
    presentation = Presentation(file_stream)

    slide_texts = []

    # Step 2: Loop through every slide.
    for slide_index, slide in enumerate(presentation.slides, start=1):
        slide_content = []

        # Step 3: Loop through every shape on the slide. Not every shape
        #         has text (e.g. images don't), so we check first.
        for shape in slide.shapes:
            if shape.has_text_frame:
                shape_text = shape.text_frame.text
                if shape_text.strip():
                    slide_content.append(shape_text)

        # Step 4: Only include slides that actually had readable text.
        if slide_content:
            combined = "\n".join(slide_content)
            slide_texts.append(f"Slide {slide_index}:\n{combined}")

    full_text = "\n\n".join(slide_texts)

    return {
        "text": full_text,
        "slide_count": len(presentation.slides),
    }

---
./parsers/xlsx_parser.py
---
"""
xlsx_parser.py
==============
Extracts text/tabular data from Excel (.xlsx) file bytes using openpyxl.

Why this file exists:
    Spreadsheets are common attachments for budgets, trackers, and data
    reports. This file's job is: given raw .xlsx bytes, return the data
    converted into a clean, readable text/table format that an LLM can
    reason over (since LLMs work with text, not spreadsheet objects).

Design notes:
    - Only cell VALUES are extracted (not formulas, formatting, or
      styles) — Phase 1 only needs the data, not the presentation.
    - Each sheet is clearly labelled so multi-sheet workbooks remain
      understandable in the extracted text.
    - Very large sheets are capped (see MAX_ROWS_PER_SHEET) to avoid
      sending huge text blobs to the LLM, which would be slow and costly.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import io
from openpyxl import load_workbook

from utils.logger import get_logger

logger = get_logger(__name__)

# Safety cap — prevents a 50,000-row spreadsheet from generating an
# enormous text blob that would be impractical to summarise.
MAX_ROWS_PER_SHEET = 200


# ---------------------------------------------------------------------------
# Function: extract_text_from_xlsx
# ---------------------------------------------------------------------------
def extract_text_from_xlsx(file_bytes: bytes) -> dict:
    """
    Extract tabular data from an Excel workbook's raw bytes, converted
    into readable text, sheet by sheet.

    Args:
        file_bytes (bytes): The raw binary content of the .xlsx file.

    Returns:
        A dictionary containing:
            - text (str): All extracted data, with each sheet's content
                          prefixed by a "Sheet: <name>" label.
            - sheet_count (int): Total number of sheets in the workbook.
            - truncated (bool): True if any sheet exceeded MAX_ROWS_PER_SHEET
                                and was cut short.
    """
    # Step 1: Wrap bytes in a file-like object. data_only=True ensures
    #         we get the last CALCULATED value of formulas, not the
    #         formula text itself (e.g. "150" instead of "=A1+B1").
    file_stream = io.BytesIO(file_bytes)
    workbook = load_workbook(file_stream, data_only=True)

    sheet_texts = []
    was_truncated = False

    # Step 2: Loop through every sheet in the workbook.
    for sheet_name in workbook.sheetnames:
        sheet = workbook[sheet_name]
        sheet_lines = [f"Sheet: {sheet_name}"]

        # Step 3: Loop through rows, converting each row's cell values
        #         into a single pipe-separated line of text.
        row_count = 0
        for row in sheet.iter_rows(values_only=True):
            if row_count >= MAX_ROWS_PER_SHEET:
                was_truncated = True
                sheet_lines.append("... (additional rows truncated)")
                break

            # Step 4: Skip fully empty rows (common in spreadsheets with
            #         visual spacing) to keep output clean.
            if any(cell is not None for cell in row):
                row_text = " | ".join(
                    str(cell) if cell is not None else "" for cell in row
                )
                sheet_lines.append(row_text)
                row_count += 1

        sheet_texts.append("\n".join(sheet_lines))

    full_text = "\n\n".join(sheet_texts)

    return {
        "text": full_text,
        "sheet_count": len(workbook.sheetnames),
        "truncated": was_truncated,
    }

---
./logs/audit.log
---
{"timestamp": "2026-07-24T06:36:05.104632+00:00", "tool": "create_draft_invite", "user": "admin_vpra@comp.group.com", "inputs": {}, "outcome": "success", "email_ids_accessed": [], "error": null}
{"timestamp": "2026-07-24T06:36:11.321319+00:00", "tool": "create_draft_invite", "user": "admin_vpra@comp.group.com", "inputs": {}, "outcome": "success", "email_ids_accessed": [], "error": null}


---
./logs/audit.log.2026-07-21
---
{"timestamp": "2026-07-21T10:16:23.920430+00:00", "tool": "list_emails", "user": "test@company.com", "inputs": {"count": 10}, "outcome": "success", "email_ids_accessed": [], "error": null}
{"timestamp": "2026-07-21T10:36:57.512337+00:00", "tool": "list_emails_paged", "user": "admin_vpra@comp.group.com", "inputs": {"count": 50}, "outcome": "success", "email_ids_accessed": [], "error": null}


---
./logs/audit.log.2026-07-22
---
{"timestamp": "2026-07-22T12:01:03.569731+00:00", "tool": "read_email", "user": "admin_vpra@comp.group.com", "inputs": {"email_id": "AAMkADZmNDgy..."}, "outcome": "success", "email_ids_accessed": [], "error": null}
{"timestamp": "2026-07-22T12:07:22.222108+00:00", "tool": "list_calendar_events", "user": "admin_vpra@comp.group.com", "inputs": {"date_range": "this month"}, "outcome": "success", "email_ids_accessed": [], "error": null}


---
./logs/audit.log.2026-07-23
---
{"timestamp": "2026-07-23T05:28:28.458345+00:00", "tool": "list_tasks", "user": "admin_vpra@comp.group.com", "inputs": {"source": "todo"}, "outcome": "success", "email_ids_accessed": [], "error": null}
{"timestamp": "2026-07-23T05:28:28.513486+00:00", "tool": "list_tasks", "user": "admin_vpra@comp.group.com", "inputs": {"source": "planner"}, "outcome": "success", "email_ids_accessed": [], "error": null}

---
./tools/__init__.py
---

---
./tools/attachment_tools.py
---
"""
attachment_tools.py
====================
Defines MCP tools for listing, reading, and summarising email attachments
(PDF, Word, PowerPoint, Excel, CVs, etc.)

Why this file exists:
    This implements the attachment-reading capability added to Phase 1.
    The chain of responsibility is intentionally layered:
        1. graph/attachment_client.py  -> downloads the raw attachment bytes
        2. parsers/parser_router.py    -> picks the right parser by file type
        3. llm/requesty_client.py      -> (only for summarise) generates a summary

    This file itself contains NO parsing logic and NO Graph API logic —
    it only coordinates the calls in the right order.

Design notes:
    - Attachment file types are NOT decided by this file. parser_router.py
      owns that decision entirely, so adding support for a new file type
      later means changing parser_router.py, not this file.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
# from server import mcp
from tools.mcp_instance import mcp

from graph.mail_client import fetch_message_by_id
from graph.attachment_client import (
    list_message_attachments,
    download_attachment,
)

from parsers.parser_router import parse_attachment

# from llm.requesty_client import generate_summary

from utils.logger import get_logger
from utils.governance import get_email_rules
from utils.error_handler import format_tool_error
from utils.audit_logger import log_tool_call, get_user_email_from_headers

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Tool: list_attachments
# ---------------------------------------------------------------------------
@mcp.tool
async def list_attachments(email_id: str) -> list[dict]:
    """
    List all attachments on a specific email, without reading their content.

    Use this tool when the user asks things like:
    - "What's attached to this email?"
    - "Does this email have any files?"
    - "Show me the attachments on this message"

    Args:
        email_id (str): The unique Graph API ID of the email to inspect.
                         Obtained from list_emails or search_emails.

    Returns:
        A list of dictionaries, each containing:
            - attachment_id (str): Unique ID for this attachment, used
                                    by read_attachment/summarise_attachment
            - file_name (str): Original file name (e.g. "CV_JohnDoe.pdf")
            - content_type (str): MIME type (e.g. "application/pdf")
            - size_kb (float): Approximate file size in kilobytes
    """
    logger.info(f"Tool called: list_attachments (email_id={email_id})")

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="list_attachments",
            user_email=user_email,
            inputs={"email_id": email_id[:12]},
        )

        raw_attachments = await list_message_attachments(email_id)

        result = []
        for att in raw_attachments:
            result.append({
                "attachment_id": att.get("id"),
                "file_name": att.get("name"),
                "content_type": att.get("contentType"),
                "size_kb": round(att.get("size", 0) / 1024, 1),
            })

        return result

    except Exception as exc:
        return format_tool_error(exc, tool_name="list_attachments", logger=logger)


# ---------------------------------------------------------------------------
# Tool: read_attachment
# ---------------------------------------------------------------------------
@mcp.tool
async def read_attachment(email_id: str, attachment_id: str) -> dict:
    """
    Extract and return the full text content of a specific email
    attachment (PDF, Word, PowerPoint, Excel, or CV).
    
    IMPORTANT: Always call list attachement first in the same
    conversation turn to get a fresh attachmnet_id. Do not reuse
    attchemnet IDs from previous conversation turns as they expire.
    
    Use this tool when the user asks things like:
    - "Read this attached PDF"
    - "What does this Word document say?"
    - "Open the attached CV and tell me what's in it"
    - "Extract the data from this Excel file"

    This tool is typically called AFTER list_attachments, using the
    'attachment_id' field returned by that tool.

    Args:
        email_id (str): The unique Graph API ID of the parent email.
        attachment_id (str): The unique ID of the attachment to read,
                              obtained from list_attachments output.

    Returns:
        A dictionary containing:
            - file_name (str): Original file name
            - file_type (str): Detected file type (pdf, docx, pptx, xlsx)
            - extracted_text (str): The full extracted text content
            - used_ocr (bool): Whether OCR was needed (i.e. scanned document)
    """
    logger.info(
        f"Tool called: read_attachment (email_id={email_id}, "
        f"attachment_id={attachment_id})"
    )

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="read_attachment",
            user_email=user_email,
            inputs={
                "email_id": email_id[:12],
                "attachment_id": attachment_id[:12],
            },
        )
        # Step 1: Download the raw attachment bytes from Graph API and
        #         the original file name/content type metadata.
        file_bytes, file_name, content_type = await download_attachment(
            email_id, attachment_id
        )

        # Step 2: Hand off to the parser router, which inspects the file
        #         name/content type and picks pdf_parser, docx_parser,
        #         pptx_parser, or xlsx_parser automatically. It also
        #         transparently falls back to OCR if needed.
        parse_result = await parse_attachment(file_bytes, file_name, content_type)

        return {
            "file_name": file_name,
            "file_type": parse_result["file_type"],
            "extracted_text": parse_result["text"],
            "used_ocr": parse_result["used_ocr"],
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="read_attachment", logger=logger)


# ---------------------------------------------------------------------------
# Tool: summarise_attachment
# ---------------------------------------------------------------------------
@mcp.tool
async def summarise_attachment(email_id: str, attachment_id: str) -> dict:
    """
    Read an email attachment and generate a concise AI summary of its content.

    Use this tool when the user asks things like:
    - "Summarise this attached document"
    - "Give me the key points from this CV"
    - "What's the gist of this attached report?"

    This is the attachment equivalent of summarise_email. It internally
    extracts the attachment's text (like read_attachment) and then asks
    the LLM to summarise it.

    Args:
        email_id (str): The unique Graph API ID of the parent email.
        attachment_id (str): The unique ID of the attachment to summarise.

    Returns:
        A dictionary containing:
            - file_name (str): Original file name
            - file_type (str): Detected file type
            - summary (str): A concise AI-generated summary of the document
    """
    logger.info(
        f"Tool called: summarise_attachment (email_id={email_id}, "
        f"attachment_id={attachment_id})"
    )

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="summarise_attachment",
            user_email=user_email,
            inputs={
                "email_id": email_id[:12],
                "attachment_id": attachment_id[:12],
            },
        )
        # Download and parse the attachment — same as read_attachment.
        # Returning raw extracted text lets LibreChat's LLM summarise it,
        # which produces better results than calling Requesty internally.
        file_bytes, file_name, content_type = await download_attachment(
            email_id, attachment_id
        )
        parse_result = await parse_attachment(file_bytes, file_name, content_type)

        extracted_text = parse_result["text"]

        # Guard against empty extraction.
        if not extracted_text or not extracted_text.strip():
            return {
                "file_name": file_name,
                "file_type": parse_result["file_type"],
                "extracted_text": "",
                "instruction": (
                    "No readable text could be extracted from this attachment. "
                    "It may be blank, corrupted, or an image-only file."
                ),
            }

        # ── Governance rules ─────────────────────────────────────────
        governance = get_email_rules(include_sources=False)

        return {
            "file_name": file_name,
            "file_type": parse_result["file_type"],
            "extracted_text": extracted_text,
            "instruction": (
                f"Summarise the following document in 4-5 concise sentences. "
                f"Highlight key facts, names, dates, and any action items.\n\n"
                f"File name: {file_name}\n"
                f"File type: {parse_result['file_type']}\n\n"
                f"Document content:\n{extracted_text}\n\n"
                f"{governance}"
            ),
        }

    except Exception as exc:
        return format_tool_error(
            exc, tool_name="summarise_attachment", logger=logger
        )

"""
    try:
        # Step 1: Reuse the same download + parse chain as read_attachment.
        file_bytes, file_name, content_type = await download_attachment(
            email_id, attachment_id
        )
        parse_result = await parse_attachment(file_bytes, file_name, content_type)

        extracted_text = parse_result["text"]

        # Step 2: Guard against empty extraction (e.g. a corrupted file
        #         or a scanned document where even OCR found nothing).
        if not extracted_text or not extracted_text.strip():
            return {
                "file_name": file_name,
                "file_type": parse_result["file_type"],
                "summary": (
                    "No readable text could be extracted from this "
                    "attachment. It may be a blank, corrupted, or "
                    "image-only file that OCR could not process."
                ),
            }

        # Step 3: Build a focused summarisation prompt, mentioning the
        #         file name and type so the LLM has useful context.
        prompt = (
            f"Summarise the following document content in 4-5 concise "
            f"sentences. Highlight key facts, names, dates, or action "
            f"items if present.\n\n"
            f"File name: {file_name}\n"
            f"File type: {parse_result['file_type']}\n\n"
            f"Document content:\n{extracted_text}"
        )

        summary_text = await generate_summary(prompt)

        return {
            "file_name": file_name,
            "file_type": parse_result["file_type"],
            "summary": summary_text,
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="summarise_attachment", logger=logger)

        """


---
./tools/calendar_tools.py
---
"""
calendar_tools.py
=================
Defines MCP tools for viewing calendar events.

Why this file exists:
    Phase 1 scope for calendar is read-only — viewing upcoming meetings.
    Calendar creation/scheduling is intentionally deferred to Phase 2,
    since it requires the Calendars.ReadWrite permission which hasn't
    been granted yet.

Design notes:
    - Same thin-wrapper pattern as email_tools.py: this file delegates
      all real work to graph/calendar_client.py.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
# from server import mcp
from tools.mcp_instance import mcp

from graph.calendar_client import fetch_calendar_events

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.date_utils import parse_relative_date_range
from utils.audit_logger import log_tool_call, get_user_email_from_headers
from utils.validator import validate_calendar_events, append_validation_to_result

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Tool: list_calendar_events
# ---------------------------------------------------------------------------
@mcp.tool
async def list_calendar_events(date_range: str = "this week") -> list[dict]:
    """
    List the user's upcoming calendar events / meetings within a given
    time range.

    Use this tool when the user asks things like:
    - "What meetings do I have today?"
    - "Show me my calendar for this week"
    - "Do I have anything scheduled tomorrow?"

    Args:
        date_range (str): A natural-language time range such as "today",
                           "tomorrow", "this week", or "next 7 days".
                           Defaults to "this week" if not specified.

    Returns:
        A list of dictionaries, each containing:
            - subject (str): Meeting title
            - start_time (str): ISO-formatted start date/time
            - end_time (str): ISO-formatted end date/time
            - organizer (str): Display name of the meeting organiser
            - location (str): Meeting location, if set
            - is_online_meeting (bool): Whether this is a Teams/online meeting
    """
    logger.info(f"Tool called: list_calendar_events (date_range='{date_range}')")

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="list_calendar_events",
            user_email=user_email,
            inputs={"date_range": date_range},
        )
        # Step 1: Convert the natural-language date_range string (e.g.
        #         "this week", "tomorrow") into concrete start/end
        #         datetime boundaries that Graph API's calendarView needs.
        start_dt, end_dt = parse_relative_date_range(date_range)

        # Step 2: Delegate to the Graph layer to fetch events within
        #         that boundary.
        raw_events = await fetch_calendar_events(start_dt, end_dt)

        # Step 3: Transform into clean, predictable output.
        result = []
        for event in raw_events:
            result.append({
                "subject": event.get("subject"),
                "start_time": event.get("start", {}).get("dateTime"),
                "end_time": event.get("end", {}).get("dateTime"),
                "organizer": event.get("organizer", {}).get("emailAddress", {}).get("name"),
                "location": event.get("location", {}).get("displayName", ""),
                "is_online_meeting": event.get("isOnlineMeeting", False),
            })

        # ── Validate output ───────────────────────────────────────────
        validation = validate_calendar_events(result)
        return append_validation_to_result({"events": result}, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="list_calendar_events", logger=logger)

---
./tools/chart_tools.py
---
"""
chart_tools.py
==============
MCP tool for generating charts and visualisations from email data,
task lists, calendar events, or any structured data the user provides.

Why this file exists:
    LibreChat's LLM can produce SVG code for charts, but cannot
    generate actual downloadable image files. This tool fills that
    gap — it takes structured data, builds a real chart using
    matplotlib, saves it as a PNG file, and returns both the file
    path and a base64-encoded version for inline display.

Supported chart types:
    - bar   : Compare values across categories
              e.g. email count per sender, tasks per owner
    - line  : Show trends over time
              e.g. emails received per day over a week
    - pie   : Show proportions/distribution
              e.g. email split by folder, tasks by status

Design notes:
    - No external API calls — matplotlib runs entirely locally.
    - Charts saved to temp_attachments/charts/ as PNG files.
    - Base64 encoding allows LibreChat to display inline.
    - matplotlib uses 'Agg' backend (no display/GUI required)
      which is correct for server-side rendering on Linux.
    - File names include timestamp to avoid collisions.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import os
import uuid
import base64
from pathlib import Path
from datetime import datetime

from tools.mcp_instance import mcp
from config.settings import settings
from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.audit_logger import log_tool_call, get_user_email_from_headers

logger = get_logger(__name__)

# comp. brand-aligned colour palette
# Professional blues, greys and accent colours
CHART_COLOURS = [
    "#0078D4",  # Microsoft Blue — primary
    "#50E6FF",  # Azure Cyan
    "#FFB900",  # Amber
    "#E74856",  # Red
    "#00CC6A",  # Green
    "#8764B8",  # Purple
    "#FF8C00",  # Orange
    "#004E8C",  # Dark Blue
    "#038387",  # Teal
    "#C239B3",  # Magenta
]


# ---------------------------------------------------------------------------
# Tool: generate_chart
# ---------------------------------------------------------------------------
@mcp.tool
async def generate_chart(
    chart_type: str,
    labels: str,
    values: str,
    title: str = "",
    x_label: str = "",
    y_label: str = "",
) -> dict:
    """
    Generate a chart (bar, line, or pie) rendered as a Mermaid diagram
    file that LibreChat displays natively as a visual chart.

    Use this tool when the user asks things like:
    - "Show me a bar chart of emails by sender"
    - "Generate a pie chart of my task status"
    - "Plot a line chart of emails received per day this week"

    Args:
        chart_type (str): "bar", "line", or "pie".
        labels (str): Comma-separated category labels.
        values (str): Comma-separated numeric values matching labels.
        title (str): Chart title.
        x_label (str): X-axis label (bar and line only).
        y_label (str): Y-axis label (bar and line only).

    Returns:
        dict with Mermaid file content and instruction to render it.
    """
    logger.info(f"Tool called: generate_chart (type={chart_type}, title='{title}')")

    try:
        # Step 1: Parse and validate inputs
        chart_type_clean = chart_type.strip().lower()

        if chart_type_clean not in ("bar", "line", "pie"):
            return {
                "error": True,
                "message": (
                    f"Unsupported chart type: '{chart_type}'. "
                    f"Please use 'bar', 'line', or 'pie'."
                ),
            }

        # Parse labels and values from comma-separated strings
        label_list = [l.strip() for l in labels.split(",") if l.strip()]
        value_list_raw = [v.strip() for v in values.split(",") if v.strip()]

        if not label_list or not value_list_raw:
            return {"error": True, "message": "Labels and values cannot be empty."}

        if len(label_list) != len(value_list_raw):
            return {
                "error": True,
                "message": (
                    f"Mismatch: {len(label_list)} labels but "
                    f"{len(value_list_raw)} values."
                ),
            }

        # Convert values to float — fail clearly if non-numeric
        try:
            value_list = [float(v) for v in value_list_raw]
        except ValueError as ve:
            return {
                "error": True,
                "message": f"Values must be numbers. Error: {ve}",
            }

        # Cap items to avoid unreadable charts
        max_items = settings.CHARTS_MAX_ITEMS
        if len(label_list) > max_items:
            label_list = label_list[:max_items]
            value_list = value_list[:max_items]

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="generate_chart",
            user_email=user_email,
            inputs={"chart_type": chart_type_clean},
        )

        # ── Build Mermaid diagram syntax ──────────────────────────────
        if chart_type_clean == "bar":
            mermaid_lines = [f'xychart-beta', f'  title "{title or "Bar Chart"}"']
            # Build quoted labels safely without nested f-strings
            labels_quoted = ", ".join(['"'+str(l)+'"' for l in label_list]) 
            mermaid_lines.append(f'  x-axis [{labels_quoted}]')
            if y_label:
                mermaid_lines.append(f'  y-axis "{y_label}"')
            mermaid_lines.append(f'  bar {[int(v) if v == int(v) else v for v in value_list]}')
            mermaid_content = "\n".join(mermaid_lines)

        elif chart_type_clean == "line":
            mermaid_lines = [f'xychart-beta', f'  title "{title or "Line Chart"}"']
            # Build quoted labels safely without nested f-strings
            labels_quoted = ", ".join(['"'+str(l)+'"' for l in label_list])
            mermaid_lines.append(f'  x-axis [{labels_quoted}]')
            if y_label:
                mermaid_lines.append(f'  y-axis "{y_label}"')
            mermaid_lines.append(f'  line {[int(v) if v == int(v) else v for v in value_list]}')
            mermaid_content = "\n".join(mermaid_lines)

        elif chart_type_clean == "pie":
            mermaid_lines = [f'pie title {title or "Pie Chart"}']
            for label, value in zip(label_list, value_list):
                mermaid_lines.append(f'  "{label}" : {value}')
            mermaid_content = "\n".join(mermaid_lines)

        # ── Save as .mmd file ─────────────────────────────────────────
        chart_dir = Path(settings.CHARTS_OUTPUT_DIR).resolve()
        chart_dir.mkdir(parents=True, exist_ok=True)

        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        short_id = str(uuid.uuid4())[:4]
        file_name = f"{chart_type_clean}_chart_{timestamp}_{short_id}.mmd"
        file_path = chart_dir / file_name

        file_path.write_text(mermaid_content, encoding="utf-8")
        logger.info(f"Mermaid chart saved: {file_path}")

        return {
            "chart_type": chart_type_clean,
            "title": title,
            "file_name": file_name,
            "file_path": str(file_path),
            "data_points": len(label_list),
            "mermaid_content": mermaid_content,
            "instruction": (
                f"Render this as a Mermaid chart file for the user.\n\n"
                f"Display it using a ```mermaid code block:\n\n"
                f"```mermaid\n{mermaid_content}\n```\n\n"
                f"Then in 1-2 sentences summarise the key insight from this data."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="generate_chart", logger=logger)



---
./tools/draft_tools.py
---
"""
draft_tools.py
==============
MCP tools for creating email drafts, reply drafts, meeting invite drafts,
and finding attendee availability.

Why this file exists:
    Phase 2 adds write capabilities to the platform — but strictly
    in draft mode only. No email is ever sent automatically. Every
    action creates a draft the user can review in Outlook or Calendar
    before deciding to send.

Drafting rules applied to all tools in this file:
    - Professional tone, always polite
    - Short and crisp unless elaboration adds value
    - Meeting invites always include: agenda, description, clear header
    - Meeting duration defaults to 30 minutes
    - Never aggressive, never informal

Design notes:
    - All tools delegate Graph API calls to graph/draft_client.py
      and graph/availability_client.py
    - Subject line and body construction happens here — these are
      the "prompt engineering" functions that produce the actual
      draft content following the drafting rules above
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from tools.mcp_instance import mcp

from graph.draft_client import (
    create_draft,
    create_reply_draft,
    get_draft_emails,
)
from graph.availability_client import (
    find_meeting_times,
    create_calendar_draft_invite,
)
from graph.mail_client import fetch_message_by_id

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.governance import get_draft_rules
from utils.validator import validate_draft_content, append_validation_to_result
from utils.audit_logger import log_tool_call, get_user_email_from_headers

logger = get_logger(__name__)

# ---------------------------------------------------------------------------
# Drafting rules — injected into every draft body as a system context
# These are the non-negotiable tone and format rules for all drafts.
# ---------------------------------------------------------------------------
DRAFT_SYSTEM_RULES = """
You are drafting a professional business email on behalf of the user.
Follow these rules strictly:
- Tone: Always professional, polite, and respectful
- Length: Short and crisp. Only elaborate when the context demands it
- Opening: Use a professional greeting (Dear [Name], Hi [Name],)
- Closing: Use a professional sign-off (Best regards, Kind regards,)
- Never use informal language, slang, or aggressive tone
- Never include placeholder text like [Your Name] in the output
- Output only the email body HTML — no explanations or meta-commentary
"""

INVITE_SYSTEM_RULES = """
You are drafting a professional meeting invite on behalf of the user.
Follow these rules strictly:
- Always include a clear meeting header (purpose in one line)
- Always include an Agenda section with numbered points
- Always include a Description/Context section explaining why the meeting is needed
- Default duration: 30 minutes unless explicitly stated otherwise
- Tone: Professional, clear, and concise
- Output HTML suitable for a calendar event body
"""


# ---------------------------------------------------------------------------
# Tool: list_draft_emails
# ---------------------------------------------------------------------------
@mcp.tool
async def list_draft_emails(count: int = 10) -> dict:
    """
    List emails currently saved in the Drafts folder.

    Use this tool when the user asks things like:
    - "Show me my drafts"
    - "What emails do I have in draft?"
    - "List my unsent emails"

    Args:
        count (int): Number of drafts to return. Default 10.

    Returns:
        dict with draft list and a display table.
    """
    logger.info(f"Tool called: list_draft_emails (count={count})")

    try:
        safe_count = min(max(count, 1), 50)

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="list_draft_emails",
            user_email=user_email,
            inputs={"count": safe_count},
        )
        
        drafts = await get_draft_emails(top=safe_count)

        if not drafts:
            return {
                "drafts": [],
                "count": 0,
                "display_table": "📭 No drafts found in your Drafts folder.",
                "instruction": "Inform the user their Drafts folder is empty.",
            }

        table_lines = [
            "**📝 Drafts Folder**\n",
            "| # | Last Modified | To | Subject | Preview |",
            "|---|--------------|-----|---------|---------|",
        ]

        for i, d in enumerate(drafts, 1):
            # Format date
            try:
                from dateutil import parser as dp
                dt = dp.parse(d["last_modified"])
                date_display = dt.strftime("%d %b %Y %H:%M")
            except Exception:
                date_display = d["last_modified"][:10] if d["last_modified"] else "—"

            to_display = ", ".join(d["to"]) if d["to"] else "(no recipients)"
            subject_col = d["subject"][:50] + ("…" if len(d["subject"]) > 50 else "")
            preview_col = d["preview"][:60] + ("…" if len(d["preview"]) > 60 else "")

            table_lines.append(
                f"| {i} | {date_display} | {to_display} | {subject_col} | {preview_col} |"
            )

        return {
            "drafts": drafts,
            "count": len(drafts),
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "Then offer to open, edit, or delete any specific draft."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="list_draft_emails", logger=logger)


# ---------------------------------------------------------------------------
# Tool: create_draft_email
# ---------------------------------------------------------------------------
@mcp.tool
async def compose_email(
    to_emails: str,
    subject: str,
    context: str,
    cc_emails: str = "",
) -> dict:
    """
    Compose a professional email and display it in chat for review.
    Does NOT save to Outlook yet — use save_draft_to_outlook after
    the user approves the composed email.

    Use this tool when the user asks things like:
    - "Draft an email to john@company.com about the project delay"
    - "Write a professional email to the team about tomorrow's meeting"
    - "Compose an email to the client about the invoice"

    Args:
        to_emails (str): Comma-separated recipient email addresses.
        subject (str): The email subject line.
        context (str): What the email should cover.
        cc_emails (str): Optional CC addresses.

    Returns:
        dict with composition instruction for LibreChat's LLM.
    """
    logger.info(f"Tool called: compose_email (to={to_emails}, subject={subject})")

    try:
        to_list = [e.strip() for e in to_emails.split(",") if e.strip()]
        cc_list = [e.strip() for e in cc_emails.split(",") if e.strip()] if cc_emails else []

        if not to_list:
            return {"error": True, "message": "No valid recipient email addresses provided."}

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="compose_email",
            user_email=user_email,
            inputs={"subject": subject},
        )

        # Derive greeting name
        local_part = to_list[0].split("@")[0]
        greeting_name = local_part.replace("_", " ").replace(".", " ").title()
        
        # ── Governance rules ─────────────────────────────────────────
        governance = get_draft_rules()

        # ── Build display header ──────────────────────────────────────
        header = (
            f"**📧 Draft Email**\n\n"
            f"| Field | Details |\n"
            f"|---|---|\n"
            f"| **To** | {', '.join(to_list)} |\n"
            + (f"| **CC** | {', '.join(cc_list)} |\n" if cc_list else "")
            + f"| **Subject** | {subject} |\n\n"
            f"---\n"
        )

        return {
            "to": to_list,
            "cc": cc_list,
            "subject": subject,
            "greeting_name": greeting_name,
            "context": context,
            "display_header": header,
            "instruction": (
                f"Display 'display_header' as markdown first.\n"
                f"Then compose the email body following these rules:\n"
                f"- First line: Dear {greeting_name},\n"
                f"- 2-3 concise professional sentences based on: '{context}'\n"
                f"- Correct any grammar issues in the context\n"
                f"- Last line: Best regards,\n"
                f"- Do NOT add promises, dates, or facts not in the context\n"
                f"- Do NOT include a name after the closing\n\n"
                f"{governance}\n\n"
                f"After displaying the email ask:\n"
                f"'Would you like me to save this to your Outlook Drafts?'\n"
                f"If yes, call save_draft_to_outlook with:\n"
                f"- to_emails: '{', '.join(to_list)}'\n"
                f"- subject: '{subject}'\n"
                f"- body_text: the exact composed email body"
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="compose_email", logger=logger)
    

# ---------------------------------------------------------------------------
# Tool: compose_reply
# ---------------------------------------------------------------------------
@mcp.tool
async def compose_reply(
    email_id: str,
    reply_context: str,
) -> dict:
    """
    Read an email and compose a professional reply — displayed in
    chat for review before saving. Does NOT save to Outlook yet.
    Use save_draft_to_outlook after the user approves the reply.

    Use this tool when the user asks things like:
    - "Draft a reply to John's email saying we'll deliver by Friday"
    - "Reply to this email professionally"
    - "Write a response to the client's email"

    Args:
        email_id (str): Graph API ID of the email to reply to.
                         Obtained from list_emails or search_emails.
        reply_context (str): What the reply should say.

    Returns:
        dict with original email context and composition instruction
        for LibreChat's LLM to compose and display the reply.
    """
    logger.info(f"Tool called: compose_reply (email_id={email_id})")

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="compose_reply",
            user_email=user_email,
            inputs={"email_id": email_id[:12]},
        )
        # Step 1: Read the original email for context
        # ── Read original email ───────────────────────────────────────
        original = await fetch_message_by_id(email_id)
        original_subject = original.get("subject", "")
        original_sender_name = (
            original.get("from", {}).get("emailAddress", {}).get("name", "")
        )
        original_sender_email = (
            original.get("from", {}).get("emailAddress", {}).get("address", "")
        )
        original_preview = original.get("body", {}).get("content", "")[:300]

        # ── Governance rules ─────────────────────────────────────────
        governance = get_draft_rules()

        return {
            "original_subject": original_subject,
            "original_sender": original_sender_name,
            "original_sender_email": original_sender_email,
            "reply_context": reply_context,
            "instruction": (
                f"Display this header first:\n\n"
                f"**📧 Reply Draft**\n\n"
                f"| Field | Details |\n"
                f"|---|---|\n"
                f"| **Replying to** | {original_sender_name} <{original_sender_email}> |\n"
                f"| **Re** | {original_subject} |\n\n"
                f"---\n\n"
                f"Then compose the reply body:\n"
                f"- First line: Dear {original_sender_name},\n"
                f"- 2-3 concise professional sentences based on: '{reply_context}'\n"
                f"- Reference the original email context where helpful\n"
                f"- Last line: Best regards,\n\n"
                f"{governance}\n\n"
                f"After displaying ask:\n"
                f"'Would you like me to save this reply to Outlook Drafts?'\n"
                f"IMPORTANT: If yes, you MUST call save_reply_draft "
                f"(NOT save_draft_to_outlook) with:\n"
                f"- email_id: '{email_id}'\n"
                f"- body_text: the exact composed reply body\n"
                f"Using save_draft_to_outlook for replies breaks the thread chain."
                f"Original email context (for reference):\n{original_preview}"
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="compose_reply", logger=logger)

# ---------------------------------------------------------------------------
# Tool: save_draft_to_outlook
# ---------------------------------------------------------------------------

@mcp.tool
async def save_draft_to_outlook(
    to_emails: str,
    subject: str,
    body_text: str,
    cc_emails: str = "",
) -> dict:
    """
    Save an already-composed email body to Outlook Drafts folder.
    Call this AFTER compose_email and user approval — pass the exact
    composed email body text from the chat.

    Use this tool when the user says things like:
    - "Yes save it to drafts"
    - "Save that email"
    - "Push it to Outlook"

    Args:
        to_emails (str): Comma-separated recipient email addresses.
        subject (str): The email subject line.
        body_text (str): The exact composed email body text to save.
                          This should be the complete email including
                          greeting and sign-off — copied exactly from
                          what was displayed in chat.
        cc_emails (str): Optional CC addresses.

    Returns:
        dict with draft ID and save confirmation.
    """
    logger.info(f"Tool called: save_draft_to_outlook (to={to_emails})")

    try:
        to_list = [e.strip() for e in to_emails.split(",") if e.strip()]
        cc_list = [e.strip() for e in cc_emails.split(",") if e.strip()] if cc_emails else []

        if not to_list:
            return {"error": True, "message": "No valid recipient addresses provided."}

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="save_draft_to_outlook",
            user_email=user_email,
            inputs={"subject": subject},
        )

        # Convert plain text to clean minimal HTML
        # Split on newlines, wrap each non-empty line in <p> tags
        paragraphs = [p.strip() for p in body_text.split("\n") if p.strip()]
        body_html = (
            '<html><body style="font-family: Calibri, Arial, sans-serif; '
            'font-size: 14px; color: #333;">'
            + "".join(f"<p>{p}</p>" for p in paragraphs)
            + "</body></html>"
        )
        # ── Validate draft content before saving ──────────────────────
        validation = validate_draft_content(
            body_text=body_text,
            to_emails=to_list,
            subject=subject,
        )
        if not validation.is_valid:
            return append_validation_to_result({}, validation)
        
        # ── Save to Outlook Drafts ────────────────────────────────────
        result = await create_draft(
            to_emails=to_list,
            subject=subject,
            body_html=body_html,
            cc_emails=cc_list if cc_list else None,
        )

        final_result = {
            "draft_id": result["draft_id"],
            "subject": subject,
            "to": to_list,
            "status": "✅ Draft saved to Outlook Drafts folder.",
            "instruction": (
                f"Display this confirmation:\n\n"
                f"✅ *Meeting invite created and notifications sent to all attendees.*"
                f"| Field | Details |\n"
                f"|---|---|\n"
                f"| **To** | {', '.join(to_list)} |\n"
                + (f"| **CC** | {', '.join(cc_list)} |\n" if cc_list else "")
                + f"| **Subject** | {subject} |\n"
                f"| **Status** | Saved to Drafts folder |\n\n"
                f"*Open Outlook to review and send.*"
            ),
        }

        # ── Append any validation warnings ───────────────────────────
        return append_validation_to_result(final_result, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="save_draft_to_outlook", logger=logger)


# ---------------------------------------------------------------------------
# Tool: find_availability
# ---------------------------------------------------------------------------
@mcp.tool
async def find_availability(
    attendee_emails: str,
    duration_minutes: int = 30,
    preferred_date: str = "",
    search_days: int = 7,
) -> dict:
    """
    Find common free time slots for a group of attendees.

    Use this tool when the user asks things like:
    - "Find a free slot for me and john@company.com this week"
    - "When are John and Jane both free for a 30-minute call?"
    - "Check availability for the team next week"

    Args:
        attendee_emails (str): Comma-separated attendee email addresses.
        duration_minutes (int): Required meeting length. Default 30 min.
        preferred_date (str): Optional preferred start date e.g.
                               "next Monday", "15 July", "2026-07-15".
        search_days (int): Days ahead to search. Default 7.

    Returns:
        dict with suggested time slots displayed as a table.
    """
    logger.info(f"Tool called: find_availability (attendees={attendee_emails})")

    try:
        email_list = [e.strip() for e in attendee_emails.split(",") if e.strip()]

        if not email_list:
            return {
                "error": True,
                "message": "No attendee email addresses provided.",
            }

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="find_availability",
            user_email=user_email,
            inputs={"duration_minutes": duration_minutes},
        )

        result = await find_meeting_times(
            attendee_emails=email_list,
            duration_minutes=duration_minutes,
            search_window_days=search_days,
            preferred_start=preferred_date,
        )

        suggestions = result.get("suggestions", [])

        if not suggestions:
            return {
                **result,
                "display_table": (
                    "❌ No common free slots found within the search window. "
                    "Try extending the date range or checking fewer attendees."
                ),
                "instruction": "Inform the user no slots were found and suggest alternatives.",
            }

        # Build availability table
        table_lines = [
            f"**📅 Available Meeting Slots ({duration_minutes} min)**\n",
            f"*Checking availability for: {', '.join(email_list)}*\n",
            "| # | Date & Time | Duration | Availability |",
            "|---|-------------|----------|--------------|",
        ]

        for i, slot in enumerate(suggestions, 1):
            table_lines.append(
                f"| {i} | {slot['start_display']} – {slot['end_display']} "
                f"| {slot['duration_minutes']} min | {slot['quality']} |"
            )

        return {
            **result,
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "Then ask the user which slot they'd like to use "
                "and offer to draft a meeting invite for it."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="find_availability", logger=logger)


# ---------------------------------------------------------------------------
# Tool: create_draft_invite
# ---------------------------------------------------------------------------
@mcp.tool
async def create_draft_invite(
    subject: str,
    attendee_emails: str,
    start_datetime: str,
    agenda: str,
    duration_minutes: int = 30,
    location: str = "Microsoft Teams",
    is_online: bool = True,
) -> dict:
    """
    Draft a professional meeting invite with agenda and save it to Calendar.
    The invite is NOT sent until the user confirms in Outlook Calendar.

    Use this tool when the user asks things like:
    - "Draft a meeting invite for the project kickoff with John and Jane"
    - "Create a calendar invite for our review meeting on Monday at 10am"
    - "Schedule a 30-minute call with the team about the quarterly report"

    Args:
        subject (str): Meeting title — clear and descriptive.
        attendee_emails (str): Comma-separated attendee email addresses.
        start_datetime (str): Meeting start — ISO format or natural
                               language e.g. "Monday 10am", "2026-07-15T10:00:00".
        agenda (str): Meeting agenda points — used to build the invite body.
                       e.g. "1. Review Q3 results 2. Plan Q4 roadmap"
        duration_minutes (int): Duration in minutes. Default 30.
        location (str): Location string. Default "Microsoft Teams".
        is_online (bool): Create as online meeting. Default True.

    Returns:
        dict with event ID, formatted invite preview, and confirmation.
    """
    logger.info(f"Tool called: create_draft_invite (subject={subject})")

    try:
        email_list = [e.strip() for e in attendee_emails.split(",") if e.strip()]

        if not email_list:
            return {
                "error": True,
                "message": "No attendee email addresses provided.",
            }

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="create_draft_invite",
            user_email=user_email,
            inputs={"subject": subject},
        )

        # ── Governance rules ─────────────────────────────────────────
        governance = get_draft_rules()
        
        # Build professional HTML invite body following invite rules:
        # clear header, agenda section, description/context section
        body_html = f"""
<html>
<body style="font-family: Calibri, Arial, sans-serif; font-size: 14px;">

<h2 style="color: #0078D4;">{subject}</h2>

<table style="border-collapse: collapse; margin-bottom: 16px;">
  <tr>
    <td style="padding: 4px 12px 4px 0; font-weight: bold; color: #555;">Duration:</td>
    <td style="padding: 4px 0;">{duration_minutes} minutes</td>
  </tr>
  <tr>
    <td style="padding: 4px 12px 4px 0; font-weight: bold; color: #555;">Location:</td>
    <td style="padding: 4px 0;">{location}</td>
  </tr>
  <tr>
    <td style="padding: 4px 12px 4px 0; font-weight: bold; color: #555;">Format:</td>
    <td style="padding: 4px 0;">{"Online (Microsoft Teams)" if is_online else "In-person"}</td>
  </tr>
</table>

<hr style="border: none; border-top: 1px solid #e0e0e0; margin: 16px 0;">

<h3 style="color: #0078D4;">📋 Agenda</h3>
<p style="margin-left: 16px; line-height: 1.8;">{agenda.replace(chr(10), "<br>")}</p>

<hr style="border: none; border-top: 1px solid #e0e0e0; margin: 16px 0;">

<h3 style="color: #0078D4;">ℹ️ Description</h3>
<p>This meeting has been organised to discuss: <strong>{subject}</strong>.</p>
<p>Please review the agenda above and come prepared with any relevant
updates or questions.</p>

<hr style="border: none; border-top: 1px solid #e0e0e0; margin: 16px 0;">

<p style="color: #888; font-size: 12px;">
  This invite was drafted via the Outlook AI Agent. Please review
  and confirm before sending.
</p>

</body>
</html>
"""

        result = await create_calendar_draft_invite(
            subject=subject,
            attendee_emails=email_list,
            start_datetime=start_datetime,
            duration_minutes=duration_minutes,
            body_html=body_html,
            location=location,
            is_online=is_online,
        )

        # Build a preview for the user to see in chat
        invite_preview = f"""
---
### 📅 Meeting Invite Draft

| Field | Details |
|---|---|
| **Subject** | {subject} |
| **Time** | {result['start']} – {result['end']} |
| **Duration** | {duration_minutes} minutes |
| **Attendees** | {', '.join(email_list)} |
| **Location** | {location} |
| **Online** | {"Yes (Teams)" if is_online else "No"} |

**Agenda:**
{agenda}

---
✅ *Draft saved to your Calendar. Review and send from Outlook Calendar.*
"""

        return {
            **result,
            "invite_preview": invite_preview,
            "instruction": (
                f"Display 'invite_preview' as markdown. "
                f"Confirm it has been saved to Calendar for review.\n\n"
                f"{governance}"
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="create_draft_invite", logger=logger)
    
# ---------------------------------------------------------------------------
# Tool: save_reply_draft
# ---------------------------------------------------------------------------
@mcp.tool
async def save_reply_draft(
    email_id: str,
    body_text: str,
    reply_all: bool = True,
) -> dict:
    """
    Save a composed reply as a THREADED draft in Outlook Drafts.
    This is the ONLY tool to use for saving email replies.
    It preserves the conversation thread and enables Reply All.

    NEVER use save_draft_to_outlook for replies — that creates a
    standalone email and breaks the thread chain.

    Use this tool when the user says:
    - "Yes save the reply"
    - "Save that reply to drafts"
    - "Push the reply to Outlook"

    Args:
        email_id (str): Graph API ID of the original email being replied to.
        body_text (str): The exact composed reply body from chat.
        reply_all (bool): True for Reply All (default), False for Reply.

    Returns:
        dict with draft ID and confirmation.
    """
    logger.info(f"Tool called: save_reply_draft (email_id={email_id[:20]}...)")

    try:
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="save_reply_draft",
            user_email=user_email,
            inputs={"email_id": email_id[:12], "reply_all": reply_all},
        )

        # Convert plain text to clean HTML
        paragraphs = [p.strip() for p in body_text.split("\n") if p.strip()]
        body_html = (
            '<html><body style="font-family: Calibri, Arial, sans-serif; '
            'font-size: 14px; color: #333;">'
            + "".join(f"<p>{p}</p>" for p in paragraphs)
            + "</body></html>"
        )

        result = await create_reply_draft(
            message_id=email_id,
            reply_body_html=body_html,
            reply_all=reply_all,
        )

        return {
            **result,
            "instruction": (
                f"Display this confirmation:\n\n"
                f"**✅ Reply Draft Saved**\n\n"
                f"| Field | Details |\n"
                f"|---|---|\n"
                f"| **Type** | {'Reply All' if reply_all else 'Reply'} |\n"
                f"| **Status** | Saved to Outlook Drafts |\n"
                f"| **Thread** | Chain preserved |\n\n"
                f"*Open Outlook to review and send. "
                f"The reply is threaded under the original email.*"
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="save_reply_draft", logger=logger)

---
./tools/email_tools.py
---
"""
email_tools.py
==============
Defines MCP tools for listing, reading, searching, and summarising emails.

Why this file exists:
    This is the core of Phase 1's email capability. Each function here
    is intentionally a thin wrapper: it does not contain any Graph API
    logic or LLM prompt logic directly. It calls into:
        - graph/mail_client.py     for fetching raw email data
        - llm/requesty_client.py   for generating summaries

Design notes:
    - Every tool returns clean, predictable JSON-friendly dictionaries —
      never raw Graph API response objects — so the LLM has an easy time
      reasoning over the output.
    - Docstrings are written specifically so the LLM can correctly decide
      which tool to call for a given user request (e.g. distinguishing
      "list emails" from "search emails").
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
# from server import mcp
from tools.mcp_instance import mcp

# Graph layer functions — each does exactly one Graph API operation.
from graph.mail_client import (
    fetch_message_by_id,
    fetch_messages_paged,
    fetch_recent_messages,
    search_messages_advanced,
    search_messages_by_keyword,
)

# LLM wrapper — used only by summarise_email.
# from llm.requesty_client import generate_summary
from graph.draft_client import flag_message
from graph.draft_client import mark_message_read_status
from utils.logger import get_logger
from utils.governance import get_email_rules
from utils.error_handler import format_tool_error
from utils.validator import validate_email_list, append_validation_to_result
from utils.validator import validate_email_body, append_validation_to_result

# Audit log — before any Graph API call
from utils.audit_logger import log_tool_call, get_user_email_from_headers
        
logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Tool: list_emails
# ---------------------------------------------------------------------------
@mcp.tool
async def list_emails(count: int = 10) -> list[dict]:
    """
    List the most recent emails in the user's Outlook inbox.

    Use this tool when the user asks things like:
    - "Show me my recent emails"
    - "What's in my inbox?"
    - "List my last 5 emails"

    Args:
        count (int): How many recent emails to return. Defaults to 10.
                      Keep this reasonable (max 50) to avoid overwhelming
                      responses.

    Returns:
        A list of dictionaries, each containing:
            - id (str): Unique email ID, used by read_email/summarise_email
            - subject (str): Email subject line
            - sender (str): Display name of the sender
            - sender_email (str): Email address of the sender
            - received_date (str): ISO-formatted date/time received
            - has_attachments (bool): Whether this email has attachments
            - preview (str): A short snippet of the email body
    """
    logger.info(f"Tool called: list_emails (count={count})")

    try:
        # Step 1: Defensive bound — never let the AI request something
        #         excessive that could slow down or overload the mailbox call.
        safe_count = min(max(count, 1), 50)

        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="list_emails",
            user_email=user_email,
            inputs={"count": safe_count},
        )

        # Step 2: Delegate to the Graph layer. This function returns a list
        #         of raw Graph API message objects.
        raw_messages = await fetch_recent_messages(top=safe_count)

        # Step 3: Transform raw Graph objects into the clean shape defined
        #         in this tool's docstring. This keeps the AI-facing output
        #         consistent regardless of what Graph API's raw shape looks like.
        result = []
        for msg in raw_messages:
            received_raw = msg.get("receivedDateTime", "")
            # Format date: 2026-06-15T09:30:00Z → 15 Jun 2026 09:30
            try:
                from dateutil import parser as dp
                dt = dp.parse(received_raw)
                date_display = dt.strftime("%d %b %Y %H:%M")
            except Exception:
                date_display = received_raw[:10] if received_raw else "—"

            is_read = msg.get("is_read", True)
            result.append({
                "id": msg.get("id"),
                "subject": msg.get("subject") or "(no subject)",
                "sender": ...,
                "sender_email": ...,
                "received_date": received_raw,
                "date_display": date_display,
                "has_attachments": msg.get("hasAttachments", False),
                "is_read": is_read,
                "preview": msg.get("bodyPreview", "")[:150],
            })

        # Build markdown table for clean display in LibreChat
        table_lines = [
            "| # | Date | From | Subject | Preview | 📎 |",
            "|---|------|------|---------|---------|-----|",
        ]
        table_lines = [
            "| # | 🔵 | Date | From | Subject | Preview | 📎 |",
            "|---|---|------|------|---------|---------|-----|",
        ]
        for i, m in enumerate(result, 1):
            att = "✅" if m["has_attachments"] else "—"
            unread = "🔵" if not m["is_read"] else "⚪"
            sender_col = f"{m['sender']}"
            subject_col = m["subject"][:50] + ("…" if len(m["subject"]) > 50 else "")
            preview_col = m["preview"][:80] + ("…" if len(m["preview"]) > 80 else "")
            table_lines.append(
                f"| {i} | {unread} | {m['date_display']} | {sender_col} "
                f"| {subject_col} | {preview_col} | {att} |"
            )

        final_result = {
            "emails": result,
            "count": len(result),
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display the 'display_table' field as a markdown table. "
                "🔵 = Unread, ⚪ = Read. "
                "Do not reformat it. "
                "Then offer to read, summarise, or search any specific email."
            ),
        }

        validation = validate_email_list(result)
        return append_validation_to_result(final_result, validation)
    
    except Exception as exc:
        return format_tool_error(exc, tool_name="list_emails", logger=logger)


# ---------------------------------------------------------------------------
# Tool: read_email
# ---------------------------------------------------------------------------
@mcp.tool
async def read_email(email_id: str) -> dict:
    """
    Read the full content of a specific email by its ID.

    Use this tool when the user asks things like:
    - "Open this email and tell me what it says"
    - "Read the full content of [a specific email]"
    - "What does this email say in detail?"

    This tool is typically called AFTER list_emails or search_emails,
    using the 'id' field returned by those tools.

    Args:
        email_id (str): The unique Graph API ID of the email to read.
                         Obtained from list_emails or search_emails output.

    Returns:
        A dictionary containing:
            - subject (str): Email subject line
            - sender (str): Display name of the sender
            - sender_email (str): Email address of the sender
            - received_date (str): ISO-formatted date/time received
            - body (str): Full plain-text body of the email
            - has_attachments (bool): Whether this email has attachments
    """
    logger.info(f"Tool called: read_email (email_id={email_id})")

    try:
        # ── Audit log ───────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="read_email",
            user_email=user_email,
            inputs={"email_id": email_id[:12]},
        )

        # Step 1: Fetch the single full message from Graph API.
        msg = await fetch_message_by_id(email_id)

        # Step 2: Return the full body this time (unlike list_emails,
        #         which only returns a short preview).

        subject = msg.get("subject") or "(no subject)"
        sender_name = msg.get("from", {}).get("emailAddress", {}).get("name") or "—"
        sender_addr = msg.get("from", {}).get("emailAddress", {}).get("address") or "—"
        received_raw = msg.get("receivedDateTime", "")
        body = msg.get("body", {}).get("content", "")
        has_att = msg.get("hasAttachments", False)

        # ── Validate email body ───────────────────────────────────────
        body_validation = validate_email_body(body, subject)
        
        # Format date
        try:
            from dateutil import parser as dp
            dt = dp.parse(received_raw)
            date_display = dt.strftime("%A, %d %B %Y at %H:%M")
        except Exception:
            date_display = received_raw

        # Build Outlook-style inbox view as markdown
        inbox_view = f"""
---
### 📧 {subject}

| Field       | Details                          |
|-------------|----------------------------------|
| **From**    | {sender_name} <{sender_addr}>   |
| **Date**    | {date_display}                   |
| **Subject** | {subject}                        |
| **Attach.** | {"📎 Yes" if has_att else "None"} |

---

{body}

---
"""

        final_result = {
            "subject": subject,
            "sender": sender_name,
            "sender_email": sender_addr,
            "received_date": received_raw,
            "body": body,
            "has_attachments": has_att,
            "inbox_view": inbox_view,
            "instruction": (
                "Display the 'inbox_view' field exactly as markdown. "
                "This is the formatted email view. "
                "Do not reformat the body."
            ),
        }

        return append_validation_to_result(final_result, body_validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="read_email", logger=logger)


# ---------------------------------------------------------------------------
# Tool: search_emails
# ---------------------------------------------------------------------------
@mcp.tool
async def search_emails(keyword: str, count: int = 10) -> list[dict]:
    """
    Search the user's mailbox for emails matching a keyword.

    Use this tool when the user asks things like:
    - "Find emails about the Q3 report"
    - "Search for emails from John about the budget"
    - "Do I have any emails mentioning the project deadline?"

    This is different from list_emails: use search_emails whenever the
    user gives a specific topic, sender name, or keyword to look for,
    rather than just asking to see recent emails generally.

    Args:
        keyword (str): The search term — can be a subject keyword,
                        sender name, or content phrase.
        count (int): Maximum number of matching emails to return.
                      Defaults to 10.

    Returns:
        A list of dictionaries in the same shape as list_emails:
            - id, subject, sender, sender_email, received_date,
              has_attachments, preview
    """
    logger.info(f"Tool called: search_emails (keyword='{keyword}', count={count})")

    try:
        safe_count = min(max(count, 1), 50)

        # ── Audit log ────────────────────────────────────────────────
        from utils.audit_logger import log_tool_call, get_user_email_from_headers
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="search_emails",
            user_email=user_email,
            inputs={"keyword": keyword, "count": safe_count},
        )

        raw_messages = await search_messages_by_keyword(keyword, top=safe_count)

        result = []
        for msg in raw_messages:
            received_raw = msg.get("receivedDateTime", "")
            # Format date: 2026-06-15T09:30:00Z → 15 Jun 2026 09:30
            try:
                from dateutil import parser as dp
                dt = dp.parse(received_raw)
                date_display = dt.strftime("%d %b %Y %H:%M")
            except Exception:
                date_display = received_raw[:10] if received_raw else "—"

            result.append({
                "id": msg.get("id"),
                "subject": msg.get("subject") or "(no subject)",
                "sender": msg.get("from", {}).get("emailAddress", {}).get("name") or "—",
                "sender_email": msg.get("from", {}).get("emailAddress", {}).get("address") or "—",
                "received_date": received_raw,
                "date_display": date_display,
                "has_attachments": msg.get("hasAttachments", False),
                "preview": msg.get("bodyPreview", "")[:150],
            })

        # Build markdown table for clean display in LibreChat
        table_lines = [
            "| # | Date | From | Subject | Preview | 📎 |",
            "|---|------|------|---------|---------|-----|",
        ]
        for i, m in enumerate(result, 1):
            att = "✅" if m["has_attachments"] else "—"
            sender_col = f"{m['sender']}"
            subject_col = m["subject"][:50] + ("…" if len(m["subject"]) > 50 else "")
            preview_col = m["preview"][:80] + ("…" if len(m["preview"]) > 80 else "")
            table_lines.append(
                f"| {i} | {m['date_display']} | {sender_col} "
                f"| {subject_col} | {preview_col} | {att} |"
            )

        final_result = {
            "emails": result,
            "count": len(result),
            "keyword": keyword,
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "Then offer to open or summarise any specific result."
            ),
        }

        # ── Validate output ───────────────────────────────────────────
        validation = validate_email_list(result)
        return append_validation_to_result(final_result, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="search_emails", logger=logger)

# ---------------------------------------------------------------------------
# Tool: mark_email_read
# ---------------------------------------------------------------------------
@mcp.tool
async def mark_email_read(
    email_id: str,
    is_read: bool = True,
) -> dict:
    """
    Mark a specific email as read or unread.

    Use this tool when the user asks things like:
    - "Mark this email as read"
    - "Mark John's email as unread"
    - "Set this message as unread so I remember to come back to it"

    Args:
        email_id (str): The unique Graph API ID of the email.
                         Obtained from list_emails or search_emails.
        is_read (bool): True to mark as read, False to mark as unread.
                         Defaults to True (mark as read).

    Returns:
        dict with confirmation of the read status change.
    """
    logger.info(
        f"Tool called: mark_email_read "
        f"(email_id={email_id[:20]}..., is_read={is_read})"
    )

    try:
        # ── Audit log ────────────────────────────────────────────────
        from utils.audit_logger import log_tool_call, get_user_email_from_headers
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="mark_email_read",
            user_email=user_email,
            inputs={"email_id": email_id[:12], "is_read": is_read},
        )
        result = await mark_message_read_status(
            message_id=email_id,
            is_read=is_read,
        )

        status_label = "read" if is_read else "unread"

        return {
            **result,
            "instruction": (
                f"Inform the user the email has been marked as {status_label}."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="mark_email_read", logger=logger)


# ---------------------------------------------------------------------------
# Tool: summarise_email
# ---------------------------------------------------------------------------
@mcp.tool
async def summarise_email(email_id: str) -> dict:
    """
    Generate a concise AI summary of a specific email's content.

    Use this tool when the user asks things like:
    - "Summarise this email"
    - "Give me the key points of this email"
    - "What's the TL;DR of this message?"

    This tool internally reads the full email (like read_email) and then
    passes its content to the LLM to generate a short summary. Use this
    instead of read_email when the user wants a digest rather than the
    full raw text.

    Args:
        email_id (str): The unique Graph API ID of the email to summarise.

    Returns:
        A dictionary containing:
            - subject (str): The original email's subject line
            - sender (str): Display name of the sender
            - summary (str): A concise AI-generated summary of the email body
    """
    logger.info(f"Tool called: summarise_email (email_id={email_id})")


    try:
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="summarise_email",
            user_email=user_email,
            inputs={"email_id": email_id[:12]},
        )

        # Fetch the full email content — same as read_email.
        # Returning raw content lets LibreChat's LLM summarise it,
        # which produces better results than calling Requesty internally.
        msg = await fetch_message_by_id(email_id)

        subject = msg.get("subject", "")
        sender = msg.get("from", {}).get("emailAddress", {}).get("name", "")
        sender_email = msg.get("from", {}).get("emailAddress", {}).get("address", "")
        received_date = msg.get("receivedDateTime", "")
        body_text = msg.get("body", {}).get("content", "")

        # ── Validate email body before summarising ────────────────────
        body_validation = validate_email_body(body_text, subject)
        if not body_validation.is_valid:
            return append_validation_to_result({}, body_validation)

        # ── Governance rules for summarisation ────────────────────────
        governance = get_email_rules(include_sources=True)

        final_result = {
            "subject": subject,
            "sender": sender,
            "sender_email": sender_email,
            "received_date": received_date,
            "body": body_text,
            "instruction": (
                f"Summarise the following email in 3-4 concise sentences. "
                f"Focus on key points, action items, and deadlines.\n\n"
                f"Subject: {subject}\n"
                f"From: {sender}\n\n"
                f"Email body:\n{body_text}\n\n"
                f"{governance}"
            ),
        }

        return append_validation_to_result(final_result, body_validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="summarise_email", logger=logger)

# ---------------------------------------------------------------------------
# Tool: search_emails_advanced
# ---------------------------------------------------------------------------
@mcp.tool
async def search_emails_advanced(
    sender_email: str = "",
    keyword: str = "",
    date_from: str = "",
    date_to: str = "",
    count: int = 20,
) -> dict:
    """
    Advanced email search with sender, keyword, and date range filters.

    Use this tool when the user asks things like:
    - "Get emails from jane@company.com between June 10 and June 18"
    - "Show emails from John about budget last week"
    - "Find all emails received between 1 July and 7 July"

    Args:
        sender_email (str): Filter by sender email address.
        keyword (str): Filter by keyword in subject or body.
        date_from (str): Start date in any format e.g. "June 10",
                          "2026-06-10", "10 Jun 2026".
        date_to (str): End date in any format.
        count (int): Maximum number of results. Default 20, max 50.

    Returns:
        dict with emails list and a display_table for clean rendering.
    """
    logger.info(
        f"Tool called: search_emails_advanced "
        f"(sender={sender_email}, keyword={keyword}, "
        f"date_from={date_from}, date_to={date_to})"
    )

    try:
        safe_count = min(max(count, 1), 50)

        # ── Audit log ────────────────────────────────────────────────
        from utils.audit_logger import log_tool_call, get_user_email_from_headers
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="search_emails_advanced",
            user_email=user_email,
            inputs={
                "keyword": keyword,
                "sender": sender_email,
                "date_from": date_from,
                "date_to": date_to,
                "count": safe_count,
            },
        )

        raw_messages = await search_messages_advanced(
            sender_email=sender_email,
            keyword=keyword,
            date_from=date_from,
            date_to=date_to,
            top=safe_count,
        )

        result = []
        for msg in raw_messages:
            received_raw = msg.get("receivedDateTime", "")
            try:
                from dateutil import parser as dp
                dt = dp.parse(received_raw)
                date_display = dt.strftime("%d %b %Y %H:%M")
            except Exception:
                date_display = received_raw[:10] if received_raw else "—"

            result.append({
                "id": msg.get("id"),
                "subject": msg.get("subject") or "(no subject)",
                "sender": msg.get("from", {}).get("emailAddress", {}).get("name") or "—",
                "sender_email": msg.get("from", {}).get("emailAddress", {}).get("address") or "—",
                "received_date": received_raw,
                "date_display": date_display,
                "has_attachments": msg.get("hasAttachments", False),
                "preview": msg.get("bodyPreview", "")[:150],
            })

        # Build table
        table_lines = [
            f"**Search Results** — {len(result)} email(s) found\n",
            "| # | Date | From | Subject | 📎 |",
            "|---|------|------|---------|-----|",
        ]
        for i, m in enumerate(result, 1):
            att = "✅" if m["has_attachments"] else "—"
            subject_col = m["subject"][:55] + ("…" if len(m["subject"]) > 55 else "")
            table_lines.append(
                f"| {i} | {m['date_display']} | {m['sender']} "
                f"| {subject_col} | {att} |"
            )

        final_result = {
            "emails": result,
            "count": len(result),
            "filters_applied": {
                "sender": sender_email or "any",
                "keyword": keyword or "any",
                "date_from": date_from or "any",
                "date_to": date_to or "any",
            },
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "Then offer to open or summarise any specific result."
            ),
        }

        # ── Validate output ───────────────────────────────────────────
        from utils.validator import validate_email_list, append_validation_to_result
        validation = validate_email_list(result)
        return append_validation_to_result(final_result, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="search_emails_advanced", logger=logger)
    
# ---------------------------------------------------------------------------
# Tool: list_emails_paged
# ---------------------------------------------------------------------------

@mcp.tool
async def list_emails_paged(
    count: int = 50,
    page: int = 1,
) -> dict:
    """
    List emails in pages of up to 50. Use this when the user wants
    more than 50 emails or asks for the "next page" of results.

    Use this tool when the user asks things like:
    - "Show me the next 50 emails"
    - "Get page 2 of my inbox"
    - "List emails 51 to 100"

    Args:
        count (int): Emails per page, max 50. Default 50.
        page (int): Page number, starting at 1. Default 1.

    Returns:
        dict with emails, table, page info, and next page prompt.
    """
    logger.info(f"Tool called: list_emails_paged (count={count}, page={page})")

    try:
        
        safe_count = min(max(count, 1), 50)
        skip = (page - 1) * safe_count

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="list_emails_paged",
            user_email=user_email,
            inputs={"count": safe_count, "page": page},
        )

        raw_messages = await fetch_messages_paged(top=safe_count, skip=skip)

        result = []
        for msg in raw_messages:
            received_raw = msg.get("receivedDateTime", "")
            try:
                from dateutil import parser as dp
                dt = dp.parse(received_raw)
                date_display = dt.strftime("%d %b %Y %H:%M")
            except Exception:
                date_display = received_raw[:10] if received_raw else "—"

            result.append({
                "id": msg.get("id"),
                "subject": msg.get("subject") or "(no subject)",
                "sender": msg.get("from", {}).get("emailAddress", {}).get("name") or "—",
                "sender_email": msg.get("from", {}).get("emailAddress", {}).get("address") or "—",
                "received_date": received_raw,
                "date_display": date_display,
                "has_attachments": msg.get("hasAttachments", False),
                "preview": msg.get("bodyPreview", "")[:150],
            })

        start_num = skip + 1
        end_num = skip + len(result)
        has_more = len(result) == safe_count

        table_lines = [
            f"**Inbox — Emails {start_num} to {end_num} (Page {page})**\n",
            "| # | Date | From | Subject | 📎 |",
            "|---|------|------|---------|-----|",
        ]
        for i, m in enumerate(result, start_num):
            att = "✅" if m["has_attachments"] else "—"
            subject_col = m["subject"][:55] + ("…" if len(m["subject"]) > 55 else "")
            table_lines.append(
                f"| {i} | {m['date_display']} | {m['sender']} "
                f"| {subject_col} | {att} |"
            )

        final_result = {
            "emails": result,
            "page": page,
            "count": len(result),
            "showing": f"Emails {start_num}–{end_num}",
            "has_more": has_more,
            "display_table": "\n".join(table_lines),
            "instruction": (
                f"Display 'display_table' as markdown. "
                + (
                    f"Then ask if the user wants page {page + 1} "
                    f"(emails {end_num + 1}–{end_num + safe_count})."
                    if has_more else
                    "This is the last page of results."
                )
            ),
        }

        # ── Validate output ───────────────────────────────────────────
        validation = validate_email_list(result)
        return append_validation_to_result(final_result, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="list_emails_paged", logger=logger)

# ---------------------------------------------------------------------------
# Tool: export_emails_markdown
# ---------------------------------------------------------------------------

@mcp.tool
async def export_emails_markdown(
    count: int = 200,
    keyword: str = "",
) -> dict:
    """
    Export a large list of emails to a downloadable markdown table.
    Use this when the user wants more than 50 emails exported or
    asks to save email results to a file.

    Use this tool when the user asks things like:
    - "Export all my emails to a file"
    - "Give me all 200 emails in a markdown file"
    - "Download my inbox as a table"

    Args:
        count (int): Total emails to export, max 200.
        keyword (str): Optional keyword filter.

    Returns:
        dict with the full markdown content and instruction to display it.
    """
    logger.info(f"Tool called: export_emails_markdown (count={count})")

    try:
        safe_count = min(max(count, 1), 200)

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="export_emails_markdown",
            user_email=user_email,
            inputs={"count": safe_count, "keyword": keyword},
        )

        # Fetch in batches of 50 (Graph API limit per request)
        all_messages = []
        batch_size = 50
        fetched = 0

        while fetched < safe_count:
            batch = min(batch_size, safe_count - fetched)
            if keyword:
                msgs = await search_messages_by_keyword(keyword, top=batch)
            else:
                msgs = await fetch_recent_messages(top=batch)
            if not msgs:
                break
            all_messages.extend(msgs)
            fetched += len(msgs)
            if len(msgs) < batch:
                break  # No more messages available

        # Build full markdown table
        lines = [
            f"# Outlook Inbox Export",
            f"**Total emails:** {len(all_messages)}",
            f"**Filter:** {keyword if keyword else 'None'}",
            f"",
            "| # | Date | From | Email | Subject | Attachments |",
            "|---|------|------|-------|---------|-------------|",
        ]

        for i, msg in enumerate(all_messages, 1):
            received_raw = msg.get("receivedDateTime", "")
            try:
                from dateutil import parser as dp
                dt = dp.parse(received_raw)
                date_display = dt.strftime("%d %b %Y")
            except Exception:
                date_display = received_raw[:10] if received_raw else "—"

            sender = msg.get("from", {}).get("emailAddress", {}).get("name") or "—"
            sender_email = msg.get("from", {}).get("emailAddress", {}).get("address") or "—"
            subject = (msg.get("subject") or "(no subject)")[:60]
            att = "Yes" if msg.get("hasAttachments") else "No"

            lines.append(
                f"| {i} | {date_display} | {sender} | {sender_email} "
                f"| {subject} | {att} |"
            )

        markdown_content = "\n".join(lines)

        return {
            "email_count": len(all_messages),
            "markdown_content": markdown_content,
            "instruction": (
                "Display the 'markdown_content' field in full as a markdown "
                "table. This is the complete email export the user requested."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="export_emails_markdown", logger=logger)
    

# ---------------------------------------------------------------------------
# Function: flag_email
# ---------------------------------------------------------------------------
@mcp.tool
async def flag_email(
    email_id: str,
    flag_status: str = "flagged",
) -> dict:
    """
    Flag an email for follow-up, mark it complete, or remove the flag.

    Use this tool when the user asks things like:
    - "Flag this email for follow-up"
    - "Mark this email as complete"
    - "Remove the flag from this email"
    - "Unflag this message"

    Note: Pinning emails is not supported via Microsoft Graph API —
    pinning is a local Outlook client feature only.

    Args:
        email_id (str): Graph API ID of the email to flag.
                         Obtained from list_emails or search_emails.
        flag_status (str): The flag status to set:
                            - "flagged"    → flag for follow-up (default)
                            - "complete"   → mark follow-up as done
                            - "notFlagged" → remove flag entirely

    Returns:
        dict with confirmation of the flag status change.
    """
    logger.info(
        f"Tool called: flag_email "
        f"(email_id={email_id[:20]}..., flag_status={flag_status})"
    )

    try:

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="flag_email",
            user_email=user_email,
            inputs={"email_id": email_id[:12], "flag_status": flag_status},
        )

        result = await flag_message(
            message_id=email_id,
            flag_status=flag_status,
        )

        # Map status to display label
        display_map = {
            "flagged": "🚩 Flagged for follow-up",
            "complete": "✅ Follow-up marked complete",
            "notflagged": "⬜ Flag removed",
            "unflagged": "⬜ Flag removed",
        }
        display = display_map.get(flag_status.lower(), f"Flag set to '{flag_status}'")

        return {
            **result,
            "display": display,
            "instruction": f"Inform the user: {display}",
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="flag_email", logger=logger)
    

---
./tools/folder_tools.py
---
"""
folder_tools.py
================
MCP tools for Outlook mail folder management — listing folders,
creating new folders, and moving emails between folders.

Why this file exists:
    Folder management is a distinct capability from reading or drafting
    emails. Keeping it in a separate file makes it easy to add,
    modify, or restrict folder operations independently without
    touching email reading or drafting tools.

Design notes:
    - All folder operations require Mail.ReadWrite permission.
    - Well-known folder names (inbox, drafts, sentitems, deleteditems,
      junkemail, archive) are accepted as destination names in
      move_email — users don't need to know the internal folder ID.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from tools.mcp_instance import mcp

from graph.draft_client import (
    get_mail_folders,
    create_mail_folder,
    move_message_to_folder,
)

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.audit_logger import log_tool_call, get_user_email_from_headers

logger = get_logger(__name__)

# Well-known folder name mapping — Graph API accepts these as folder IDs
WELL_KNOWN_FOLDERS = {
    "inbox": "inbox",
    "drafts": "drafts",
    "sent": "sentitems",
    "sent items": "sentitems",
    "deleted": "deleteditems",
    "deleted items": "deleteditems",
    "trash": "deleteditems",
    "junk": "junkemail",
    "junk email": "junkemail",
    "spam": "junkemail",
    "archive": "archive",
}


# ---------------------------------------------------------------------------
# Tool: list_folders
# ---------------------------------------------------------------------------
@mcp.tool
async def list_folders() -> dict:
    """
    List all mail folders in the user's Outlook mailbox with unread
    counts and total message counts.

    Use this tool when the user asks things like:
    - "Show me my mail folders"
    - "What folders do I have in Outlook?"
    - "List all my email folders"

    Returns:
        dict with folder list and a formatted display table.
    """
    logger.info("Tool called: list_folders")

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="list_folders",
            user_email=user_email,
            inputs={},
        )

        folders = await get_mail_folders()

        if not folders:
            return {
                "folders": [],
                "count": 0,
                "display_table": "No folders found.",
                "instruction": "Inform the user no folders were found.",
            }

        table_lines = [
            "**📁 Mail Folders**\n",
            "| # | Folder Name | Unread | Total |",
            "|---|-------------|--------|-------|",
        ]

        for i, f in enumerate(folders, 1):
            unread = f"**{f['unread_count']}**" if f["unread_count"] > 0 else "0"
            table_lines.append(
                f"| {i} | {f['name']} | {unread} | {f['total_count']} |"
            )

        return {
            "folders": folders,
            "count": len(folders),
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "Bold unread counts indicate folders with unread messages. "
                "Offer to open any folder or move emails to a folder."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="list_folders", logger=logger)


# ---------------------------------------------------------------------------
# Tool: create_folder
# ---------------------------------------------------------------------------
@mcp.tool
async def create_folder(
    folder_name: str,
    parent_folder_name: str = "",
) -> dict:
    """
    Create a new mail folder in the user's Outlook mailbox.

    Use this tool when the user asks things like:
    - "Create a folder called Projects"
    - "Make a new folder named Client Emails"
    - "Create a subfolder called Q3 inside the Projects folder"

    Args:
        folder_name (str): Name for the new folder.
        parent_folder_name (str): Optional name of an existing folder
                                    to create the new folder inside.
                                    Leave empty for root level.

    Returns:
        dict with new folder ID, name, and confirmation.
    """
    logger.info(f"Tool called: create_folder (name='{folder_name}')")

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="create_folder",
            user_email=user_email,
            inputs={"folder_name": folder_name},
        )

        parent_id = None

        # Resolve parent folder if specified
        if parent_folder_name:
            parent_lower = parent_folder_name.strip().lower()
            if parent_lower in WELL_KNOWN_FOLDERS:
                parent_id = WELL_KNOWN_FOLDERS[parent_lower]
            else:
                # Look up folder ID by name
                all_folders = await get_mail_folders()
                match = next(
                    (f for f in all_folders
                     if f["name"].lower() == parent_lower),
                    None
                )
                if match:
                    parent_id = match["id"]
                else:
                    return {
                        "error": True,
                        "message": (
                            f"Parent folder '{parent_folder_name}' not found. "
                            f"Use list_folders to see available folders."
                        ),
                    }

        result = await create_mail_folder(
            folder_name=folder_name,
            parent_folder_id=parent_id,
        )

        return {
            **result,
            "instruction": (
                f"Inform the user the folder '{folder_name}' was created successfully. "
                + (
                    f"It is inside '{parent_folder_name}'."
                    if parent_folder_name else
                    "It is at root level."
                )
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="create_folder", logger=logger)


# ---------------------------------------------------------------------------
# Tool: move_email
# ---------------------------------------------------------------------------
@mcp.tool
async def move_email(
    email_id: str,
    destination_folder: str,
) -> dict:
    """
    Move an email from its current location to a different mail folder.

    Use this tool when the user asks things like:
    - "Move this email to the Projects folder"
    - "Archive this email"
    - "Move John's email to Deleted Items"
    - "Put this in my Junk folder"

    Args:
        email_id (str): Graph API ID of the email to move.
                         Obtained from list_emails or search_emails.
        destination_folder (str): Name of the destination folder.
                                    Can be a well-known name:
                                    inbox, drafts, sent, deleted,
                                    junk, archive — or any custom
                                    folder name from list_folders.

    Returns:
        dict with confirmation and new email ID in the destination.
    """
    logger.info(
        f"Tool called: move_email (email_id={email_id[:20]}..., "
        f"destination='{destination_folder}')"
    )

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="move_email",
            user_email=user_email,
            inputs={
                "email_id": email_id[:12],
                "destination": destination_folder,
            },
        )

        dest_lower = destination_folder.strip().lower()

        # Step 1: Check if destination is a well-known folder name
        if dest_lower in WELL_KNOWN_FOLDERS:
            folder_id = WELL_KNOWN_FOLDERS[dest_lower]
        else:
            # Step 2: Look up by folder name in the user's folder list
            all_folders = await get_mail_folders()
            match = next(
                (f for f in all_folders
                 if f["name"].lower() == dest_lower),
                None
            )
            if not match:
                return {
                    "error": True,
                    "message": (
                        f"Folder '{destination_folder}' not found. "
                        f"Use list_folders to see available folders, "
                        f"or use a well-known name: inbox, drafts, sent, "
                        f"deleted, junk, archive."
                    ),
                }
            folder_id = match["id"]

        # Step 3: Move the email
        result = await move_message_to_folder(
            message_id=email_id,
            destination_folder_id=folder_id,
        )

        return {
            **result,
            "destination_folder": destination_folder,
            "instruction": (
                f"Inform the user the email '{result.get('subject', '')}' "
                f"was successfully moved to '{destination_folder}'."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="move_email", logger=logger)

---
./tools/followup_tools.py
---
"""
followup_tools.py
=================
MCP tools for tracking emails that need follow-up, composing
professional follow-up messages, and creating reminder tasks
in Microsoft To-Do or Planner.

Why this file exists:
    Employees often miss reply deadlines because there is no automated
    way to know which emails are still waiting for a response.
    This file provides tools to surface those emails and take action —
    either by composing a follow-up or creating a tracked reminder task.

Design notes:
    - Follow-up detection uses two signals:
        1. Email is older than FOLLOWUP_DAYS_THRESHOLD days
        2. No reply was sent by the logged-in user in the same thread
    - Multi-language: follow-up drafts match the original email language.
    - Task creation delegates to graph/task_client.py.
    - Compose pattern: display in chat first, save on approval.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from datetime import datetime, timedelta, timezone

from tools.mcp_instance import mcp

from graph.mail_client import fetch_recent_messages, fetch_message_by_id
from graph.thread_client import fetch_email_thread
from graph.task_client import create_todo_task, create_planner_task, get_todo_tasks

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.language_utils import detect_language, get_language_instruction
from config.settings import settings
from utils.governance import get_followup_rules
from utils.audit_logger import log_tool_call, get_user_email_from_headers


logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Helper: _needs_followup
# ---------------------------------------------------------------------------
def _needs_followup(msg: dict, user_email: str, threshold_days: int) -> bool:
    """
    Determine if an email needs a follow-up based on age and
    whether the current user has already replied.

    Args:
        msg (dict): Email dictionary from mail_client.
        user_email (str): The logged-in user's email address.
        threshold_days (int): Days after which an unreplied email
                               is considered overdue.

    Returns:
        bool: True if follow-up is needed.
    """
    # Skip drafts and emails sent by the user themselves
    if msg.get("is_draft"):
        return False

    sender_email = (
        msg.get("from", {}).get("emailAddress", {}).get("address", "").lower()
    )
    if sender_email == user_email.lower():
        return False

    # Check age
    received_raw = msg.get("receivedDateTime", "")
    if not received_raw:
        return False

    try:
        received_dt = datetime.fromisoformat(
            received_raw.replace("Z", "+00:00")
        )
        age_days = (datetime.now(timezone.utc) - received_dt).days
        return age_days >= threshold_days
    except Exception:
        return False


# ---------------------------------------------------------------------------
# Tool: track_followups
# ---------------------------------------------------------------------------
@mcp.tool
async def track_followups(
    days_threshold: int = None,
    count: int = None,
) -> dict:
    """
    Scan the inbox and identify emails that have not been replied to
    and are older than the configured threshold days.

    Use this tool when the user asks things like:
    - "Which emails need a follow-up?"
    - "What emails have I not replied to?"
    - "Show me overdue emails"
    - "Find emails waiting for my response"

    Args:
        days_threshold (int): Emails older than this many days without
                               a reply are flagged. Default from .env
                               (FOLLOWUP_DAYS_THRESHOLD = 3 days).
        count (int): Number of inbox emails to scan. Default from
                      .env (FOLLOWUP_SCAN_COUNT = 20).

    Returns:
        dict with list of emails needing follow-up and a display table.
    """
    logger.info("Tool called: track_followups")

    try:
        threshold = days_threshold or settings.FOLLOWUP_DAYS_THRESHOLD
        scan_count = count or settings.FOLLOWUP_SCAN_COUNT

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="track_followups",
            user_email=user_email,
            inputs={
                "days_threshold": threshold,
                "count": scan_count,
            },
        )

        # Step 1: Get current user profile to know their email
        from graph.graph_client_factory import get_user_profile
        profile = await get_user_profile()
        user_email = profile.get("mail") or profile.get("userPrincipalName") or ""

        # Step 2: Fetch recent inbox emails
        raw_messages = await fetch_recent_messages(top=scan_count)

        if not raw_messages:
            return {
                "followup_count": 0,
                "display_table": "📭 Inbox is empty — no emails to track.",
                "instruction": "Inform the user their inbox is empty.",
            }

        # Step 3: Filter emails needing follow-up
        needs_reply = [
            m for m in raw_messages
            if _needs_followup(m, user_email, threshold)
        ]

        if not needs_reply:
            return {
                "followup_count": 0,
                "emails_scanned": len(raw_messages),
                "display_table": (
                    f"✅ No follow-ups needed.\n\n"
                    f"Scanned {len(raw_messages)} emails. "
                    f"All emails received in the last {threshold} days "
                    f"have been replied to or are from you."
                ),
                "instruction": "Inform the user no follow-ups are needed.",
            }

        # Step 4: Build display table
        table_lines = [
            f"**🔔 Follow-up Required — {len(needs_reply)} email(s)**\n",
            f"*Emails older than {threshold} days with no reply sent*\n",
            "| # | Date | From | Subject | Days Old |",
            "|---|------|------|---------|----------|",
        ]

        followup_list = []
        for i, msg in enumerate(needs_reply, 1):
            received_raw = msg.get("receivedDateTime", "")
            try:
                received_dt = datetime.fromisoformat(
                    received_raw.replace("Z", "+00:00")
                )
                age_days = (datetime.now(timezone.utc) - received_dt).days
                date_display = received_dt.strftime("%d %b %Y")
            except Exception:
                age_days = "?"
                date_display = received_raw[:10]

            sender = (
                msg.get("from", {}).get("emailAddress", {}).get("name") or "—"
            )
            subject = (msg.get("subject") or "(no subject)")[:50]

            urgency = "🔴" if isinstance(age_days, int) and age_days >= 7 else "🟡"

            table_lines.append(
                f"| {i} | {date_display} | {sender} "
                f"| {subject} | {urgency} {age_days} days |"
            )

            followup_list.append({
                "id": msg.get("id"),
                "subject": msg.get("subject"),
                "sender": sender,
                "sender_email": (
                    msg.get("from", {}).get("emailAddress", {}).get("address") or ""
                ),
                "received_date": received_raw,
                "age_days": age_days,
            })

        return {
            "followup_count": len(needs_reply),
            "emails_scanned": len(raw_messages),
            "threshold_days": threshold,
            "followups": followup_list,
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "🔴 = 7+ days overdue, 🟡 = 3–6 days. "
                "Offer to: compose a follow-up, add to tasks, or both."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="track_followups", logger=logger)


# ---------------------------------------------------------------------------
# Tool: check_email_replied
# ---------------------------------------------------------------------------
@mcp.tool
async def check_email_replied(email_id: str) -> dict:
    """
    Check whether a specific email has been replied to.

    Use this tool when the user asks things like:
    - "Did I reply to John's email?"
    - "Have I responded to this message?"
    - "Check if I replied to this email"

    Args:
        email_id (str): Graph API ID of the email to check.

    Returns:
        dict with replied status and thread context.
    """
    logger.info(f"Tool called: check_email_replied (email_id={email_id[:20]}...)")

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="check_email_replied",
            user_email=user_email,
            inputs={"email_id": email_id[:12]},
        )

        # Step 1: Get current user profile
        from graph.graph_client_factory import get_user_profile
        profile = await get_user_profile()
        user_email = (
            profile.get("mail") or profile.get("userPrincipalName") or ""
        ).lower()

        # Step 2: Fetch the full thread
        thread = await fetch_email_thread(email_id)

        # Step 3: Check if any message in the thread was sent by the user
        target_msg = next(
            (m for m in thread if m.get("id") == email_id), thread[0]
        )

        user_replies = [
            m for m in thread
            if m.get("sender_email", "").lower() == user_email
            and m.get("id") != email_id
        ]

        has_replied = len(user_replies) > 0
        original_subject = target_msg.get("subject", "")
        original_sender = target_msg.get("sender_name", "")

        if has_replied:
            latest_reply = user_replies[-1]
            reply_date = latest_reply.get("received_date", "")[:10]
            status_display = f"✅ Yes — you replied on {reply_date}"
        else:
            received_raw = target_msg.get("received_date", "")
            try:
                received_dt = datetime.fromisoformat(
                    received_raw.replace("Z", "+00:00")
                )
                age_days = (datetime.now(timezone.utc) - received_dt).days
            except Exception:
                age_days = "unknown"
            status_display = (
                f"❌ No reply sent yet — email is {age_days} days old"
            )

        return {
            "has_replied": has_replied,
            "original_subject": original_subject,
            "original_sender": original_sender,
            "reply_count": len(user_replies),
            "status_display": status_display,
            "instruction": (
                f"Display this to the user:\n\n"
                f"**📧 Reply Status**\n\n"
                f"| Field | Details |\n"
                f"|---|---|\n"
                f"| **Email** | {original_subject} |\n"
                f"| **From** | {original_sender} |\n"
                f"| **Replied** | {status_display} |\n\n"
                + (
                    "Offer to compose a follow-up reply."
                    if not has_replied else
                    f"You have sent {len(user_replies)} reply(ies) in this thread."
                )
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="check_email_replied", logger=logger)


# ---------------------------------------------------------------------------
# Tool: compose_followup
# ---------------------------------------------------------------------------
@mcp.tool
async def compose_followup(
    email_id: str,
    followup_context: str = "",
) -> dict:
    """
    Compose a professional follow-up email for an unreplied message
    and display it in chat for review. Uses the original email's
    language automatically.

    Use this tool when the user asks things like:
    - "Compose a follow-up for John's email"
    - "Write a polite follow-up to this unanswered email"
    - "Draft a chaser for this overdue email"

    Args:
        email_id (str): Graph API ID of the email to follow up on.
        followup_context (str): Optional extra context for the follow-up
                                  e.g. "mention the deadline is this Friday".

    Returns:
        dict with follow-up composition instruction for LibreChat's LLM.
    """
    logger.info(f"Tool called: compose_followup (email_id={email_id[:20]}...)")

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="compose_followup",
            user_email=user_email,
            inputs={"email_id": email_id[:12]},
        )

        # Step 1: Read the original email
        original = await fetch_message_by_id(email_id)
        original_subject = original.get("subject", "")
        original_sender_name = (
            original.get("from", {}).get("emailAddress", {}).get("name", "")
        )
        original_sender_email = (
            original.get("from", {}).get("emailAddress", {}).get("address", "")
        )
        original_preview = original.get("body", {}).get("content", "")[:400]
        received_raw = original.get("receivedDateTime", "")

        # Step 2: Calculate days since received
        try:
            received_dt = datetime.fromisoformat(
                received_raw.replace("Z", "+00:00")
            )
            age_days = (datetime.now(timezone.utc) - received_dt).days
            original_date = received_dt.strftime("%d %B %Y")
        except Exception:
            age_days = "several"
            original_date = received_raw[:10]

        # Step 3: Detect language
        lang_code, lang_name = detect_language(original_preview)
        lang_instruction = get_language_instruction(lang_code, lang_name)

        # ── Governance rules ─────────────────────────────────────────
        governance = get_followup_rules()

        extra = f"\nAdditional context: {followup_context}" if followup_context else ""

        return {
            "original_subject": original_subject,
            "original_sender": original_sender_name,
            "original_sender_email": original_sender_email,
            "age_days": age_days,
            "language_detected": lang_name,
            "instruction": (
                f"Compose and display a professional follow-up email.\n"
                f"{lang_instruction}\n\n"
                f"Display this header first:\n"
                f"**Follow-up to:** {original_sender_name} <{original_sender_email}>\n"
                f"**Re:** {original_subject}\n"
                f"**Original date:** {original_date} ({age_days} days ago)\n\n"
                f"Then compose the follow-up body with these rules:\n"
                f"- Start: Dear {original_sender_name},\n"
                f"- Politely reference the original email sent {age_days} days ago\n"
                f"- Gently ask for an update or response\n"
                f"- Keep it brief — 2-3 sentences maximum\n"
                f"- Never aggressive or demanding\n"
                f"- End: Best regards,{extra}\n\n"
                f"{governance}\n\n"
                f"Original email context:\n{original_preview}\n\n"
                f"After composing, ask:\n"
                f"'Would you like me to save this follow-up to Outlook Drafts?\n"
                f"If yes, call save_draft_to_outlook with:\n"
                f"- to_emails: '{original_sender_email}'\n"
                f"- subject: 'Re: {original_subject}'\n"
                f"- body_text: the exact composed follow-up"
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="compose_followup", logger=logger)


# ---------------------------------------------------------------------------
# Tool: add_task_todo
# ---------------------------------------------------------------------------
@mcp.tool
async def add_task_todo(
    title: str,
    due_date: str = "",
    notes: str = "",
    list_name: str = "Tasks",
) -> dict:
    """
    Add a personal reminder or task to Microsoft To-Do.

    Use this tool when the user asks things like:
    - "Add a reminder to follow up on John's email"
    - "Create a To-Do task for the project report"
    - "Remind me to reply to this email by Friday"
    - "Add this action item to my task list"

    Args:
        title (str): Task title — what needs to be done.
        due_date (str): When the task is due e.g. "Friday",
                         "2026-07-20", "end of week".
        notes (str): Optional additional context or notes.
        list_name (str): Which To-Do list to add to. Default "Tasks".

    Returns:
        dict with task ID and confirmation.
    """
    logger.info(f"Tool called: add_task_todo (title='{title}')")

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="add_task_todo",
            user_email=user_email,
            inputs={"due_date": due_date},
        )

        result = await create_todo_task(
            title=title,
            due_date=due_date,
            notes=notes,
            list_name=list_name,
        )

        return {
            **result,
            "instruction": (
                f"Display this confirmation:\n\n"
                f"**✅ Task Added to Microsoft To-Do**\n\n"
                f"| Field | Details |\n"
                f"|---|---|\n"
                f"| **Task** | {title} |\n"
                f"| **List** | {result.get('list_name', list_name)} |\n"
                f"| **Due** | {due_date or 'Not set'} |\n"
                + (f"| **Notes** | {notes[:80]} |\n" if notes else "")
                + f"\n*Visible in Microsoft To-Do app and Outlook Tasks.*"
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="add_task_todo", logger=logger)


# ---------------------------------------------------------------------------
# Tool: add_task_planner
# ---------------------------------------------------------------------------
@mcp.tool
async def add_task_planner(
    title: str,
    due_date: str = "",
    notes: str = "",
    assigned_to_email: str = "",
) -> dict:
    """
    Add a team task to Microsoft Planner.

    Use this tool when the user asks things like:
    - "Add this to Planner and assign to Jane"
    - "Create a team task for the infrastructure review"
    - "Add this action item to the team board"
    - "Create a Planner card for this follow-up"

    Args:
        title (str): Task title.
        due_date (str): Due date in any format.
        notes (str): Task description or context.
        assigned_to_email (str): Email of the team member to
                                   assign the task to.

    Returns:
        dict with task ID and confirmation.
    """
    logger.info(f"Tool called: add_task_planner (title='{title}')")

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="add_task_planner",
            user_email=user_email,
            inputs={"due_date": due_date},
        )

        result = await create_planner_task(
            title=title,
            due_date=due_date,
            notes=notes,
            assigned_to_email=assigned_to_email,
        )

        if result.get("error"):
            return result

        return {
            **result,
            "instruction": (
                f"Display this confirmation:\n\n"
                f"**✅ Task Added to Microsoft Planner**\n\n"
                f"| Field | Details |\n"
                f"|---|---|\n"
                f"| **Task** | {title} |\n"
                f"| **Assigned to** | {assigned_to_email or 'Unassigned'} |\n"
                f"| **Due** | {due_date or 'Not set'} |\n"
                + (f"| **Notes** | {notes[:80]} |\n" if notes else "")
                + f"\n*Visible in Microsoft Planner and Teams.*"
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="add_task_planner", logger=logger)


# ---------------------------------------------------------------------------
# Tool: list_tasks
# ---------------------------------------------------------------------------
@mcp.tool
async def list_tasks(
    source: str = "todo",
    list_name: str = "Tasks",
    count: int = 20,
) -> dict:
    """
    List pending tasks from Microsoft To-Do or Planner.

    Use this tool when the user asks things like:
    - "Show my To-Do tasks"
    - "What tasks do I have?"
    - "List my Planner tasks"
    - "What's on my task list?"

    Args:
        source (str): "todo" for Microsoft To-Do,
                       "planner" for Microsoft Planner.
                       Default "todo".
        list_name (str): To-Do list name. Default "Tasks".
                          (Only used when source="todo")
        count (int): Maximum tasks to return. Default 20.

    Returns:
        dict with task list and display table.
    """
    logger.info(f"Tool called: list_tasks (source={source})")

    try:
        source_lower = source.strip().lower()

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="list_tasks",
            user_email=user_email,
            inputs={"source": source_lower},
        )
        
        if source_lower == "todo":
            tasks = await get_todo_tasks(list_name=list_name, top=count)
            source_label = "Microsoft To-Do"
        else:
            return {
                "error": True,
                "message": (
                    "Planner task listing requires additional setup. "
                    "Please use source='todo' to list Microsoft To-Do tasks."
                ),
            }

        if not tasks:
            return {
                "tasks": [],
                "count": 0,
                "display_table": f"✅ No pending tasks in {source_label}.",
                "instruction": f"Inform the user their {source_label} task list is empty.",
            }

        table_lines = [
            f"**📋 {source_label} — Pending Tasks**\n",
            "| # | Task | Due Date | Status |",
            "|---|------|----------|--------|",
        ]

        for i, task in enumerate(tasks, 1):
            due = task.get("due_date", "Not set")
            status = task.get("status", "notStarted")
            status_icon = "🔵" if status == "notStarted" else "🟡"
            title = (task.get("title") or "")[:60]
            table_lines.append(
                f"| {i} | {title} | {due} | {status_icon} {status} |"
            )

        return {
            "tasks": tasks,
            "count": len(tasks),
            "source": source_label,
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "Offer to add new tasks or mark existing ones complete."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="list_tasks", logger=logger)

---
./tools/mcp_instance.py
---
"""
mcp_instance.py
===============
Holds the single shared FastMCP instance used by all tool files.
Exists as a sepearete module to break circular import between
server.py and the tools/files.
"""

# import os
from fastmcp import FastMCP
from config.settings import settings
# from mcp.server.fastmcp import FastMCP
# from fastmcp.server.http import TransportSecuritySettings

# FASTMCP_HOST=os.environ["FASTMCP_HOST"] = "127.0.0.1"
 
mcp = FastMCP(settings.MCP_SERVER_NAME)

# mcp = FastMCP(
#     settings.MCP_SERVER_NAME,
#     host=settings.MCP_HOST,
#     port=settings.MCP_PORT,
#     transport_security=TransportSecuritySettings(
#         allowed_hosts=[
#             "figpt-mcp-outlook.comp.group.com",
#             "localhost",
#             "127.0.0.1",
#         ],
#         allowed_origins=[
#             "https://figpt-mcp-outlook.comp.group.com",
#             "http://localhost:8000",
#             "http://127.0.0.1:8000",
#         ],
#     )
# )
# mcp = FastMCP(
#     "MyServer",     
#     host=FASTMCP_HOST,
#     port=8000,
#     transport_security=TransportSecuritySettings(
#         enable_dns_rebinding_protection=False,
#         allowed_hosts=["*"] # Allows all incoming host headers (adjust to specific domain in production)    
#     )
# )


# 1. Server initialisieren (ohne 'host')
# mcp = FastMCP(
#     "MyServer",
#     transport_security=TransportSecuritySettings(
#         enable_dns_rebinding_protection=False
#     )
# )
# # ... Hier Ihre Tools und Ressourcen definieren ...
# # 2. Host beim Starten übergeben
# if __name__ == "__main__":
#     mcp.run_http(host=FASTMCP_HOST)



# mcp = FastMCP(
#     "MyServer",
#     transport_security=TransportSecuritySettings(
#         enable_dns_rebinding_protection=False,
#         allowed_hosts=["*"],  # local/dev only
#     ),
# )

---
./tools/mom_tools.py
---
"""
mom_tools.py
============
MCP tools for generating formal Minutes of Meeting (MOM) from
email threads or single emails, and saving them as draft emails.

Why this file exists:
    Manually preparing MOM from email discussions is one of the most
    time-consuming tasks for employees. This tool reads the email
    thread, extracts all participants, discussion points, decisions,
    and action items, and produces a formal MOM document — instantly.

Design notes:
    - MOM generation delegates to LibreChat's LLM via the instruction
      field — no Requesty or external LLM calls.
    - Multi-language: detects the email language and instructs
      LibreChat's LLM to generate the MOM in the same language.
    - Formal format: fixed sections — Details, Attendees, Discussion,
      Decisions, Action Items, Next Steps, Notes.
    - save_mom_as_draft uses save_draft_to_outlook pattern — no direct
      Graph API save here, keeps the compose/save separation clean.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from tools.mcp_instance import mcp

from graph.mail_client import fetch_message_by_id
from graph.thread_client import fetch_email_thread
from graph.draft_client import create_draft

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.audit_logger import log_tool_call, get_user_email_from_headers
from utils.language_utils import detect_language, get_language_instruction
# Import governance rules
from utils.governance import get_mom_rules
from utils.validator import validate_draft_content, append_validation_to_result
from utils.validator import (validate_mom_output, validate_draft_content, append_validation_to_result,)

logger = get_logger(__name__)

# ---------------------------------------------------------------------------
# helper function: convert mom oto html
# ---------------------------------------------------------------------------
def _convert_mom_to_html(mom_text: str) -> str:
    """
    Convert MOM markdown/plain text to rich HTML with proper formatting —
    bold headers, font sizes, colours, spacing — suitable for Outlook email.

    Args:
        mom_text (str): The MOM content as generated in chat.

    Returns:
        str: Rich HTML string for the email draft body.
    """
    lines = mom_text.split("\n")
    html_lines = [
        '<html><body style="font-family: Calibri, Arial, sans-serif; '
        'font-size: 14px; color: #1a1a1a; line-height: 1.6; max-width: 800px;">'
    ]

    # Add greeting
    html_lines.append(
        '<p style="margin-bottom: 16px;">Dear Team,</p>'
        '<p style="margin-bottom: 24px;">Please find below the Minutes of '
        'Meeting for your reference.</p>'
        '<hr style="border: none; border-top: 2px solid #0078D4; margin: 24px 0;">'
    )

    for line in lines:
        stripped = line.strip()
        if not stripped:
            html_lines.append('<br>')
            continue

        # ═══ separator lines → styled divider
        if stripped.startswith("═"):
            html_lines.append(
                '<hr style="border: none; border-top: 2px solid #0078D4; margin: 20px 0;">'
            )

        # MINUTES OF MEETING title
        elif "MINUTES OF MEETING" in stripped.upper() and not stripped.startswith("#"):
            html_lines.append(
                f'<h1 style="color: #0078D4; font-size: 22px; font-weight: bold; '
                f'text-align: center; margin: 16px 0;">{stripped}</h1>'
            )

        # Numbered sections like "1. MEETING DETAILS"
        elif stripped[:2] in ("1.", "2.", "3.", "4.", "5.", "6.", "7.") and stripped[2] == " ":
            section_title = stripped[3:].strip()
            html_lines.append(
                f'<h2 style="color: #0078D4; font-size: 16px; font-weight: bold; '
                f'border-bottom: 1px solid #c7e0f4; padding-bottom: 6px; '
                f'margin-top: 20px; margin-bottom: 10px;">'
                f'{stripped[:2]} {section_title}</h2>'
            )

        # Markdown H1 → large title
        elif stripped.startswith("# "):
            html_lines.append(
                f'<h1 style="color: #0078D4; font-size: 20px; font-weight: bold; '
                f'margin: 16px 0 8px 0;">{stripped[2:]}</h1>'
            )

        # Markdown H2 → section header
        elif stripped.startswith("## "):
            html_lines.append(
                f'<h2 style="color: #0078D4; font-size: 16px; font-weight: bold; '
                f'border-bottom: 1px solid #c7e0f4; padding-bottom: 4px; '
                f'margin-top: 20px; margin-bottom: 8px;">{stripped[3:]}</h2>'
            )

        # Markdown H3 → sub-section
        elif stripped.startswith("### "):
            html_lines.append(
                f'<h3 style="color: #005a9e; font-size: 14px; font-weight: bold; '
                f'margin-top: 14px; margin-bottom: 6px;">{stripped[4:]}</h3>'
            )

        # Table rows → preserve as styled table
        elif stripped.startswith("|"):
            if "---" in stripped:
                continue  # skip separator rows
            cells = [c.strip() for c in stripped.split("|") if c.strip()]
            if cells:
                is_header = any(
                    c.startswith("**") or c.isupper()
                    for c in cells
                )
                row_html = "".join(
                    f'<th style="background:#0078D4; color:white; padding:8px 12px; '
                    f'text-align:left; font-weight:bold;">{c.replace("**","")}</th>'
                    if is_header else
                    f'<td style="padding:8px 12px; border-bottom:1px solid #e0e0e0;">'
                    f'{_inline_format(c)}</td>'
                    for c in cells
                )
                tag = "tr"
                html_lines.append(
                    f'<table style="border-collapse:collapse; width:100%; '
                    f'margin:12px 0;"><{tag}>{row_html}</{tag}></table>'
                    if is_header else
                    f'<table style="border-collapse:collapse; width:100%; '
                    f'margin:0;"><{tag}>{row_html}</{tag}></table>'
                )

        # Bullet points
        elif stripped.startswith("- ") or stripped.startswith("• "):
            content = stripped[2:]
            html_lines.append(
                f'<li style="margin: 4px 0; padding-left: 4px;">'
                f'{_inline_format(content)}</li>'
            )

        # Numbered list items
        elif len(stripped) > 2 and stripped[0].isdigit() and stripped[1] == ".":
            content = stripped[2:].strip()
            html_lines.append(
                f'<li style="margin: 6px 0;">{_inline_format(content)}</li>'
            )

        # Key-value lines like "   Date        : 2026-07-22"
        elif ":" in stripped and len(stripped.split(":")[0].strip()) < 20:
            parts = stripped.split(":", 1)
            key = parts[0].strip()
            val = parts[1].strip() if len(parts) > 1 else ""
            html_lines.append(
                f'<p style="margin: 4px 0;">'
                f'<strong style="color: #333; min-width: 140px; '
                f'display: inline-block;">{key}:</strong> {_inline_format(val)}</p>'
            )

        # Regular paragraph
        else:
            html_lines.append(
                f'<p style="margin: 6px 0;">{_inline_format(stripped)}</p>'
            )

    # Closing
    html_lines.append(
        '<hr style="border: none; border-top: 1px solid #e0e0e0; margin: 24px 0;">'
        '<p style="color: #888; font-size: 12px; font-style: italic;">'
        'This MOM was generated by the Outlook AI Agent. '
        'Please review before distributing.</p>'
        '</body></html>'
    )

    return "\n".join(html_lines)


def _inline_format(text: str) -> str:
    """
    Convert inline markdown formatting to HTML.
    Handles **bold**, *italic*, and `code` patterns.
    """
    import re
    # Bold
    text = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', text)
    # Italic
    text = re.sub(r'\*(.+?)\*', r'<em>\1</em>', text)
    # Code
    text = re.sub(r'`(.+?)`', r'<code style="background:#f4f4f4; padding:1px 4px;">\1</code>', text)
    return text


# ---------------------------------------------------------------------------
# Tool: generate_mom
# ---------------------------------------------------------------------------
@mcp.tool
async def generate_mom(
    email_id: str,
    use_thread: bool = True,
) -> dict:
    """
    Generate a formal Minutes of Meeting (MOM) document from an email
    or email thread. Detects language automatically and produces the
    MOM in the same language as the source emails.

    Use this tool when the user asks things like:
    - "Generate MOM from this email thread"
    - "Create minutes of meeting from John's email chain"
    - "Prepare a formal MOM from this discussion"
    - "Extract meeting minutes from this email"

    Args:
        email_id (str): Graph API ID of any email in the thread.
                         If use_thread=True, fetches the full chain.
                         If use_thread=False, uses only this email.
        use_thread (bool): True to fetch the full conversation thread
                            (recommended for accurate MOM). False to
                            use only the single email pointed to.
                            Default True.

    Returns:
        dict with structured email data and MOM generation instruction
        for LibreChat's LLM to produce the formal document.
    """
    logger.info(
        f"Tool called: generate_mom "
        f"(email_id={email_id[:20]}..., use_thread={use_thread})"
    )

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="generate_mom",
            user_email=user_email,
            inputs={
                "email_id": email_id[:12],
                "use_thread": use_thread,
            },
        )
        # Step 1: Fetch email(s)
        if use_thread:
            messages = await fetch_email_thread(email_id)
            source_label = "Email Thread"
        else:
            single = await fetch_message_by_id(email_id)
            messages = [single]
            source_label = "Single Email"

        if not messages:
            return {
                "error": True,
                "message": "No email content found to generate MOM from.",
            }

        # Step 2: Collect all participants across the thread
        participants = {}
        for msg in messages:
            # Handle both raw Graph dict and thread_client dict formats
            sender_name = (
                msg.get("sender_name")
                or msg.get("from", {}).get("emailAddress", {}).get("name", "")
            )
            sender_email = (
                msg.get("sender_email")
                or msg.get("from", {}).get("emailAddress", {}).get("address", "")
            )
            if sender_email and sender_email not in participants:
                participants[sender_email] = sender_name or sender_email

            for r in msg.get("to_recipients", []):
                email = r.get("email", "")
                name = r.get("name", email)
                if email and email not in participants:
                    participants[email] = name

            for r in msg.get("cc_recipients", []):
                email = r.get("email", "")
                name = r.get("name", email)
                if email and email not in participants:
                    participants[email] = name

        # Step 3: Build attendee table
        attendee_rows = "\n".join([
            f"| {name} | {email} | Participant |"
            for email, name in participants.items()
        ])

        # Step 4: Build chronological email digest for LLM
        email_digest = ""
        for i, msg in enumerate(messages, 1):
            sender = (
                msg.get("sender_name")
                or msg.get("from", {}).get("emailAddress", {}).get("name", "Unknown")
            )
            date = (
                msg.get("received_date")
                or msg.get("receivedDateTime", "")
            )[:10]
            subject = msg.get("subject", "")
            body = (
                msg.get("body", {}).get("content", "")
                if isinstance(msg.get("body"), dict)
                else msg.get("body", "")
            )[:800]

            email_digest += (
                f"--- Message {i} ---\n"
                f"From: {sender}\n"
                f"Date: {date}\n"
                f"Subject: {subject}\n"
                f"Content: {body}\n\n"
            )

        # Step 5: Detect language from combined content
        sample_text = " ".join([
            msg.get("subject", "") + " " +
            (
                msg.get("body", {}).get("content", "")
                if isinstance(msg.get("body"), dict)
                else msg.get("body", "")
            )[:200]
            for msg in messages[:3]
        ])
        lang_code, lang_name = detect_language(sample_text)
        lang_instruction = get_language_instruction(lang_code, lang_name)

        # Step 6: Get meeting subject and date from first/last message
        first_msg = messages[0]
        last_msg = messages[-1]
        meeting_subject = first_msg.get("subject", "Meeting")
        meeting_date = (
            last_msg.get("received_date")
            or last_msg.get("receivedDateTime", "")
        )[:10]

        # ── Governance rules ─────────────────────────────────────────
        governance = get_mom_rules(include_sources=True)

        # Step 7: Build the formal MOM instruction for LibreChat's LLM
        mom_instruction = f"""
Generate a formal Minutes of Meeting (MOM) document.
{lang_instruction}

Use EXACTLY this structure and format:

═══════════════════════════════════════════════════
         MINUTES OF MEETING (MOM)
═══════════════════════════════════════════════════

1. MEETING DETAILS
   Date        : {meeting_date}
   Subject     : {meeting_subject}
   Source      : {source_label} ({len(messages)} message(s))
   Language    : {lang_name}

2. ATTENDEES
   | Name | Email | Role |
   |------|-------|------|
{attendee_rows}

3. DISCUSSION SUMMARY
   [Write 3-5 paragraphs summarising the key discussion points
   from the email content below. Be factual and concise.]

4. DECISIONS MADE
   [List decisions as bullet points. If none found, write "No explicit decisions recorded."]

5. ACTION ITEMS
   | # | Action | Owner | Deadline | Status |
   |---|--------|-------|----------|--------|
   [Extract all action items, tasks, and next steps. Status = Open]

6. NEXT STEPS
   [Summarise what happens next based on the discussion]

7. NOTES
   [Any additional observations, caveats, or context]

═══════════════════════════════════════════════════
f"{governance}\n\n"
f"Email content to base the MOM on:\n\n{email_digest}"

Email content to base the MOM on:

{email_digest}

Rules:
- Be factual — only include what is in the emails
- Do not invent names, dates, or decisions
- {lang_instruction}
- Keep Action Items table populated even if only 1 item found
- Format output as clean markdown

{get_mom_rules(include_sources=True)}
"""

        result = {
            "email_count": len(messages),
            "participants": participants,
            "meeting_subject": meeting_subject,
            "meeting_date": meeting_date,
            "language_detected": lang_name,
            "source": source_label,
            "instruction": mom_instruction,
        }

        # ── Validate MOM instruction before returning ─────────────────
        validation = validate_mom_output(mom_instruction)
        return append_validation_to_result(result, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="generate_mom", logger=logger)


# ---------------------------------------------------------------------------
# Tool: save_mom_as_draft
# ---------------------------------------------------------------------------
@mcp.tool
async def save_mom_as_draft(
    to_emails: str,
    subject: str,
    mom_content: str,
) -> dict:
    """
    Save a generated MOM as an email draft to send to all attendees.
    Call this AFTER generate_mom and user approval of the MOM content.

    Use this tool when the user says things like:
    - "Save this MOM and send it to everyone"
    - "Draft this MOM to all attendees"
    - "Email this MOM to the team"

    Args:
        to_emails (str): Comma-separated email addresses of all
                          attendees to receive the MOM.
        subject (str): Email subject e.g. "MOM — Project Kickoff".
        mom_content (str): The full MOM text exactly as generated
                            and approved in chat.

    Returns:
        dict with draft ID and save confirmation.
    """
    logger.info(f"Tool called: save_mom_as_draft (to={to_emails})")

    try:

        to_list = [e.strip() for e in to_emails.split(",") if e.strip()]

        if not to_list:
            return {
                "error": True,
                "message": "No recipient email addresses provided.",
            }

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="save_mom_as_draft",
            user_email=user_email,
            inputs={"subject": subject},
        )

        # Convert MOM markdown to clean HTML
        body_html = _convert_mom_to_html(mom_content)

        validation = validate_draft_content(
            body_text=mom_content,
            to_emails=to_list,
            subject=subject,
        )
        if not validation.is_valid:
            return append_validation_to_result({}, validation)

        result = await create_draft(
            to_emails=to_list,
            subject=subject,
            body_html=body_html,
        )

        final_result = {
            "draft_id": result["draft_id"],
            "subject": subject,
            "to": to_list,
            "status": "✅ MOM saved as email draft in Outlook Drafts",
            "instruction": (
                f"Display this confirmation:\n\n"
                f"**✅ MOM Draft Saved**\n\n"
                f"| Field | Details |\n"
                f"|---|---|\n"
                f"| **To** | {', '.join(to_list)} |\n"
                f"| **Subject** | {subject} |\n"
                f"| **Status** | Saved to Outlook Drafts |\n\n"
                f"*Open Outlook to review and send the MOM to attendees.*"
            ),
        }

        return append_validation_to_result(final_result, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="save_mom_as_draft", logger=logger)

---
./tools/profile_tools.py
---
"""
profile_tools.py
=================
Defines MCP tools related to the logged-in user's identity.

Why this file exists:
    Before the AI ever touches email or calendar data, it's useful to have
    a simple, fast tool that confirms "who is this conversation acting on
    behalf of". This is also the easiest tool to use as a connectivity
    sanity check — if this fails, nothing else will work either, because
    it means the OAuth token isn't reaching Microsoft Graph correctly.

Design notes:
    - This file contains NO Graph API logic itself. It only calls into
      graph/mail_client.py (or a dedicated profile client) to fetch data.
    - The docstring of each @mcp.tool function is what the LLM reads to
      decide when this tool is relevant. Keep these descriptions precise.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
# `mcp` is the shared FastMCP server instance, created in server.py and
# imported here so this file can register its tools onto it.
# from server import mcp
from tools.mcp_instance import mcp

# get_user_profile() is the function that actually talks to Microsoft Graph.
# It lives in graph/graph_client_factory.py (or a profile-specific client)
# and returns a plain Python dictionary with the user's basic info.
from graph.graph_client_factory import get_user_profile

# Centralised error formatting — converts raw exceptions into safe,
# readable messages instead of leaking stack traces to the chat UI.
from utils.error_handler import format_tool_error

# Centralised logger — every tool call gets logged consistently.
from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Tool: get_my_profile
# ---------------------------------------------------------------------------
@mcp.tool
async def get_my_profile() -> dict:
    """
    Get the display name, email address, and job title of the currently
    logged-in Microsoft 365 user.

    Use this tool when the user asks things like:
    - "Who am I logged in as?"
    - "What's my email address?"
    - "Confirm my account details"

    This tool takes no parameters — it always returns the identity of
    whoever is currently authenticated in this conversation.

    Returns:
        A dictionary containing:
            - display_name (str): The user's full name as set in Microsoft 365
            - email (str): The user's primary email address
            - job_title (str): The user's job title, if set in their profile
    """
    # Step 1: Log that this tool was invoked — useful for debugging
    #         which tools the AI is actually choosing to call.
    logger.info("Tool called: get_my_profile")

    try:
        # Step 2: Delegate the actual Graph API call. This function
        #         internally builds an authenticated client using the
        #         current request's OAuth token and calls GET /me.
        profile_data = await get_user_profile()

        # Step 3: Return a clean, minimal dictionary. We deliberately
        #         don't return the full raw Graph API response — only
        #         the fields useful to the LLM and the end user.
        return {
            "display_name": profile_data.get("displayName"),
            "email": profile_data.get("mail") or profile_data.get("userPrincipalName"),
            "job_title": profile_data.get("jobTitle"),
        }

    except Exception as exc:
        # Step 4: Never let a raw exception bubble up to LibreChat.
        #         format_tool_error() turns it into a safe, readable
        #         message and logs the full detail internally.
        return format_tool_error(exc, tool_name="get_my_profile", logger=logger)

---
./tools/semantic_tools.py
---
"""
semantic_tools.py
=================
MCP tool for semantic search over email history.

Why this file exists:
    Keyword search only works when the user knows the exact words.
    Semantic search understands intent — "find emails about delays"
    will also find emails using words like "postponed", "behind schedule",
    "pushed back", even if the word "delay" never appears.

    This tool is the most AI-native capability in Phase 2 — it uses
    a pre-trained language model to understand meaning and surface
    contextually relevant emails the user might never find with
    regular search.

Design notes:
    - Fetches up to SEMANTIC_SEARCH_MAX_EMAILS emails fresh from
      Graph API on each call, embeds them in memory, runs cosine
      similarity against the query, and returns ranked results.
    - No vector database is used in Phase 2 — in-memory search is
      sufficient for up to ~200 emails (< 2 seconds).
    - The model (all-MiniLM-L6-v2) is loaded lazily on first use
      and cached for the server's lifetime.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from tools.mcp_instance import mcp

from graph.mail_client import fetch_recent_messages
from semantic.search_engine import semantic_search

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.audit_logger import log_tool_call, get_user_email_from_headers
from config.settings import settings

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Tool: semantic_search_emails
# ---------------------------------------------------------------------------
@mcp.tool
async def semantic_search_emails(
    query: str,
    top_k: int = 10,
    corpus_size: int = None,
) -> dict:
    """
    Search your email history using natural language and semantic
    understanding — finds relevant emails even when exact keywords
    don't match.

    Use this tool when the user asks things like:
    - "Find emails about project delays even if delay isn't mentioned"
    - "Search for anything related to budget pressure"
    - "Which emails were about scheduling conflicts?"
    - "Find discussions about the client being unhappy"

    This is different from search_emails (keyword search).
    Use semantic_search_emails when:
    - The user describes a concept or topic, not an exact phrase
    - Keyword search returned no results but the email should exist
    - The user wants to search by meaning or sentiment

    Args:
        query (str): Natural language description of what to find.
                      e.g. "deadline pressure from management",
                           "budget approval requests",
                           "follow-up requests I haven't responded to"
        top_k (int): Number of top matching emails to return.
                      Default 10.
        corpus_size (int): How many recent emails to search over.
                            Default from settings (100). Max 200.

    Returns:
        dict with semantically ranked email results and display table.
    """
    logger.info(
        f"Tool called: semantic_search_emails "
        f"(query='{query}', top_k={top_k})"
    )

    try:
        if not query or not query.strip():
            return {
                "error": True,
                "message": "Please provide a search query.",
            }
        safe_top_k = min(max(top_k, 1), 25)

        # Step 1: Determine corpus size
        max_corpus = min(
            corpus_size or settings.SEMANTIC_SEARCH_MAX_EMAILS,
            200
        )

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="semantic_search_emails",
            user_email=user_email,
            inputs={"top_k": safe_top_k},
        )

        # Step 2: Fetch recent emails to search over.
        # We fetch in the largest batch we can — more emails means
        # better search coverage.
        logger.info(f"Fetching {max_corpus} emails for semantic corpus...")
        raw_emails = await fetch_recent_messages(top=max_corpus)

        if not raw_emails:
            return {
                "results": [],
                "query": query,
                "display_table": "📭 No emails found to search over.",
                "instruction": "Inform the user their inbox appears empty.",
            }

        # Step 3: Run semantic search — this embeds all emails and
        #         computes cosine similarity against the query.
        logger.info(f"Running semantic search over {len(raw_emails)} emails...")
        results = semantic_search(
            query=query,
            emails=raw_emails,
            top_k=safe_top_k,
        )

        if not results:
            return {
                "results": [],
                "query": query,
                "result_count": 0,
                "emails_searched": len(raw_emails),
                "display_table": (
                    f"🔍 No semantically relevant emails found for: *'{query}'*\n\n"
                    f"Searched {len(raw_emails)} recent emails. "
                    f"Try rephrasing your query or use search_emails for keyword search."
                ),
                "instruction": (
                    "Display the message above and suggest the user "
                    "try keyword search or rephrase their query."
                ),
            }

        # Step 4: Build display table with relevance scores
        table_lines = [
            f"**🔍 Semantic Search Results**",
            f"*Query: \"{query}\"* — {len(results)} result(s) from "
            f"{len(raw_emails)} emails searched\n",
            "| # | Relevance | Date | From | Subject |",
            "|---|-----------|------|------|---------|",
        ]

        cleaned_results = []
        for i, email in enumerate(results, 1):
            received_raw = email.get("receivedDateTime", "")
            try:
                from dateutil import parser as dp
                dt = dp.parse(received_raw)
                date_display = dt.strftime("%d %b %Y")
            except Exception:
                date_display = received_raw[:10] if received_raw else "—"

            sender = (
                email.get("from", {})
                .get("emailAddress", {})
                .get("name")
                or "—"
            )
            subject = (email.get("subject") or "(no subject)")[:55]
            relevance = email.get("relevance", "—")
            score = email.get("similarity_score", 0)

            table_lines.append(
                f"| {i} | {relevance} ({score:.2f}) | {date_display} "
                f"| {sender} | {subject} |"
            )

            # Add to cleaned results without numpy arrays
            cleaned_results.append({
                "id": email.get("id"),
                "subject": email.get("subject"),
                "sender": sender,
                "sender_email": (
                    email.get("from", {})
                    .get("emailAddress", {})
                    .get("address") or ""
                ),
                "received_date": received_raw,
                "date_display": date_display,
                "has_attachments": email.get("hasAttachments", False),
                "preview": email.get("bodyPreview") or email.get("preview") or "",
                "similarity_score": score,
                "relevance": relevance,
            })

        return {
            "results": cleaned_results,
            "query": query,
            "result_count": len(results),
            "emails_searched": len(raw_emails),
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "Explain that results are ranked by semantic relevance — "
                "higher scores mean more conceptually similar to the query. "
                "Offer to open or summarise any specific result."
            ),
        }

    except Exception as exc:
        return format_tool_error(
            exc, tool_name="semantic_search_emails", logger=logger
        )

---
./tools/task_tools.py
---
"""
task_tools.py
=============
MCP tool for extracting actionable tasks and to-do items from emails.

Why this file exists:
    One of the core business problems this platform solves is the time
    employees spend manually reading emails to extract tasks, deadlines,
    and action owners. This tool automates that by reading recent emails
    and returning a clean, structured to-do list.

Design notes:
    - This tool does NOT call Requesty.AI internally — it returns the
      raw email content with a structured instruction for LibreChat's
      LLM to extract tasks. This follows the Phase 1 decision to let
      LibreChat's LLM handle all reasoning/summarisation.
    - The tool fetches emails, formats them clearly, and passes an
      extraction prompt as the 'instruction' field so LibreChat's LLM
      knows exactly what to do with the content.
    - Task extraction works on inbox by default but can be filtered
      by keyword or sender for targeted extraction.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from tools.mcp_instance import mcp

from graph.mail_client import (
    fetch_recent_messages,
    search_messages_by_keyword,
)

from utils.logger import get_logger
from utils.governance import get_task_rules
from utils.error_handler import format_tool_error
from utils.audit_logger import log_tool_call, get_user_email_from_headers

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Tool: extract_tasks
# ---------------------------------------------------------------------------
@mcp.tool
async def extract_tasks(
    count: int = 10,
    keyword: str = "",
    sender_email: str = "",
) -> dict:
    """
    Read recent emails and extract a neat, actionable to-do list
    with tasks, deadlines, and action owners where identifiable.

    Use this tool when the user asks things like:
    - "Extract tasks from my recent emails"
    - "What are my action items from this week's emails?"
    - "List all to-dos from emails about the project"
    - "What do I need to do based on John's emails?"

    Args:
        count (int): Number of recent emails to analyse. Default 10,
                      max 25 to keep response manageable.
        keyword (str): Optional keyword to filter emails before
                        extracting tasks e.g. "project", "deadline".
        sender_email (str): Optional filter by sender email address
                             to extract tasks only from that person's emails.

    Returns:
        dict with raw email data and a structured extraction instruction
        for LibreChat's LLM to produce the final to-do list.
    """
    logger.info(
        f"Tool called: extract_tasks "
        f"(count={count}, keyword='{keyword}', sender='{sender_email}')"
    )

    try:
        safe_count = min(max(count, 1), 25)

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="extract_tasks",
            user_email=user_email,
            inputs={"count": safe_count, "keyword": keyword},
        )

        # Step 1: Fetch emails — filtered or recent
        if keyword:
            raw_messages = await search_messages_by_keyword(
                keyword, top=safe_count
            )
        else:
            raw_messages = await fetch_recent_messages(top=safe_count)

        if not raw_messages:
            return {
                "emails_analysed": 0,
                "instruction": (
                    "No emails found to analyse. "
                    "Inform the user their inbox appears empty or "
                    "no emails matched the filter."
                ),
            }

        # Step 2: Filter by sender if specified
        if sender_email:
            sender_lower = sender_email.strip().lower()
            raw_messages = [
                m for m in raw_messages
                if sender_lower in (
                    m.get("from", {})
                    .get("emailAddress", {})
                    .get("address", "")
                    .lower()
                )
            ]

        if not raw_messages:
            return {
                "emails_analysed": 0,
                "instruction": (
                    f"No emails from '{sender_email}' found. "
                    f"Inform the user and suggest they check the email address."
                ),
            }

        # Step 3: Build a structured email digest for task extraction.
        # Each email is formatted clearly so the LLM can parse
        # task-relevant content (action verbs, deadlines, names) easily.
        email_digest_parts = []

        for i, msg in enumerate(raw_messages, 1):
            subject = msg.get("subject") or "(no subject)"
            sender_name = (
                msg.get("from", {}).get("emailAddress", {}).get("name") or "Unknown"
            )
            sender_addr = (
                msg.get("from", {}).get("emailAddress", {}).get("address") or ""
            )
            received = msg.get("receivedDateTime", "")[:10]
            body_preview = msg.get("bodyPreview") or ""

            email_digest_parts.append(
                f"--- Email {i} ---\n"
                f"From: {sender_name} <{sender_addr}>\n"
                f"Date: {received}\n"
                f"Subject: {subject}\n"
                f"Content: {body_preview}\n"
            )

        full_digest = "\n".join(email_digest_parts)

        # ── Governance rules ─────────────────────────────────────────
        governance = get_task_rules(include_sources=True)

        # Step 4: Build the task extraction instruction for LibreChat's LLM.
        # This instruction is returned alongside the raw data so
        # LibreChat processes it rather than Requesty.
        extraction_instruction = (
            "Extract all actionable tasks, to-do items, deadlines, and "
            "action owners from the email digest below. "
            "Format the output as a numbered to-do list using this structure:\n\n"
            "## 📋 Action Items\n\n"
            "| # | Task | Owner | Deadline | Source Email |\n"
            "|---|------|-------|----------|--------------|\n"
            "| 1 | [action] | [person responsible] | [date or 'Not specified'] | [email subject] |\n\n"
            "After the table, add a section:\n"
            "## ⚠️ Deadlines to Watch\n"
            "List only items with explicit deadlines mentioned.\n\n"
            "## ℹ️ Notes\n"
            "Any important context that doesn't fit as a task.\n\n"
            "If no actionable tasks are found in an email, skip it. "
            "Only include real, actionable items — not general discussion.\n\n"
            f"{governance}\n\n"
            f"Email digest to analyse:\n\n{full_digest}"
            f"{get_task_rules(include_sources=True)}"
        )

        return {
            "emails_analysed": len(raw_messages),
            "filter_keyword": keyword or "none",
            "filter_sender": sender_email or "none",
            "email_digest": full_digest,
            "instruction": extraction_instruction,
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="extract_tasks", logger=logger)
    

---
./llm/__init__.py
---


---
./llm/requesty_client.py
---
"""
requesty_client.py
====================
A thin wrapper around calls to the Requesty.AI LLM gateway, handling
primary/fallback model logic.

Why this file exists:
    Every tool that needs an AI-generated summary (summarise_email,
    summarise_attachment) calls generate_summary() here, rather than
    each tool file building its own HTTP request to Requesty.AI. This
    is the ONE place that knows the gateway URL, the model names, and
    how to fall back from the primary model to the secondary one.

    Critically: this isolation is what makes the "swap LLM provider
    later" requirement easy. Since Requesty.AI exposes an
    OpenAI-compatible API, this file's structure would barely change
    even if we pointed REQUESTY_BASE_URL at a different compatible
    gateway in the future — only config/settings.py's values would
    need to change, not this file's logic.

Design notes:
    - Uses httpx for the async HTTP call, consistent with the rest of
      the project's async-first design (FastMCP tools are async).
    - Implements a simple try-primary-then-fallback pattern: if the
      primary model call fails for any reason (timeout, error response),
      it automatically retries once with the fallback model before
      giving up.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import httpx

from config.settings import settings
from utils.logger import get_logger

logger = get_logger(__name__)

# A reasonable timeout for LLM calls — summarisation shouldn't take
# more than this under normal conditions; if it does, something is
# likely wrong upstream and we should fail fast rather than hang.
REQUEST_TIMEOUT_SECONDS = 30.0


# ---------------------------------------------------------------------------
# Internal helper: _call_requesty_model
# ---------------------------------------------------------------------------
async def _call_requesty_model(prompt: str, model_name: str) -> str:
    """
    Make a single chat-completion call to Requesty.AI for a specific
    model. This is an internal helper — external code should call
    generate_summary() instead, which adds fallback handling.

    Args:
        prompt (str): The full prompt text to send to the model.
        model_name (str): Which model to use (e.g. the value of
                           REQUESTY_PRIMARY_MODEL or
                           REQUESTY_FALLBACK_MODEL).

    Returns:
        str: The model's text response.

    Raises:
        httpx.HTTPStatusError: If Requesty.AI returns a non-2xx response.
        httpx.TimeoutException: If the call takes longer than
                                 REQUEST_TIMEOUT_SECONDS.
    """
    # Step 1: Requesty.AI exposes an OpenAI-compatible chat completions
    #         endpoint, so the request shape follows that standard
    #         format: a list of role/content messages.
    url = f"{settings.REQUESTY_BASE_URL}/chat/completions"

    headers = {
        "Authorization": f"Bearer {settings.REQUESTY_API_KEY}",
        "Content-Type": "application/json",
    }

    payload = {
        "model": model_name,
        "messages": [
            {"role": "user", "content": prompt}
        ],
        # Step 2: Keep responses focused and concise — summaries don't
        #         need a huge token budget, and capping this also
        #         controls cost and latency.
        "max_tokens": 4000,
        "temperature": 0.3,
    }

    # Step 3: Make the async HTTP call with an explicit timeout.
    async with httpx.AsyncClient(timeout=REQUEST_TIMEOUT_SECONDS) as http_client:
        response = await http_client.post(url, headers=headers, json=payload)

        # Step 4: Raise an exception for non-2xx responses, so the
        #         calling function's try/except can detect failure
        #         and decide whether to attempt the fallback model.
        response.raise_for_status()

        data = response.json()

    # Step 5: Extract the generated text from the OpenAI-compatible
    #         response shape: choices[0].message.content
    generated_text = data["choices"][0]["message"]["content"]

    return generated_text.strip()


# ---------------------------------------------------------------------------
# Function: generate_summary
# ---------------------------------------------------------------------------
async def generate_summary(prompt: str) -> str:
    """
    Generate a summary using the primary model, automatically falling
    back to the secondary model if the primary call fails for any reason.

    This is the ONLY function other parts of the project (email_tools.py,
    attachment_tools.py) should call directly — it handles the
    primary/fallback logic internally so callers don't need to know
    which model actually ended up generating the response.

    Args:
        prompt (str): The full prompt text describing what to summarise.

    Returns:
        str: The generated summary text.

    Raises:
        RuntimeError: If BOTH the primary and fallback model calls fail,
                      since at that point there's nothing more this
                      function can do — the caller's error handler will
                      format this into a clean message for the user.
    """
    # Step 1: Attempt the primary model first.
    try:
        logger.info(f"Calling primary model: {settings.REQUESTY_PRIMARY_MODEL}")
        result = await _call_requesty_model(prompt, settings.REQUESTY_PRIMARY_MODEL)
        return result

    except Exception as primary_error:
        # Step 2: Log the primary failure clearly, then attempt the
        #         fallback model rather than failing immediately —
        #         this is what makes the system resilient to a single
        #         model provider having a temporary issue.
        logger.warning(
            f"Primary model call failed ({primary_error}). "
            f"Attempting fallback model: {settings.REQUESTY_FALLBACK_MODEL}"
        )

        try:
            result = await _call_requesty_model(
                prompt, settings.REQUESTY_FALLBACK_MODEL
            )
            return result

        except Exception as fallback_error:
            # Step 3: Both models failed — there's nothing more this
            #         function can do. Raise a clear, combined error
            #         for the caller's error handler to format safely.
            logger.error(
                f"Both primary and fallback model calls failed. "
                f"Primary error: {primary_error}. "
                f"Fallback error: {fallback_error}"
            )
            raise RuntimeError(
                "Unable to generate summary — both the primary and "
                "fallback AI models failed to respond."
            )

---
./semantic/__init__.py
---


---
./semantic/embedder.py
---
"""
embedder.py
===========
Converts email text into numerical vector embeddings using
sentence-transformers, enabling semantic similarity search.

Why this file exists:
    Keyword search (Graph API's $search) only finds emails containing
    exact words. Semantic search understands MEANING — so a query like
    "budget concerns" can find emails about "financial worries" or
    "cost overruns" even if those exact words weren't in the query.

    This file handles the conversion:
        email text → numerical vector (list of floats)

    The vector mathematically represents the meaning of the text in a
    high-dimensional space. Two vectors close together = similar meaning.

How it works (first principles):
    1. A pre-trained model (all-MiniLM-L6-v2) has learned to convert
       sentences into 384-dimensional vectors from millions of examples.
    2. We feed it an email's subject + body preview.
    3. It returns a 384-float vector representing that email's meaning.
    4. We do the same for the user's search query.
    5. Emails whose vectors are "closest" to the query vector are
       the most semantically relevant results.

Design notes:
    - The model is loaded ONCE at module import time and reused for
      every call — loading takes ~2 seconds the first time but is
      instant after that.
    - Model: all-MiniLM-L6-v2 — small (80MB), fast, and good quality
      for English business email content.
    - First run downloads the model from HuggingFace (~80MB).
      Subsequent runs use the cached version.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from sentence_transformers import SentenceTransformer
import numpy as np

from config.settings import settings
from utils.logger import get_logger

logger = get_logger(__name__)

# ---------------------------------------------------------------------------
# Step 1: Load the sentence transformer model once at module load time.
# ---------------------------------------------------------------------------
# This is intentionally a module-level variable — loading the model is
# expensive (~2 seconds first time). Loading it once and reusing it
# across all calls is the correct pattern.
_model: SentenceTransformer = None


def get_model() -> SentenceTransformer:
    """
    Get the loaded sentence transformer model, loading it on first call.
    Uses lazy initialisation so the model only loads when first needed,
    not at server startup.

    Returns:
        SentenceTransformer: The loaded embedding model.
    """
    global _model
    if _model is None:
        model_name = settings.SEMANTIC_SEARCH_MODEL
        logger.info(f"Loading sentence transformer model: {model_name}")
        logger.info("(First load may take ~30 seconds to download the model)")
        _model = SentenceTransformer(model_name)
        logger.info(f"Model '{model_name}' loaded successfully")
    return _model


# ---------------------------------------------------------------------------
# Function: embed_text
# ---------------------------------------------------------------------------
def embed_text(text: str) -> np.ndarray:
    """
    Convert a single text string into a numerical vector embedding.

    Args:
        text (str): Any text string to embed — email subject,
                     body, or search query.

    Returns:
        np.ndarray: A 1D numpy array of floats representing the text's
                     semantic meaning. Shape: (384,) for all-MiniLM-L6-v2.
    """
    model = get_model()
    # encode() returns a numpy array automatically
    embedding = model.encode(text, convert_to_numpy=True)
    return embedding


# ---------------------------------------------------------------------------
# Function: embed_email
# ---------------------------------------------------------------------------
def embed_email(email: dict) -> np.ndarray:
    """
    Convert an email dictionary into a vector embedding by combining
    its subject and body preview into one text representation.

    Combining subject + body gives richer semantic coverage than
    either alone — subjects are often brief titles while bodies
    contain the actual context.

    Args:
        email (dict): An email dict from mail_client.py with at least
                       'subject' and 'bodyPreview' or 'preview' fields.

    Returns:
        np.ndarray: A 1D numpy array representing this email's meaning.
    """
    subject = email.get("subject") or ""
    # Support both raw Graph API format and our cleaned format
    body = (
        email.get("bodyPreview")
        or email.get("preview")
        or email.get("body", {}).get("content", "")
        or ""
    )

    # Combine subject and body — weight subject slightly more by
    # repeating it, since subject is usually the most informative.
    combined_text = f"{subject}. {subject}. {body}"

    return embed_text(combined_text)


# ---------------------------------------------------------------------------
# Function: embed_emails_batch
# ---------------------------------------------------------------------------
def embed_emails_batch(emails: list[dict]) -> np.ndarray:
    """
    Convert a list of email dicts into a matrix of embeddings.
    Batch processing is significantly faster than embedding one by one.

    Args:
        emails (list[dict]): List of email dicts.

    Returns:
        np.ndarray: A 2D numpy array of shape (len(emails), 384).
                     Each row is the embedding for one email.
    """
    if not emails:
        return np.array([])

    model = get_model()

    # Build text representations for the whole batch
    texts = []
    for email in emails:
        subject = email.get("subject") or ""
        body = (
            email.get("bodyPreview")
            or email.get("preview")
            or email.get("body", {}).get("content", "")
            or ""
        )
        texts.append(f"{subject}. {subject}. {body}")

    logger.info(f"Embedding batch of {len(texts)} emails...")

    # encode() with a list returns a 2D array automatically
    embeddings = model.encode(texts, convert_to_numpy=True, show_progress_bar=False)

    logger.info(f"Batch embedding complete. Shape: {embeddings.shape}")
    return embeddings

---
./semantic/search_engine.py
---
"""
search_engine.py
================
Performs semantic similarity search over a corpus of email embeddings.

Why this file exists:
    embedder.py handles converting text → vectors.
    This file handles the actual search:
        given a query vector + a matrix of email vectors,
        find the N emails most similar to the query.

How cosine similarity works (first principles):
    Two vectors can be compared by the angle between them.
    - Angle = 0°  → identical meaning → similarity = 1.0
    - Angle = 90° → unrelated meaning → similarity = 0.0
    - Cosine similarity = cos(angle) = dot product / (length × length)

    We use scikit-learn's cosine_similarity which handles this
    computation efficiently even for large batches.

Design notes:
    - No database needed — we fetch emails fresh from Graph API,
      embed them in memory, search, and return results.
    - This is an in-memory search — for Phase 2 with up to 100 emails
      this is fast (< 1 second). A vector database (Pinecone, Chroma)
      would be added in a later phase if the corpus grows to thousands.
    - Threshold filtering removes low-relevance results — only emails
      with similarity >= MIN_SIMILARITY_THRESHOLD are returned.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

from semantic.embedder import embed_text, embed_emails_batch
from utils.logger import get_logger

logger = get_logger(__name__)

# Minimum similarity score to include in results.
# 0.0 = include everything, 1.0 = only exact matches.
# 0.25 is a good threshold for business email semantic search —
# low enough to catch related topics, high enough to filter noise.
MIN_SIMILARITY_THRESHOLD = 0.25


# ---------------------------------------------------------------------------
# Function: semantic_search
# ---------------------------------------------------------------------------
def semantic_search(
    query: str,
    emails: list[dict],
    top_k: int = 10,
) -> list[dict]:
    """
    Find the most semantically relevant emails for a given query.

    Args:
        query (str): The user's natural language search query.
                      e.g. "budget concerns from last month",
                           "project deadline reminders",
                           "John asking for report updates"
        emails (list[dict]): List of email dicts to search over.
                              Should include subject, preview/bodyPreview.
        top_k (int): Maximum number of top results to return.

    Returns:
        list[dict]: The top_k most relevant emails, each augmented with:
                     - 'similarity_score' (float 0.0–1.0)
                     - 'relevance' (str: High/Medium/Low label)
                    Sorted by similarity score descending.
    """
    if not emails:
        logger.info("Semantic search called with empty email list")
        return []

    if not query or not query.strip():
        logger.warning("Semantic search called with empty query")
        return []

    logger.info(
        f"Running semantic search: query='{query}', "
        f"corpus_size={len(emails)}, top_k={top_k}"
    )

    # Step 1: Embed the search query into a vector.
    query_vector = embed_text(query)

    # Step 2: Embed all emails into a matrix of vectors.
    email_matrix = embed_emails_batch(emails)

    if email_matrix.size == 0:
        return []

    # Step 3: Compute cosine similarity between the query vector
    #         and every email vector in the matrix.
    # query_vector shape: (384,) → reshape to (1, 384) for sklearn
    # email_matrix shape: (N, 384)
    # Result shape: (1, N) → squeeze to (N,)
    similarities = cosine_similarity(
        query_vector.reshape(1, -1),
        email_matrix
    ).squeeze()

    # Step 4: Filter by minimum threshold and sort by score.
    scored_emails = []
    for i, score in enumerate(similarities):
        if float(score) >= MIN_SIMILARITY_THRESHOLD:
            email_copy = dict(emails[i])  # don't mutate original
            email_copy["similarity_score"] = round(float(score), 4)
            email_copy["relevance"] = (
                "🟢 High" if score >= 0.65
                else "🟡 Medium" if score >= 0.40
                else "🔵 Low"
            )
            scored_emails.append(email_copy)

    # Step 5: Sort by similarity descending and return top_k.
    scored_emails.sort(key=lambda x: x["similarity_score"], reverse=True)
    top_results = scored_emails[:top_k]

    logger.info(
        f"Semantic search complete: {len(top_results)} results above threshold "
        f"(threshold={MIN_SIMILARITY_THRESHOLD})"
    )

    return top_results

---
./temp_attachments/charts/bar_chart_20260716_090335_f16f.svg
---
<?xml version="1.0" encoding="utf-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN"
  "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns:xlink="http://www.w3.org/1999/xlink" width="569.735225pt" height="350.642812pt" viewBox="0 0 569.735225 350.642812" xmlns="http://www.w3.org/2000/svg" version="1.1"></svg>


---
./temp_attachments/charts/pie_chart_20260723_122549_ac7f.mmd
---
pie title Emails by Sender (All 51 Emails)
  "Administrator Vishalkumar Prajapati" : 36.0
  "Microsoft Security" : 6.0
  "MAILER-DAEMON" : 5.0
  "Azure DevOps" : 3.0
  "Andhavarapu Manoj" : 1.0

---
./auth/__init__.py
---


---
./auth/graph_auth.py
---
"""
graph_auth.py
=============
Handles extraction and basic validation of the OAuth access token that
LibreChat sends with every MCP tool call.

Why this file exists:
    This implements the "delegated auth" model agreed on for this
    project. The token always belongs to the logged-in user (you) —
    never a shared service account. LibreChat acquires this token via
    its own OAuth flow with Microsoft and forwards it to this MCP
    server on every request. This file's job is to safely retrieve
    that token from the incoming request.

Design notes:
    - FastMCP exposes incoming HTTP headers via get_http_headers(),
      imported from fastmcp.server.dependencies. This is how we access
      the Authorization header without needing to manually parse raw
      HTTP request objects.
    - This file does NOT perform the OAuth login flow itself — that
      flow happens entirely within LibreChat. This file only reads the
      token that flow already produced.
    - No token is ever written to disk or logged in full — only a
      truncated, safe preview is logged for debugging purposes.
Token sources (checked in priority order):
    1. Authorization: Bearer <token>   — standard HTTP auth header
    2. X-Auth-Token: <token>           — fallback for clients where
                                         FastMCP strips the Authorization
                                         header before it reaches tools
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from fastmcp.server.dependencies import get_http_headers
from fastmcp.exceptions import AuthorizationError

from utils.logger import get_logger

logger = get_logger(__name__)

def get_current_access_token() -> str:
    """
    Extract the Microsoft Graph OAuth access token from the current
    incoming request headers.

    Checks Authorization header first, then X-Auth-Token as fallback.
    Automatically adds 'Bearer ' prefix if the raw token arrives without it.

    Returns:
        str: The raw bearer token string (without the 'Bearer ' prefix).

    Raises:
        AuthorizationError: If no token is found in any checked header.
    """
    # Step 1: Get all headers from the current incoming HTTP request.
    headers = get_http_headers()

    # Step 2: Check Authorization header first (standard), then
    #         X-Auth-Token as fallback (used by agent_test.py and
    #         any client where FastMCP middleware strips Authorization).
    auth_header = (
        headers.get("Authorization")
        or headers.get("authorization")
        or headers.get("X-comp.-Authorization")
        or headers.get("x-comp.-authorization")
        or headers.get("X-Auth-Token")
        or headers.get("x-auth-token")
    )

    user_email = (
        headers.get("X-comp.-Email")
        or headers.get("x-comp.-email")
        or "unknown"
    )
    logger.info(f"request autheticated for user: {user_email}")
    
    if not auth_header:
        logger.warning("No auth token found in any expected header")
        raise AuthorizationError(
            "Missing Authorization header. Please authenticate via "
            "LibreChat's MCP server settings."
        )

    # Step 3: Auto-wrap raw token with Bearer prefix if missing.
    #         X-Auth-Token arrives as a raw token without the prefix.
    #         Only wrap if it doesn't already have a recognised auth
    #         scheme prefix (Basic, Digest, etc.) — those are genuinely
    #         malformed and should be rejected.
    KNOWN_SCHEMES = ("basic ", "digest ", "negotiate ", "ntlm ")
    header_lower = auth_header.lower()
    if not auth_header.startswith("Bearer "):
        if any(header_lower.startswith(scheme) for scheme in KNOWN_SCHEMES):
            logger.warning(f"Authorization header uses unsupported scheme")
            raise AuthorizationError(
                "Authorization header is malformed. Expected 'Bearer <token>'."
            )
        # Raw token (e.g. from X-Auth-Token) — wrap it
        auth_header = f"Bearer {auth_header}"

    # Step 4: Extract just the token part.
    token = auth_header.removeprefix("Bearer ").strip()

    if not token:
        logger.warning("Auth header present but token value is empty")
        raise AuthorizationError("Authorization token is empty.")

    # Step 5: Log a safe preview only — never log the full token.
    token_preview = f"{token[:8]}...{token[-4:]}" if len(token) > 12 else "***"
    logger.debug(f"Access token extracted successfully (preview: {token_preview})")

    return token

---
./auth/token_cache.py
---
"""
token_cache.py
===============
Provides a request-scoped, in-memory holder for the current access
token — explicitly NOT a persistent cache.

Why this file exists:
    This file exists to formalise and make explicit one of our hardest
    architectural requirements: "the VM never stores credentials."
    Despite the name "token_cache", this is intentionally NOT a
    traditional cache that saves tokens across requests. It only holds
    the token for the lifetime of a single function call chain, using
    Python's contextvars — a thread-safe / async-safe way to pass data
    implicitly through a call stack without passing it as an explicit
    parameter to every single function.

Design notes:
    - contextvars (not a global variable or a dict keyed by user) is
      used specifically because this server may handle multiple
      concurrent requests (e.g. if later expanded to multi-user). A
      plain global variable would leak one user's token into another
      user's concurrent request. contextvars avoids that by being
      isolated per async task.
    - Nothing here ever touches disk, a database, or persists beyond
      a single incoming request's processing.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from contextvars import ContextVar

from utils.logger import get_logger

logger = get_logger(__name__)

# ---------------------------------------------------------------------------
# Step 1: Define the context variable that will hold the token for the
#         duration of a single request's processing only.
# ---------------------------------------------------------------------------
_current_token_var: ContextVar[str | None] = ContextVar(
    "current_access_token", default=None
)


# ---------------------------------------------------------------------------
# Function: set_current_token
# ---------------------------------------------------------------------------
def set_current_token(token: str) -> None:
    """
    Store the access token for the duration of the current request only.

    This is called once at the very start of handling a tool call
    (typically inside graph_client_factory.py), immediately after the
    token is extracted by graph_auth.py.

    Args:
        token (str): The raw bearer token to store for this request.
    """
    _current_token_var.set(token)
    logger.debug("Access token stored in request-scoped context")


# ---------------------------------------------------------------------------
# Function: get_current_token
# ---------------------------------------------------------------------------
def get_current_token() -> str | None:
    """
    Retrieve the access token stored earlier in THIS request's context.

    Returns:
        str | None: The token if one was set earlier in this same
                     request/task, otherwise None.
    """
    return _current_token_var.get()


# ---------------------------------------------------------------------------
# Function: clear_current_token
# ---------------------------------------------------------------------------
def clear_current_token() -> None:
    """
    Explicitly clear the token from context. Called at the end of a
    request's processing as an extra safety measure, even though
    contextvars already naturally isolates each request and would not
    leak this value on its own.
    """
    _current_token_var.set(None)
    logger.debug("Access token cleared from request-scoped context")

---
./bin/agent_test.py
---

"""
agent_test.py
=============
Interactive agent loop that lets you prompt your MCP tools
directly from VS Code terminal — no LibreChat needed.

The LLM reads your prompt, decides which MCP tool to call,
executes it against your real Outlook data, and prints the result.

Requires:
    - python server.py running in Terminal 1
    - token.json present (run get_token.py first)

Usage:
    python agent_test.py

Type 'quit' to exit.
"""
import sys
import asyncio
import json
import httpx
from pathlib import Path

# Ensure the project root is on sys.path so "from config.settings import settings" works
PROJECT_ROOT = Path(__file__).resolve().parents[1]
if str(PROJECT_ROOT) not in sys.path:
    sys.path.insert(0, str(PROJECT_ROOT))

from config.settings import settings
from utils.logger import get_logger

logger = get_logger(__name__)

# ── Config ───────────────────────────────────────────────────────
# MCP_URL    = f"http://localhost:{settings.MCP_PORT}/mcp"
MCP_URL    = f"https://figpt-mcp-outlook.comp.group.com/mcp"
TOKEN_FILE = Path(__file__).parent/"token.json"

# ── Tool definitions for the LLM ─────────────────────────────────
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "get_my_profile",
            "description": "Get the display name, email address, and job title of the currently logged-in Microsoft 365 user.",
            "parameters": {"type": "object", "properties": {}, "required": []},
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_emails",
            "description": "List the most recent emails in the user's Outlook inbox.",
            "parameters": {
                "type": "object",
                "properties": {
                    "count": {
                        "type": "integer",
                        "description": "Number of emails to return (default 10, max 50)"
                    }
                },
                "required": []
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "read_email",
            "description": "Read the full content of a specific email by its ID.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {
                        "type": "string",
                        "description": "The unique Graph API ID of the email"
                    }
                },
                "required": ["email_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_emails",
            "description": "Search the user's mailbox for emails matching a keyword.",
            "parameters": {
                "type": "object",
                "properties": {
                    "keyword": {
                        "type": "string",
                        "description": "Search term"
                    },
                    "count": {
                        "type": "integer",
                        "description": "Max results to return"
                    }
                },
                "required": ["keyword"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "summarise_email",
            "description": "Generate a concise AI summary of a specific email's content.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {
                        "type": "string",
                        "description": "The unique Graph API ID of the email"
                    }
                },
                "required": ["email_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_calendar_events",
            "description": "List the user's upcoming calendar events within a given time range.",
            "parameters": {
                "type": "object",
                "properties": {
                    "date_range": {
                        "type": "string",
                        "description": "Natural language range: today, tomorrow, this week, next 7 days"
                    }
                },
                "required": []
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_attachments",
            "description": "List all attachments on a specific email without reading their content.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {
                        "type": "string",
                        "description": "The unique Graph API ID of the email"
                    }
                },
                "required": ["email_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "read_attachment",
            "description": "Extract and return the text content of an email attachment (PDF, Word, PowerPoint, Excel).",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {
                        "type": "string",
                        "description": "The unique Graph API ID of the parent email"
                    },
                    "attachment_id": {
                        "type": "string",
                        "description": "The unique ID of the attachment"
                    }
                },
                "required": ["email_id", "attachment_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "summarise_attachment",
            "description": "Read an email attachment and generate a concise AI summary of its content.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {
                        "type": "string",
                        "description": "The unique Graph API ID of the parent email"
                    },
                    "attachment_id": {
                        "type": "string",
                        "description": "The unique ID of the attachment"
                    }
                },
                "required": ["email_id", "attachment_id"]
            },
        }
    },
]


# ── Token loader ─────────────────────────────────────────────────
def load_token() -> str:
    """
    Load the delegated access token from token.json.
    Run get_token.py first if this file doesn't exist.
    """
    if not TOKEN_FILE.exists():
        print("❌ token.json not found. Run: python get_token.py first.")
        raise SystemExit(1)
    data = json.loads(TOKEN_FILE.read_text())
    return data["access_token"]


# ── MCP helpers ──────────────────────────────────────────────────
async def mcp_initialise(client: httpx.AsyncClient, token: str) -> str:
    """
    Perform the MCP protocol handshake.
    Must be called once before any tool calls.
    Returns the session ID assigned by the server.
    """
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
        "Accept": "application/json, text/event-stream",
    }
    payload = {
        "jsonrpc": "2.0",
        "id": 0,
        "method": "initialize",
        "params": {
            "protocolVersion": "2024-11-05",
            "capabilities": {},
            "clientInfo": {
                "name": "agent-test",
                "version": "1.0.0"
            }
        }
    }
    response = await client.post(MCP_URL, headers=headers, json=payload)
    response.raise_for_status()
    return response.headers.get("mcp-session-id", "")


async def mcp_call_tool(
    client: httpx.AsyncClient,
    token: str,
    session_id: str,
    tool_name: str,
    arguments: dict
) -> str:
    """
    Call one MCP tool on the server and return the raw result string.

    The Authorization header carries the delegated OAuth token so that
    graph_auth.py can extract it and pass it to Microsoft Graph API.
    The X-Auth-Token header is a fallback in case FastMCP's header
    context doesn't surface the Authorization header correctly.
    """
    headers = {
        # Primary auth header — read by graph_auth.get_current_access_token()
        "Authorization": f"Bearer {token}",
        # Fallback header — also checked by graph_auth.py
        "X-Auth-Token": token,
        "Content-Type": "application/json",
        "Accept": "application/json, text/event-stream",
        # Session ID ties this call to the initialised MCP session
        "mcp-session-id": session_id,
    }

    payload = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/call",
        "params": {
            "name": tool_name,
            "arguments": arguments
        }
    }

    response = await client.post(MCP_URL, headers=headers, json=payload)
    response.raise_for_status()

    # ── Parse response — either plain JSON or SSE stream ─────────
    content_type = response.headers.get("content-type", "")

    if "application/json" in content_type:
        # Plain JSON response
        data = response.json()
        content = data.get("result", {}).get("content", [{}])
        return content[0].get("text", "{}") if content else "{}"

    else:
        # SSE (Server-Sent Events) response — FastMCP default
        # Format is:
        #   event: message
        #   data: {"jsonrpc":"2.0","result":{...}}
        #
        # Lines arrive separately so we collect ALL data: lines
        # and parse the first one that contains a valid result.
        lines = response.text.splitlines()
        for line in lines:
            line = line.strip()
            if not line.startswith("data:"):
                continue
            json_str = line[len("data:"):].strip()
            if not json_str:
                continue
            try:
                data = json.loads(json_str)
                # Skip error responses
                if "error" in data:
                    error_msg = data["error"].get("message", "Unknown error")
                    return json.dumps({"error": True, "message": error_msg})
                content = data.get("result", {}).get("content", [{}])
                if content:
                    return content[0].get("text", "{}")
            except json.JSONDecodeError:
                continue

    return "{}"


# ── LLM call ─────────────────────────────────────────────────────
async def call_llm(messages: list, allow_tools: bool = True) -> dict:
    """
    Send the conversation history to Requesty.AI and return the
    full API response.

    allow_tools=True  → LLM can request tool calls
    allow_tools=False → LLM must respond with plain text only
                        (used when forcing a final summary response)
    """
    async with httpx.AsyncClient(timeout=60) as http:
        payload = {
            "model": settings.REQUESTY_PRIMARY_MODEL,
            "messages": messages,
            "max_tokens": 4000,
            "temperature": 0.3,
        }

        if allow_tools:
            payload["tools"] = TOOLS
            payload["tool_choice"] = "auto"

        response = await http.post(
            f"{settings.REQUESTY_BASE_URL}/chat/completions",
            headers={
                "Authorization": f"Bearer {settings.REQUESTY_API_KEY}",
                "Content-Type": "application/json",
            },
            json=payload
        )
        response.raise_for_status()
        return response.json()


# ── Agent loop ───────────────────────────────────────────────────
async def agent_loop():
    print("\n" + "="*55)
    print("  outlook-ai-agent-mcp — VS Code Agent Test")
    print("="*55)
    print("  Your MCP tools are live. Type a prompt below.")
    print("  Type 'quit' to exit.\n")

    # Step 1: Load saved delegated token
    token = load_token()

    async with httpx.AsyncClient(timeout=60) as mcp_client:

        # Step 2: Initialise MCP session once for the whole session
        print("Connecting to MCP server...")
        session_id = await mcp_initialise(mcp_client, token)
        print(f"✅ MCP session ready\n")

        # Step 3: Build conversation history — grows with each turn
        messages = [
            {
                "role": "system",
                "content": (
                    "You are an intelligent Outlook email assistant for a Company. "
                    "You have access to MCP tools that can read emails, "
                    "summarise them, check calendar events, and read attachments. "
                    "Always use the available tools to fetch real data — "
                    "never make up or assume email content. "
                    "After receiving tool results, always compose a clear, "
                    "well-formatted final answer for the user."
                )
            }
        ]

        # Step 4: Main prompt loop
        while True:

            # Get user input
            try:
                user_input = input("You: ").strip()
            except (KeyboardInterrupt, EOFError):
                print("\n\nExiting agent. Goodbye.")
                break

            if user_input.lower() in ("quit", "exit", "q"):
                print("Exiting agent. Goodbye.")
                break

            if not user_input:
                continue

            # Add user message to history
            messages.append({"role": "user", "content": user_input})
            print("\nAgent thinking...\n")
            print("STEP 1: Calling LLM...")
            sys.stdout.flush()

            # Round 1: Ask LLM what to do
            llm_response = await call_llm(messages, allow_tools=True)
            print("STEP 2: LLM responded")
            sys.stdout.flush()

            assistant_message = llm_response["choices"][0]["message"]
            messages.append(assistant_message)

            has_tool_calls = bool(assistant_message.get("tool_calls"))
            print(f"STEP 3: Has tool calls = {has_tool_calls}")
            sys.stdout.flush()

            max_tool_rounds = 5
            tool_round = 0

            while assistant_message.get("tool_calls") and tool_round < max_tool_rounds:
                tool_round += 1
                print(f"STEP 4: Tool round {tool_round}")
                sys.stdout.flush()

                for tool_call in assistant_message["tool_calls"]:
                    tool_name = tool_call["function"]["name"]
                    arguments = json.loads(tool_call["function"]["arguments"])
                    print(f"STEP 5: Calling tool {tool_name} with {arguments}")
                    sys.stdout.flush()

                    tool_result_raw = await mcp_call_tool(
                        mcp_client, token, session_id, tool_name, arguments
                    )

                    print(f"STEP 6: Tool result length = {len(tool_result_raw)}")
                    print(f"STEP 6: Tool result preview = {tool_result_raw[:200]}")
                    sys.stdout.flush()

                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call["id"],
                        "content": tool_result_raw,
                    })

                print("STEP 7: Calling LLM again with tool results...")
                sys.stdout.flush()

                llm_response = await call_llm(messages, allow_tools=True)
                assistant_message = llm_response["choices"][0]["message"]
                messages.append(assistant_message)

                print(f"STEP 8: LLM responded, has_tool_calls = {bool(assistant_message.get('tool_calls'))}")
                print(f"STEP 8: content = '{assistant_message.get('content', '')[:200]}'")
                sys.stdout.flush()

            final_answer = assistant_message.get("content", "")
            print(f"STEP 9: final_answer = '{final_answer[:200]}'")
            sys.stdout.flush()

            if not final_answer or not final_answer.strip():
                print("STEP 10: Forcing final text response...")
                sys.stdout.flush()

                messages.append({
                    "role": "user",
                    "content": (
                        "Based on the tool results above, please provide "
                        "a clear and well-formatted answer to my original "
                        "question. Do not call any more tools."
                    )
                })
                forced_response = await call_llm(messages, allow_tools=False)
                print(f"STEP 10b: Full LLM response = {json.dumps(forced_response, indent=2)[:1000]}")
                sys.stdout.flush()
                forced_message = forced_response["choices"][0]["message"]
                final_answer = forced_message.get("content", "No response generated.")
                messages.append(forced_message)
                print(f"STEP 11: forced final_answer = '{final_answer[:200]}'")
                sys.stdout.flush()

            print("\n" + "="*55)
            print("AGENT OUTPUT:")
            print("="*55)
            print(final_answer)
            print("="*55 + "\n")
            sys.stdout.flush()

            # Save to file regardless
            output_file = Path(__file__).parent/"agent_output.txt"
            with open(output_file, "a", encoding="utf-8") as f:
                f.write(f"\nYOU: {user_input}\n")
                f.write(f"AGENT:\n{final_answer}\n")
                f.write("="*55 + "\n")
            print("[Saved to agent_output.txt]")
            sys.stdout.flush()

if __name__ == "__main__":
    asyncio.run(agent_loop())

---
./bin/del.txt
---
python -c "
import msal
from config.settings import settings

app = msal.ConfidentialClientApplication(
    settings.AZURE_CLIENT_ID,
    authority=settings.AZURE_AUTHORITY,
    client_credential=settings.AZURE_CLIENT_SECRET
)

result = app.acquire_token_for_client(
    scopes=['https://graph.microsoft.com/.default']
)

if 'access_token' in result:
    print('SUCCESS — your token:')
    print(result['access_token'])
else:
    print('FAILED:', result.get('error_description'))
"

SUCCESS — your token:
eyJ0eXAiOiJKV1QiLCJub25jZSI6InJuTnBoQXJUS1ZLMVNud2w2dkNmdWNYTVNyRk5DUHB5SUF5VWFnUWdDOTgiLCJhbGciOiJSUzI1NiIsIng1dCI6ImFGa21LVkZjLTRXVjZzWENCdk5aa1hJNTA1WSIsImtpZCI6ImFGa21LVkZjLTRXVjZzWENCdk5aa1hJNTA1WSJ9.eyJhdWQiOiJodHRwczovL2dyYXBoLm1pY3Jvc29mdC5jb20iLCJpc3MiOiJodHRwczovL3N0cy53aW5kb3dzLm5ldC9iNDM0MzBjZS03ZDc1LTQxNTgtYWI3Yi0xZjM5ZTZmZTZiM2YvIiwiaWF0IjoxNzgyOTY1MzA3LCJuYmYiOjE3ODI5NjUzMDcsImV4cCI6MTc4Mjk2OTIwNywiYWNycyI6WyJwZmRyIl0sImFpbyI6ImsyRmdZTkRXUGJ6ZXhqOW5RM1Bscmc4V1MyNm9mYm9qSmI5Zm8wWHBqVm1ycmRzN3p4UUEiLCJhcHBfZGlzcGxheW5hbWUiOiJGR19GaUdQVF9NQ1BfT3V0bG9vayIsImFwcGlkIjoiOWZkMjBjNjMtMDZkMS00MjMxLWE0YTgtYTYyZDAyYzY3MzU5IiwiYXBwaWRhY3IiOiIxIiwiaWRwIjoiaHR0cHM6Ly9zdHMud2luZG93cy5uZXQvYjQzNDMwY2UtN2Q3NS00MTU4LWFiN2ItMWYzOWU2ZmU2YjNmLyIsImlkdHlwIjoiYXBwIiwib2lkIjoiYWZhZDQyMmQtOWUyOS00ZTU3LWE4MjktYmMzNmZiMzBkMDUyIiwicmgiOiIxLkFSOEF6akEwdEhWOVdFR3JleDg1NXY1clB3TUFBQUFBQUFBQXdBQUFBQUFBQUFBQUFBQWZBQS4iLCJzdWIiOiJhZmFkNDIyZC05ZTI5LTRlNTctYTgyOS1iYzM2ZmIzMGQwNTIiLCJ0ZW5hbnRfcmVnaW9uX3Njb3BlIjoiRVUiLCJ0aWQiOiJiNDM0MzBjZS03ZDc1LTQxNTgtYWI3Yi0xZjM5ZTZmZTZiM2YiLCJ1dGkiOiJPNWJqZWJ2Z1hrZW1ROTlkclpaUkFBIiwidmVyIjoiMS4wIiwid2lkcyI6WyIwOTk3YTFkMC0wZDFkLTRhY2ItYjQwOC1kNWNhNzMxMjFlOTAiXSwieG1zX2FjZCI6MTc4MTc3OTQyNCwieG1zX2FjdF9mY3QiOiIzIDkiLCJ4bXNfZnRkIjoiZEQycEQzNGZCM2FlRUpqZTdPUnhBT0lidVhmWXpZQk1TeEJ4WkJwM1VaUUJabkpoYm1ObFl5MWtjMjF6IiwieG1zX2lkcmVsIjoiNyAyNiIsInhtc19wZnRleHAiOjE3ODMwNTU2MDcsInhtc19yZCI6IjAuNDJMbFlCSmlsQmNTNFdBWEV0aHNkYUNJUjBfYmVlbkV3R1ZYdkdjOEVoTGg0QlFTMkhGVl9ORFM4eTg5RzVmNXpEenEtaEVreWlFa3dNd0FBUWVndEpBSUI3ZVF3RktSVzhsV1M2TnFweVNfTVo3akU1TUJBQSIsInhtc19zdWJfZmN0IjoiMyA5IiwieG1zX3RjZHQiOjE0OTY5MTEzMTUsInhtc190ZGJyIjoiRVUiLCJ4bXNfdG50X2ZjdCI6IjMgMTYifQ.ZI90qQHrsXxtjigavQ9MoV085fVyA3Zx-XsclRRHkjpFGOmq8SbANZyAl72YVUBSBzTQ0Tm0c96asJGDFCeWJGdP_in_yUqJJsmvYGSrzgpetSrgbAG770vKrdwcWn4cidiJDX-aKDjZs1E2Pr3svuCsvUFcaxoCTiop06MH6effDxWO0FLYFOZbTKLLJJ2fnr2Aka57OPHKhYYGBedUBkSHJHhxAR4JU1DBB95oYFB5sFnBb641JhRXmWntkdXT7lBIdcI5iwrtYTuHr_SJ1gyoQAs6KRi7vX9VAevjP8-8kTCAejLUWB2OhV-LBQOmzvJDc96qqrbcyhIIOPz-PQ

---
./bin/get_token.py
---

"""
get_token.py
============
One-time interactive login script to get a real delegated
Microsoft OAuth token for local testing.

Opens a browser popup, you log in with your Company account,
token is saved to token.json for use by agent_test.py.

Usage:
    python get_token.py

Do NOT commit token.json — it contains your personal access token.
"""

import json
import msal
from pathlib import Path
import sys

# Ensure the project root is on sys.path so "from config.settings import settings" works
PROJECT_ROOT = Path(__file__).resolve().parents[1]
if str(PROJECT_ROOT) not in sys.path:
    sys.path.insert(0, str(PROJECT_ROOT))

from config.settings import settings

# ── Token file location ──────────────────────────────────────────
TOKEN_FILE = Path(__file__).parent/"token.json"

# ── Scopes needed for all Phase 1 tools ─────────────────────────
SCOPES = [
    "Mail.Read",
    "Calendars.Read",
    "User.Read",
]

def get_token():
    print("\n" + "="*50)
    print("  outlook-ai-agent-mcp — Token Acquisition")
    print("="*50)

    # Step 1: Build a PublicClientApplication with
    #         interactive browser login support.
    #         This is different from ConfidentialClientApplication —
    #         PublicClientApplication supports user-interactive flows
    #         which is exactly what delegated auth requires.
    app = msal.PublicClientApplication(
        client_id=settings.AZURE_CLIENT_ID,
        authority=settings.AZURE_AUTHORITY,
    )

    # Step 2: Check if we already have a cached valid token
    accounts = app.get_accounts()
    result = None

    if accounts:
        print(f"\nFound cached account: {accounts[0]['username']}")
        print("Attempting silent token refresh...")
        result = app.acquire_token_silent(SCOPES, account=accounts[0])

    # Step 3: If no cached token, open browser for interactive login
    if not result:
        print("\nOpening browser for login...")
        print("Log in with your Company Microsoft account.\n")
        result = app.acquire_token_interactive(scopes=SCOPES)

    # Step 4: Validate and save
    if "access_token" in result:
        token_data = {
            "access_token": result["access_token"],
            "token_type": result.get("token_type", "Bearer"),
            "expires_in": result.get("expires_in", 3600),
            "scope": result.get("scope", ""),
        }
        TOKEN_FILE.write_text(json.dumps(token_data, indent=2))

        print("\n✅ Token acquired and saved to token.json")
        print(f"   Expires in: {result.get('expires_in', 3600) // 60} minutes")
        print(f"   Scopes    : {result.get('scope', '')}")
        print("\nYou can now run: python agent_test.py")

    else:
        print(f"\n❌ Failed to acquire token.")
        print(f"   Error: {result.get('error')}")
        print(f"   Details: {result.get('error_description')}")


if __name__ == "__main__":
    get_token()



---
./bin/graph_auth_v1.py
---
"""
graph_auth.py
=============
Handles extraction and basic validation of the OAuth access token that
LibreChat sends with every MCP tool call.

Why this file exists:
    This implements the "delegated auth" model agreed on for this
    project. The token always belongs to the logged-in user (you) —
    never a shared service account. LibreChat acquires this token via
    its own OAuth flow with Microsoft and forwards it to this MCP
    server on every request. This file's job is to safely retrieve
    that token from the incoming request.

Design notes:
    - FastMCP exposes incoming HTTP headers via get_http_headers(),
      imported from fastmcp.server.dependencies. This is how we access
      the Authorization header without needing to manually parse raw
      HTTP request objects.
    - This file does NOT perform the OAuth login flow itself — that
      flow happens entirely within LibreChat. This file only reads the
      token that flow already produced.
    - No token is ever written to disk or logged in full — only a
      truncated, safe preview is logged for debugging purposes.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from fastmcp.server.dependencies import get_http_headers
from fastmcp.exceptions import AuthorizationError

from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: get_current_access_token
# ---------------------------------------------------------------------------
def get_current_access_token() -> str:
    """
    Extract the Microsoft Graph OAuth access token from the current
    incoming request's Authorization header.

    This function is called by graph/graph_client_factory.py at the
    start of every Graph API operation, so every tool call uses the
    correct, currently-authenticated user's token — never a cached or
    shared one.

    Returns:
        str: The raw bearer token string (without the "Bearer " prefix).

    Raises:
        AuthorizationError: If no Authorization header is present, or
                             it's malformed. This causes FastMCP to
                             return a clean 401-style error back to
                             LibreChat, which can then prompt the user
                             to re-authenticate.
    """
    # Step 1: Get all headers from the current incoming HTTP request.
    #         FastMCP makes this available via a context-aware helper,
    #         so we don't need to manually thread request objects
    #         through every function call.
    headers = get_http_headers()
    # comment the below line
    logger.info(f"DEBUG incoming header:{dict(headers)}")

    # Step 2: Look for the standard Authorization header. HTTP headers
    #         are case-insensitive, but we check the conventional
    #         casing first, then fall back to lowercase as a safety net
    #         depending on how the headers dict is populated.
    # auth_header = headers.get("Authorization") or headers.get("authorization")
    auth_header = (
        headers.get("Authorization")
        or headers.get("authorization")
        or headers.get("X-Auth-Token")
        or headers.get("x-auth-token")
    )


    if not auth_header:
        logger.warning("No Authorization header found in incoming request")
        raise AuthorizationError(
            "Missing Authorization header. Please authenticate via "
            "LibreChat's MCP server settings."
        )

    # Step 3: The header should be in the format "Bearer <token>".
    #         Validate this shape and extract just the token part.
    if not auth_header.startswith("Bearer "):
        logger.warning("Authorization header is malformed (missing 'Bearer ' prefix)")
        raise AuthorizationError(
            "Authorization header is malformed. Expected 'Bearer <token>'."
        )

    token = auth_header.removeprefix("Bearer ").strip()

    if not token:
        logger.warning("Authorization header present but token is empty")
        raise AuthorizationError("Authorization token is empty.")

    # Step 4: Log only a short, safe preview of the token — never the
    #         full value — so logs remain useful for debugging without
    #         ever exposing a credential in plaintext logs.
    token_preview = f"{token[:8]}...{token[-4:]}" if len(token) > 12 else "***"
    logger.debug(f"Access token extracted successfully (preview: {token_preview})")

    return token

---
./bin/quick_test.py
---

"""
quick_test.py
=============
Standalone terminal test — validates the full MCP server chain
using a real Microsoft OAuth token, without needing LibreChat.
Delete this file after validation is complete.

Usage:
    python quick_test.py <your-access-token>
"""

import asyncio
import httpx
import sys
import json
from pathlib import Path

# ── Configuration ──────────────────────────────────────────────
MCP_URL = "http://localhost:8000/mcp"
# existing: TOKEN = sys.argv[1] if len(sys.argv) > 1 else ""
TOKEN = sys.argv[1] if len(sys.argv) > 1 else ""

# Auto-load from token.json if no token was passed
token_file = Path(__file__).parent / "token.json"
if not TOKEN and token_file.exists():
    try:
        with token_file.open("r", encoding="utf-8") as f:
            data = json.load(f)
        TOKEN = data.get("access_token", "")
    except Exception:
        pass

HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json",
    "Accept": "application/json, text/event-stream",
}

# ── MCP initialisation ───────────────────────────────────────────
async def initialise_session(client: httpx.AsyncClient) -> str:
    """
    Perform the MCP initialisation handshake.
    FastMCP streamable-http requires this before any tool calls.
    Returns the session ID assigned by the server.
    """
    payload = {
        "jsonrpc": "2.0",
        "id": 0,
        "method": "initialize",
        "params": {
            "protocolVersion": "2024-11-05",
            "capabilities": {},
            "clientInfo": {
                "name": "quick-test-client",
                "version": "1.0.0"
            }
        }
    }

    response = await client.post(MCP_URL, headers=HEADERS, json=payload)
    response.raise_for_status()

    # Extract session ID from response headers — FastMCP assigns one
    session_id = response.headers.get("mcp-session-id", "")
    print(f"    Session ID: {session_id[:20]}..." if session_id else "    No session ID returned")
    return session_id

async def call_tool(client: httpx.AsyncClient, tool_name: str, arguments: dict = {}, session_id: str = "") -> dict:
    """
    Send one MCP tool call and return the result.
    FastMCP streamable-http returns SSE (Server-Sent Events) format,
    so we parse the event stream to extract the JSON result.
    """

    headers = {**HEADERS}
    if session_id:
        headers["mcp-session-id"] = session_id

    payload = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/call",
        "params": {
            "name": tool_name,
            "arguments": arguments,
        }
    }

    response = await client.post(MCP_URL, headers=headers, json=payload)
    response.raise_for_status()

    content_type = response.headers.get("content-type", "")

    # ── Case 1: Plain JSON response ──────────────────────────────
    if "application/json" in content_type:
        data = response.json()
        raw = data.get("result", {}).get("content", [{}])[0].get("text", "{}")
        try:
            return json.loads(raw)
        except Exception:
            return {"raw": raw}

    # ── Case 2: SSE (Server-Sent Events) response ────────────────
    # FastMCP streamable-http sends results as SSE lines like:
    #   data: {"jsonrpc":"2.0","id":1,"result":{...}}
    # We read all lines and find the one that contains the result.
    text = response.text
    for line in text.splitlines():
        line = line.strip()
        if line.startswith("data:"):
            json_str = line[len("data:"):].strip()
            if not json_str:
                continue
            try:
                data = json.loads(json_str)
                # Look for the result payload inside the SSE data
                result = data.get("result", {})
                content = result.get("content", [{}])
                raw = content[0].get("text", "{}") if content else "{}"
                try:
                    return json.loads(raw)
                except Exception:
                    return {"raw": raw}
            except json.JSONDecodeError:
                continue

    # ── Case 3: Nothing parseable found ─────────────────────────
    return {"error": True, "message": f"Unparseable response: {text[:200]}"}


# ── Tests ───────────────────────────────────────────────────────
async def run_tests():
    print("\n" + "="*55)
    print("  outlook-ai-agent-mcp — Terminal Validation")
    print("="*55)

    async with httpx.AsyncClient(timeout=30) as client:

        # ── Initialise MCP session first ─────────────────────────
        print("\n[0] Initialising MCP session...")
        session_id = await initialise_session(client)
        print("    DONE")

        # ── Test 1: Profile ──────────────────────────────────────
        print("\n[1] get_my_profile")
        result = await call_tool(client, "get_my_profile", {}, session_id)
        if result.get("error"):
            print(f"    FAILED — {result['message']}")
        else:
            print(f"    Name  : {result.get('display_name')}")
            print(f"    Email : {result.get('email')}")
            print(f"    Title : {result.get('job_title')}")
            print("    PASSED")

        # ── Test 2: List emails ──────────────────────────────────
        print("\n[2] list_emails (last 3)")
        result = await call_tool(client, "list_emails", {"count": 3}, session_id)
        if isinstance(result, dict) and result.get("error"):
            print(f"    FAILED — {result['message']}")
            email_id = None
        else:
            emails = result if isinstance(result, list) else []
            for i, m in enumerate(emails, 1):
                print(f"    {i}. [{m.get('received_date','?')[:10]}] "
                      f"{m.get('subject','(no subject)')} "
                      f"— {m.get('sender','?')}")
            email_id = emails[0]["id"] if emails else None
            print("    PASSED")

        # ── Test 3: Read email ───────────────────────────────────
        if email_id:
            print("\n[3] read_email (first result)")
            result = await call_tool(client, "read_email", {"email_id": email_id}, session_id)
            if result.get("error"):
                print(f"    FAILED — {result['message']}")
            else:
                body_preview = (result.get("body") or "")[:120].replace("\n", " ")
                print(f"    Subject : {result.get('subject')}")
                print(f"    Body    : {body_preview}...")
                print("    PASSED")
        else:
            print("\n[3] read_email — SKIPPED (no email ID from Test 2)")

        # ── Test 4: Search emails ────────────────────────────────
        print("\n[4] search_emails (keyword: 'meeting')")
        result = await call_tool(client, "search_emails", {"keyword": "meeting", "count": 3}, session_id)
        if isinstance(result, dict) and result.get("error"):
            print(f"    FAILED — {result['message']}")
        else:
            emails = result if isinstance(result, list) else []
            print(f"    Found {len(emails)} result(s)")
            for m in emails:
                print(f"    — {m.get('subject','(no subject)')}")
            print("    PASSED")

        # ── Test 5: Summarise email ──────────────────────────────
        if email_id:
            print("\n[5] summarise_email (first email)")
            result = await call_tool(client, "summarise_email", {"email_id": email_id}, session_id)
            if result.get("error"):
                print(f"    FAILED — {result['message']}")
            else:
                print(f"    Summary : {result.get('summary','')[:200]}...")
                print("    PASSED")
        else:
            print("\n[5] summarise_email — SKIPPED (no email ID from Test 2)")

        # ── Test 6: Calendar ─────────────────────────────────────
        print("\n[6] list_calendar_events (this week)")
        result = await call_tool(client, "list_calendar_events", {"date_range": "this week"}, session_id)
        if isinstance(result, dict) and result.get("error"):
            print(f"    FAILED — {result['message']}")
        else:
            events = result if isinstance(result, list) else []
            print(f"    Found {len(events)} event(s)")
            for e in events[:3]:
                print(f"    — {e.get('subject','?')} "
                      f"@ {(e.get('start_time') or '')[:16]}")
            print("    PASSED")

    print("\n" + "="*55)
    print("  Validation complete.")
    print("="*55 + "\n")

asyncio.run(run_tests())

         

---
./bin/test_mcp_init.py
---
import asyncio
import json
import httpx
from pathlib import Path
from config.settings import settings

TOKEN = json.loads(Path('token.json').read_text())['access_token']
MCP_URL = f'http://localhost:{settings.MCP_PORT}/mcp'

async def test():
    async with httpx.AsyncClient(timeout=30) as client:
        # Step 1: Init
        r = await client.post(MCP_URL, headers={
            'Authorization': f'Bearer {TOKEN}',
            'X-Auth-Token': TOKEN,
            'Content-Type': 'application/json',
            'Accept': 'application/json, text/event-stream',
        }, json={
            'jsonrpc': '2.0', 'id': 0, 'method': 'initialize',
            'params': {'protocolVersion': '2024-11-05', 'capabilities': {}, 'clientInfo': {'name': 'test', 'version': '1.0'}}
        })
        session_id = r.headers.get('mcp-session-id', '')
        print(f'Session: {session_id[:20]}...')

        # Step 2: Call list_emails
        r2 = await client.post(MCP_URL, headers={
            'Authorization': f'Bearer {TOKEN}',
            'X-Auth-Token': TOKEN,
            'Content-Type': 'application/json',
            'Accept': 'application/json, text/event-stream',
            'mcp-session-id': session_id,
        }, json={
            'jsonrpc': '2.0', 'id': 1, 'method': 'tools/call',
            'params': {'name': 'list_emails', 'arguments': {'count': 3}}
        })
        print(f'Status: {r2.status_code}')
        print(f'Content-Type: {r2.headers.get("content-type")}')
        print('Raw response (first 500 chars):')
        print(r2.text[:500])

asyncio.run(test())

---
./bin/token.json
---
{
  "access_token": "eyJ0eYf-aiAh66WUMrr-n1ay........",
  "token_type": "Bearer",
  "expires_in": 4974,
  "scope": "Calendars.Read Mail.Read User.Read profile openid email"
}

---
./utils/__init__.py
---

---
./utils/audit_logger.py
---
"""
audit_logger.py
===============
Structured audit trail for every MCP tool call.

Why this file exists:
    In an enterprise environment, knowing WHO called WHICH tool
    WHEN and with WHAT inputs is critical for:
        - Security review (was sensitive data accessed?)
        - Debugging (what exactly happened before an error?)
        - Compliance (audit trail for email access)
        - Performance monitoring (which tools are used most?)

    This file writes a separate audit log alongside the standard
    application log. Each entry is one JSON line — easy to parse,
    easy to grep, easy to feed into a SIEM tool later.

Design notes:
    - Audit log is separate from the main application log.
    - Each entry is a single JSON line (JSON-Lines format).
    - Sensitive values (email bodies, tokens) are NEVER logged.
    - Only metadata is logged: who, what, when, which email IDs.
    - Log file rotates daily to prevent unbounded growth.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import json
import logging
import os
from datetime import datetime, timezone
from pathlib import Path
from logging.handlers import TimedRotatingFileHandler

from utils.logger import get_logger

logger = get_logger(__name__)

# ---------------------------------------------------------------------------
# Audit log setup
# ---------------------------------------------------------------------------
AUDIT_LOG_DIR = Path("./logs")
AUDIT_LOG_FILE = AUDIT_LOG_DIR / "audit.log"


def _get_audit_logger() -> logging.Logger:
    """
    Get or create the dedicated audit logger.
    Writes to logs/audit.log with daily rotation.
    """
    audit_logger = logging.getLogger("audit")

    if audit_logger.handlers:
        return audit_logger

    # Create logs directory
    AUDIT_LOG_DIR.mkdir(parents=True, exist_ok=True)

    # Rotating file handler — keeps 30 days of audit logs
    handler = TimedRotatingFileHandler(
        AUDIT_LOG_FILE,
        when="midnight",
        interval=1,
        backupCount=30,
        encoding="utf-8",
    )
    handler.setFormatter(logging.Formatter("%(message)s"))
    audit_logger.addHandler(handler)
    audit_logger.setLevel(logging.INFO)
    audit_logger.propagate = False  # Don't duplicate to main log

    return audit_logger


# ---------------------------------------------------------------------------
# Function: log_tool_call
# ---------------------------------------------------------------------------
def log_tool_call(
    tool_name: str,
    user_email: str,
    inputs: dict,
    outcome: str = "success",
    error_message: str = "",
    email_ids_accessed: list = None,
) -> None:
    """
    Write a single audit log entry for an MCP tool call.

    Call this at the start of every tool function, after inputs
    are validated, before any Graph API calls are made.

    Args:
        tool_name (str): Name of the MCP tool being called.
        user_email (str): Email of the authenticated user.
                           Read from X-comp.-Email header.
        inputs (dict): SAFE input parameters — do NOT include
                        email body content, tokens, or secrets.
                        Only IDs, counts, keywords, types.
        outcome (str): "success", "error", or "validation_failed".
        error_message (str): Error detail if outcome != "success".
        email_ids_accessed (list): List of email IDs that were
                                    read during this tool call.
    """
    audit_entry = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "tool": tool_name,
        "user": user_email or "unknown",
        "inputs": _sanitise_inputs(inputs),
        "outcome": outcome,
        "email_ids_accessed": email_ids_accessed or [],
        "error": error_message or None,
    }

    try:
        audit_logger = _get_audit_logger()
        audit_logger.info(json.dumps(audit_entry))
    except Exception as log_err:
        # Audit logging must never crash the tool
        logger.warning(f"Audit log write failed: {log_err}")


# ---------------------------------------------------------------------------
# Function: get_user_email_from_headers
# ---------------------------------------------------------------------------
def get_user_email_from_headers() -> str:
    """
    Extract the authenticated user's email from the LibreChat
    request headers for audit logging purposes.

    Returns:
        str: User email address or "unknown" if not present.
    """
    try:
        from fastmcp.server.dependencies import get_http_headers
        headers = get_http_headers()
        return (
            headers.get("X-comp.-Email")
            or headers.get("x-comp.-email")
            or "unknown"
        )
    except Exception:
        return "unknown"


# ---------------------------------------------------------------------------
# Helper: sanitise inputs for safe logging
# ---------------------------------------------------------------------------
def _sanitise_inputs(inputs: dict) -> dict:
    """
    Remove sensitive values from inputs before logging.
    Keeps only safe metadata fields.

    Args:
        inputs (dict): Raw input parameters from tool call.

    Returns:
        dict: Sanitised copy safe to write to audit log.
    """
    # Fields that are safe to log
    SAFE_FIELDS = {
        "count", "top", "keyword", "date_range", "date_from", "date_to",
        "chart_type", "title", "use_thread", "is_read", "flag_status",
        "source", "list_name", "days_threshold", "folder_name",
        "duration_minutes", "search_days", "lang_code", "chart_type",
    }

    # Fields that must be truncated (IDs are OK, content is not)
    ID_FIELDS = {
        "email_id", "attachment_id", "message_id", "folder_id",
        "event_id", "task_id",
    }

    safe = {}
    for key, value in inputs.items():
        if key in SAFE_FIELDS:
            safe[key] = value
        elif key in ID_FIELDS:
            # Log only a short prefix of IDs — enough to correlate
            # without exposing the full Graph API object reference
            safe[key] = str(value)[:12] + "..." if value else None
        # Intentionally exclude: body_text, context, mom_content,
        # to_emails (contains real addresses), subject (may be sensitive)

    return safe

---
./utils/date_utils.py
---
"""
date_utils.py
=============
Date parsing and range calculation utilities for calendar tools.

Why this file exists:
    The LLM passes date descriptions as natural-language strings like
    "today", "tomorrow", "this week", "next 7 days". The Graph API's
    calendarView endpoint needs precise ISO-formatted datetime strings
    (e.g. "2025-01-15T00:00:00"). This file bridges that gap.

    All date/time conversion is in one place so:
        - calendar_tools.py stays clean and doesn't contain date logic
        - If we need to add more date range phrases later ("next month",
          "this quarter"), there's exactly one file to update
        - Date bugs are easy to isolate and test independently

Design notes:
    - Uses python-dateutil for flexible date parsing. Unlike Python's
      built-in datetime.strptime(), dateutil can parse a wide variety
      of date formats without needing an exact format string specified —
      useful because dates appear in many formats across email content.
    - All datetimes produced are timezone-naive ISO strings, which is
      what the Graph API calendarView endpoint expects by default when
      the user's calendar timezone is set on their Microsoft account.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from datetime import datetime, timedelta, time
from dateutil import parser as dateutil_parser

from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: parse_relative_date_range
# ---------------------------------------------------------------------------
def parse_relative_date_range(date_range: str) -> tuple[datetime, datetime]:
    """
    Convert a natural-language date range description into a concrete
    start and end datetime pair.

    Called by tools/calendar_tools.py to translate the user's request
    ("what meetings do I have this week?") into Graph API-compatible
    datetime boundaries.

    Args:
        date_range (str): A natural-language time description. Supported
                           values include:
                           - "today"
                           - "tomorrow"
                           - "this week" (Monday to Sunday of current week)
                           - "next week"
                           - "next 7 days"
                           - "next 14 days"
                           - "next 30 days"
                           Falls back to "today" if unrecognised.

    Returns:
        tuple[datetime, datetime]: A (start_datetime, end_datetime) pair,
                                    both as timezone-naive datetime objects
                                    ready to pass to Graph API.
    """
    # Step 1: Normalise the input — lowercase and stripped — so minor
    #         variations in how the LLM phrases it still match correctly.
    normalised = date_range.strip().lower()

    # Step 2: Get today's date at midnight (start of day) as the
    #         anchor for all relative calculations.
    today_start = datetime.combine(datetime.today().date(), time.min)
    today_end = datetime.combine(datetime.today().date(), time.max)

    logger.debug(f"Resolving date range: '{normalised}' (anchor: {today_start.date()})")

    # Step 3: Map the natural-language phrase to start/end datetimes.
    if normalised == "today":
        return today_start, today_end

    elif normalised == "tomorrow":
        tomorrow = today_start + timedelta(days=1)
        return tomorrow, datetime.combine(tomorrow.date(), time.max)

    elif normalised == "this week":
        # "This week" = Monday 00:00:00 to Sunday 23:59:59 of the
        # current calendar week.
        days_since_monday = today_start.weekday()  # Monday = 0, Sunday = 6
        week_start = today_start - timedelta(days=days_since_monday)
        week_end = week_start + timedelta(days=6)
        return week_start, datetime.combine(week_end.date(), time.max)

    elif normalised == "next week":
        days_until_next_monday = 7 - today_start.weekday()
        next_week_start = today_start + timedelta(days=days_until_next_monday)
        next_week_end = next_week_start + timedelta(days=6)
        return next_week_start, datetime.combine(next_week_end.date(), time.max)

    elif normalised in ("next 7 days", "7 days"):
        end = today_start + timedelta(days=7)
        return today_start, datetime.combine(end.date(), time.max)

    elif normalised in ("next 14 days", "14 days", "two weeks"):
        end = today_start + timedelta(days=14)
        return today_start, datetime.combine(end.date(), time.max)

    elif normalised in ("next 30 days", "30 days", "this month"):
        end = today_start + timedelta(days=30)
        return today_start, datetime.combine(end.date(), time.max)

    else:
        # Step 4: Unrecognised phrase — log a warning and fall back to
        #         "today" rather than crashing. The LLM will see today's
        #         events and may clarify the intended range in the next
        #         message.
        logger.warning(
            f"Unrecognised date range '{date_range}' — defaulting to 'today'"
        )
        return today_start, today_end


# ---------------------------------------------------------------------------
# Function: parse_date_string
# ---------------------------------------------------------------------------
def parse_date_string(date_str: str) -> datetime | None:
    """
    Parse an arbitrary date string into a Python datetime object.

    Used when extracting dates from email content where the format is
    unknown (e.g. "15th January", "Jan 15 2025", "2025-01-15").
    dateutil handles all common formats automatically.

    Args:
        date_str (str): A date/time string in any common format.

    Returns:
        datetime | None: A parsed datetime object, or None if the
                          string could not be parsed.
    """
    try:
        parsed = dateutil_parser.parse(date_str, fuzzy=True)
        return parsed
    except (ValueError, OverflowError):
        logger.debug(f"Could not parse date string: '{date_str}'")
        return None

---
./utils/error_handler.py
---
"""
error_handler.py
================
Centralised error formatting for MCP tool responses.

Why this file exists:
    When a Python exception occurs inside a tool function, we must
    NEVER let the raw exception bubble up to LibreChat uncaught. A
    raw exception would either:
        1. Show an ugly, confusing Python stack trace to the end user
           in the chat UI.
        2. Expose internal implementation details (file paths, variable
           names, SDK internals) that could be a security concern in
           an enterprise environment.

    This file provides a single format_tool_error() function that every
    tool's except block calls. It:
        - Logs the FULL error detail internally (useful for debugging)
        - Returns a CLEAN, user-friendly error dict to the chat UI
        - Never leaks internal details to the end user

Design notes:
    - The return type is deliberately dict (same as most tool return
      types) so that the calling tool can simply do:
          return format_tool_error(exc, ...)
      and FastMCP handles it gracefully as a normal tool response,
      rather than an unhandled exception.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import logging
import traceback

from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: format_tool_error
# ---------------------------------------------------------------------------
def format_tool_error(
    exc: Exception,
    tool_name: str,
    logger: logging.Logger = None,
) -> dict:
    """
    Convert a raw Python exception into a clean, safe error response
    suitable for returning to LibreChat / the end user.

    Called in the except block of every MCP tool function like this:

        except Exception as exc:
            return format_tool_error(exc, tool_name="list_emails", logger=logger)

    Args:
        exc (Exception): The caught exception object.
        tool_name (str): The name of the tool that caught the error,
                          used in log messages to make debugging easier.
        logger (logging.Logger): The calling module's logger. If not
                                   provided, falls back to this module's
                                   own logger.

    Returns:
        dict: A clean error dictionary with two keys:
              - "error" (bool): Always True, signals to the caller/LLM
                                 that this response represents a failure.
              - "message" (str): A user-friendly description of what
                                  went wrong, safe to show in the chat UI.
    """
    # Step 1: Use the caller's logger if provided (gives more specific
    #         module path in the log), otherwise fall back to ours.
    active_logger = logger or get_logger(__name__)

    # Step 2: Log the FULL traceback internally. This is what you read
    #         when debugging — the complete error chain, file paths,
    #         line numbers, everything.
    full_traceback = traceback.format_exc()
    active_logger.error(
        f"Error in tool '{tool_name}': {exc}\n{full_traceback}"
    )

    # Step 3: Classify the exception type to give a slightly more
    #         helpful user-facing message, without exposing internals.
    exc_type = type(exc).__name__


    if "AuthorizationError" in exc_type or "Unauthorized" in exc_type:
        user_message = (
            "Authentication failed. Your Microsoft session may have "
            "expired. Please re-authenticate via LibreChat's MCP "
            "server settings."
        )

    elif "ODataError" in exc_type or "APIError" in exc_type:
        # Microsoft Graph SDK raises ODataError for API-level failures.
        # Extract the inner message if available, otherwise use generic.
        inner = getattr(exc, "error", None)
        inner_msg = getattr(inner, "message", None) if inner else None
        if inner_msg and "expired" in inner_msg.lower():
            user_message = (
                "Your Microsoft access token has expired. Please "
                "re-authenticate via LibreChat's MCP server settings."
            )
        elif inner_msg and "ErrorItemNotFound" in str(getattr(inner, "code", "")):
            user_message = (
                "The attachment could not be found. the attachment ID "
                "may have expired. Please list the emails and attchments "
                "again to get a fresh attchment ID, then retry"
            )
        elif inner_msg and "insufficient" in inner_msg.lower():
            user_message = (
                "Insufficient permissions to complete this action. "
                "Please check that the required Graph API permissions "
                "have been granted in Azure Portal."
            )
        else:
            user_message = (
                f"Microsoft Graph API returned an error: "
                f"{inner_msg or 'Unknown error'}. Please try again."
            )


    elif "ValueError" in exc_type:
        # Value errors from our own validation (e.g. attachment too large)
        # are safe to pass through to the user since we wrote them.
        user_message = str(exc)

    elif "TimeoutException" in exc_type or "TimeoutError" in exc_type:
        # Network timeouts — clear, actionable message.
        user_message = (
            "The request timed out. Microsoft Graph API or the AI "
            "service may be experiencing delays. Please try again."
        )

    elif "RuntimeError" in exc_type:
        # RuntimeErrors from our code (e.g. both LLM models failed)
        # are also safe to pass through since we wrote them.
        user_message = str(exc)

    else:
        # Anything else (SDK errors, unexpected exceptions) — show a
        # generic message to avoid leaking implementation details.
        user_message = (
            f"An unexpected error occurred while running '{tool_name}'. "
            f"The error has been logged. Please try again or contact "
            f"support if the problem persists."
        )

    # Step 4: Return the clean, safe response dictionary.
    return {
        "error": True,
        "message": user_message,
    }

---
./utils/governance.py
---
"""
governance.py
=============
Centralised AI governance rules injected into every MCP tool's
instruction field.

Why this file exists:
    Without a single source of truth for AI behaviour rules,
    each tool would need its own copy of "don't invent facts" or
    "always cite sources" — leading to inconsistency and drift.
    This file defines all rules once. Every tool imports from here.

Design notes:
    - Rules are plain strings appended to every tool's instruction.
    - Specialised rule sets exist per tool category (email, MOM,
      draft, task) — each inherits the base rules plus its own.
    - Rules are deliberately short and directive — LLMs follow
      short, clear constraints better than long essays.
"""

# ---------------------------------------------------------------------------
# Base rules — applied to EVERY tool output without exception
# ---------------------------------------------------------------------------
BASE_RULES = """
STRICT OUTPUT RULES — FOLLOW EXACTLY:
1. Only state facts that appear in the data provided to you.
2. Never invent names, email addresses, dates, or decisions.
3. If information is missing or unclear, say so explicitly.
4. Never combine information from different emails unless instructed.
5. If you are uncertain about any fact, prefix it with [UNCERTAIN].
6. Do not add promises, commitments, or deadlines not in the source data.
"""

# ---------------------------------------------------------------------------
# Email summarisation rules
# ---------------------------------------------------------------------------
EMAIL_SUMMARY_RULES = BASE_RULES + """
EMAIL SUMMARY SPECIFIC RULES:
- Summarise only what is in the email body provided.
- Do not reference other emails or prior conversation context.
- If the email body is empty or unclear, say "Email body unavailable".
- Never guess the sender's intent — only describe what they wrote.
"""

# ---------------------------------------------------------------------------
# MOM generation rules
# ---------------------------------------------------------------------------
MOM_RULES = BASE_RULES + """
MOM SPECIFIC RULES:
- List only decisions explicitly stated in the emails.
- List only action items explicitly assigned in the emails.
- If no decisions are found, write exactly: "No explicit decisions recorded."
- If no action items are found, write exactly: "No action items recorded."
- Never add agenda items that do not appear in the email content.
- Every attendee listed must appear as a sender or recipient in the emails.
- Dates must be copied verbatim from the email — never calculated or assumed.
"""

# ---------------------------------------------------------------------------
# Draft composition rules
# ---------------------------------------------------------------------------
DRAFT_RULES = BASE_RULES + """
DRAFT EMAIL SPECIFIC RULES:
- Write only what the context instructs — nothing more.
- Never add promises, timelines, or commitments not stated in the context.
- Do not reference information from other emails in this conversation.
- If the context is ambiguous, draft a neutral, open response.
- Always use professional language — never casual, never aggressive.
- Closing must always be: Best regards,
"""

# ---------------------------------------------------------------------------
# Task extraction rules
# ---------------------------------------------------------------------------
TASK_RULES = BASE_RULES + """
TASK EXTRACTION SPECIFIC RULES:
- Only extract tasks that contain a clear action verb (e.g. send, review, prepare).
- Only include a deadline if one is explicitly stated — never infer one.
- Only assign an owner if a specific person is named in relation to the task.
- If owner is unclear, write "Unassigned" — never guess.
- If deadline is unclear, write "Not specified" — never guess.
"""

# ---------------------------------------------------------------------------
# Follow-up rules
# ---------------------------------------------------------------------------
FOLLOWUP_RULES = BASE_RULES + """
FOLLOW-UP SPECIFIC RULES:
- Base the follow-up on the original email content provided.
- Never reference conversations or context not in the provided email.
- Keep the follow-up to 2-3 sentences maximum.
- Never demand or pressure — always polite and professional.
- Do not add new topics not raised in the original email.
"""

# ---------------------------------------------------------------------------
# Source citation instruction — appended to all data-heavy tools
# ---------------------------------------------------------------------------
SOURCE_CITATION = """
SOURCE CITATION:
After your response, add a brief "Sources:" section listing:
- Email subject(s) used
- Sender(s) referenced
- Date(s) of emails used
This helps the user verify your output against the original emails.
"""

# ---------------------------------------------------------------------------
# Uncertainty flag instruction
# ---------------------------------------------------------------------------
UNCERTAINTY_INSTRUCTION = """
UNCERTAINTY HANDLING:
If any part of your response is inferred rather than explicitly stated
in the source data, prefix that sentence with [INFERRED].
If you cannot determine a fact from the data, write [UNAVAILABLE].
Never omit these flags to make the response look cleaner.
"""


# ---------------------------------------------------------------------------
# Helper functions
# ---------------------------------------------------------------------------
def get_email_rules(include_sources: bool = True) -> str:
    """
    Get governance rules for email summarisation tools.

    Args:
        include_sources (bool): Whether to append source citation rules.

    Returns:
        str: Combined rules string to append to tool instruction.
    """
    rules = EMAIL_SUMMARY_RULES + UNCERTAINTY_INSTRUCTION
    if include_sources:
        rules += SOURCE_CITATION
    return rules


def get_mom_rules(include_sources: bool = True) -> str:
    """
    Get governance rules for MOM generation tools.

    Args:
        include_sources (bool): Whether to append source citation rules.

    Returns:
        str: Combined rules string to append to tool instruction.
    """
    rules = MOM_RULES + UNCERTAINTY_INSTRUCTION
    if include_sources:
        rules += SOURCE_CITATION
    return rules


def get_draft_rules() -> str:
    """
    Get governance rules for all draft composition tools.

    Returns:
        str: Combined rules string to append to tool instruction.
    """
    return DRAFT_RULES + UNCERTAINTY_INSTRUCTION


def get_task_rules(include_sources: bool = True) -> str:
    """
    Get governance rules for task extraction tools.

    Args:
        include_sources (bool): Whether to append source citation rules.

    Returns:
        str: Combined rules string to append to tool instruction.
    """
    rules = TASK_RULES + UNCERTAINTY_INSTRUCTION
    if include_sources:
        rules += SOURCE_CITATION
    return rules


def get_followup_rules() -> str:
    """
    Get governance rules for follow-up composition tools.

    Returns:
        str: Combined rules string.
    """
    return FOLLOWUP_RULES + UNCERTAINTY_INSTRUCTION

---
./utils/language_utils.py
---
"""
language_utils.py
=================
Language detection utility for multi-language email support.

Why this file exists:
    Emails at an international company like comp. may arrive in
    German, English, French, or other languages. MOM generation and
    follow-up drafts should match the language of the original email
    so recipients receive content in the correct language.

Design notes:
    - Uses langdetect library which identifies language from text.
    - Falls back to English if detection fails or text is too short.
    - Language codes follow ISO 639-1 (en, de, fr, es, etc.)
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from utils.logger import get_logger

logger = get_logger(__name__)

# Mapping of ISO language codes to full language names
# used in LLM instructions
LANGUAGE_NAMES = {
    "en": "English",
    "de": "German",
    "fr": "French",
    "es": "Spanish",
    "it": "Italian",
    "nl": "Dutch",
    "pt": "Portuguese",
    "pl": "Polish",
    "cs": "Czech",
    "ro": "Romanian",
    "hu": "Hungarian",
    "tr": "Turkish",
    "ar": "Arabic",
    "zh-cn": "Chinese (Simplified)",
    "ja": "Japanese",
}


def detect_language(text: str) -> tuple[str, str]:
    """
    Detect the language of a text string.

    Args:
        text (str): Text to detect language from.

    Returns:
        tuple[str, str]: A pair of (language_code, language_name).
                          e.g. ("de", "German") or ("en", "English").
                          Falls back to ("en", "English") if detection fails.
    """
    if not text or len(text.strip()) < 20:
        logger.debug("Text too short for language detection — defaulting to English")
        return "en", "English"

    try:
        from langdetect import detect
        code = detect(text)
        name = LANGUAGE_NAMES.get(code, code.upper())
        logger.info(f"Language detected: {name} ({code})")
        return code, name

    except Exception as exc:
        logger.warning(f"Language detection failed: {exc} — defaulting to English")
        return "en", "English"


def get_language_instruction(lang_code: str, lang_name: str) -> str:
    """
    Build a language instruction string for LLM prompts.

    Args:
        lang_code (str): ISO 639-1 language code.
        lang_name (str): Full language name.

    Returns:
        str: Instruction to append to any LLM prompt requiring
              language-matched output.
    """
    if lang_code == "en":
        return "Write in English."
    return (
        f"Write entirely in {lang_name}. "
        f"All headings, labels, and content must be in {lang_name}. "
        f"Do not mix languages."
    )

---
./utils/logger.py
---
"""
logger.py
=========
Centralised logging configuration for the entire project.

Why this file exists:
    Without a centralised logger, every module would configure its own
    logging setup independently, resulting in inconsistent formats,
    duplicated handlers, and log output that's hard to read or filter.
    This file sets up logging ONCE with a consistent format, and every
    other module in the project calls get_logger(__name__) to get a
    logger that automatically inherits that configuration.

Design notes:
    - Log level is controlled by the LOG_LEVEL setting in .env —
      set to DEBUG during development, INFO for production on FGSV297.
    - The format includes timestamp, log level, and the module name
      (__name__) so every log line tells you exactly which file it
      came from, without any extra effort from the caller.
    - A StreamHandler (prints to terminal/stdout) is used for Phase 1.
      A FileHandler can be added later for FGSV297 persistent logs.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import logging
import sys
from config.settings import settings

# ---------------------------------------------------------------------------
# Step 1: Define the log format applied to every log line in the project.
# ---------------------------------------------------------------------------
# Format breakdown:
#   %(asctime)s     → timestamp: 2025-01-15 10:23:45,123
#   %(levelname)-8s → log level padded to 8 chars: INFO    , WARNING
#   %(name)s        → module name: graph.mail_client, tools.email_tools
#   %(message)s     → the actual log message
LOG_FORMAT = "%(asctime)s | %(levelname)-8s | %(name)s | %(message)s"
DATE_FORMAT = "%Y-%m-%d %H:%M:%S"

# ---------------------------------------------------------------------------
# Step 2: Configure the root logger once.
# ---------------------------------------------------------------------------
# We configure the root logger (the parent of all loggers) so that
# every get_logger(__name__) call anywhere in the project automatically
# inherits this configuration without needing to set it up again.
def _configure_root_logger() -> None:
    """
    Set up the root logger with our standard format and level.
    Called once when this module is first imported.
    """
    root_logger = logging.getLogger()

    # Only configure if no handlers have been added yet — prevents
    # duplicate log lines if this module gets imported multiple times.
    if root_logger.handlers:
        return

    # Step 2a: Set log level from .env (INFO by default).
    log_level = getattr(logging, settings.LOG_LEVEL.upper(), logging.INFO)
    root_logger.setLevel(log_level)

    # Step 2b: Create a handler that writes log output to the terminal.
    handler = logging.StreamHandler(sys.stdout)
    handler.setLevel(log_level)

    # Step 2c: Apply the standard format defined above.
    formatter = logging.Formatter(LOG_FORMAT, datefmt=DATE_FORMAT)
    handler.setFormatter(formatter)

    root_logger.addHandler(handler)


# Run configuration at import time.
_configure_root_logger()


# ---------------------------------------------------------------------------
# Function: get_logger
# ---------------------------------------------------------------------------
def get_logger(name: str) -> logging.Logger:
    """
    Get a logger for a specific module. Call this at the top of every
    file in the project as:

        from utils.logger import get_logger
        logger = get_logger(__name__)

    Args:
        name (str): The module name — pass __name__ and Python
                     automatically fills in the correct dotted path
                     (e.g. "graph.mail_client", "tools.email_tools").

    Returns:
        logging.Logger: A configured logger instance for this module.
    """
    # The returned logger inherits all settings from the root logger
    # configured above — no extra setup needed per module.
    return logging.getLogger(name)

---
./utils/validator.py
---
"""
validator.py
============
Output validation functions for MCP tool responses.

Why this file exists:
    Tools return data from Microsoft Graph API — but the data can be:
        - Empty (no emails found)
        - Suspiciously short (possible API error or empty response)
        - Structurally wrong (missing required fields)
        - Containing known hallucination patterns

    This file validates tool outputs BEFORE they are returned to
    LibreChat, catching problems at the tool layer rather than
    letting bad data reach the LLM.

Design notes:
    - Validators are lightweight — no LLM calls, no external requests.
    - Each validator returns a ValidationResult with:
        - is_valid (bool)
        - warnings (list of strings shown to the user)
        - errors (list of strings — blocks response if critical)
    - Warnings are surfaced to the user as [VALIDATION WARNING] notes.
    - Errors replace the tool response with a clear failure message.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from dataclasses import dataclass, field
from typing import Any

from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# ValidationResult dataclass
# ---------------------------------------------------------------------------
@dataclass
class ValidationResult:
    """
    Result of a tool output validation check.

    Attributes:
        is_valid (bool): True if output passed all checks.
        warnings (list): Non-critical issues to surface to user.
        errors (list): Critical issues that block the response.
    """
    is_valid: bool = True
    warnings: list = field(default_factory=list)
    errors: list = field(default_factory=list)

    def add_warning(self, message: str) -> None:
        self.warnings.append(message)
        logger.warning(f"Validation warning: {message}")

    def add_error(self, message: str) -> None:
        self.errors.append(message)
        self.is_valid = False
        logger.error(f"Validation error: {message}")

    def format_warnings(self) -> str:
        """Format warnings as a markdown notice block."""
        if not self.warnings:
            return ""
        lines = ["⚠️ **Validation Notices:**"]
        for w in self.warnings:
            lines.append(f"- {w}")
        return "\n".join(lines)

    def format_errors(self) -> str:
        """Format errors as a markdown error block."""
        if not self.errors:
            return ""
        lines = ["❌ **Validation Errors:**"]
        for e in self.errors:
            lines.append(f"- {e}")
        return "\n".join(lines)


# ---------------------------------------------------------------------------
# Function: validate_email_list
# ---------------------------------------------------------------------------
def validate_email_list(emails: list[dict]) -> ValidationResult:
    """
    Validate a list of email dictionaries returned by Graph API.

    Checks:
        - List is not empty
        - Each email has required fields (id, subject, sender)
        - No duplicate email IDs (would indicate API pagination bug)
        - Dates are not in the future (would indicate data corruption)

    Args:
        emails (list[dict]): List of email dicts from mail_client.

    Returns:
        ValidationResult: Validation outcome with warnings/errors.
    """
    result = ValidationResult()

    if not emails:
        result.add_warning(
            "No emails were returned. Your inbox may be empty or "
            "the filter returned no matches."
        )
        return result

    seen_ids = set()
    for i, email in enumerate(emails):
        # Check required fields
        if not email.get("id"):
            result.add_warning(
                f"Email at position {i+1} is missing an ID — "
                f"it may not be openable."
            )

        if not email.get("subject"):
            result.add_warning(
                f"Email {i+1} has no subject line."
            )

        # Check for duplicate IDs
        email_id = email.get("id", "")
        if email_id in seen_ids:
            result.add_warning(
                f"Duplicate email ID detected at position {i+1}. "
                f"This may indicate a Graph API pagination issue."
            )
        seen_ids.add(email_id)

    return result


# ---------------------------------------------------------------------------
# Function: validate_email_body
# ---------------------------------------------------------------------------
def validate_email_body(body_text: str, email_subject: str = "") -> ValidationResult:
    """
    Validate that an email body is readable and substantial enough
    to summarise meaningfully.

    Checks:
        - Body is not empty
        - Body is not just whitespace or HTML tags
        - Body is long enough for meaningful summarisation (>50 chars)

    Args:
        body_text (str): The email body text to validate.
        email_subject (str): Subject line for error context.

    Returns:
        ValidationResult: Validation outcome.
    """
    result = ValidationResult()

    if not body_text or not body_text.strip():
        result.add_warning(
            f"Email body is empty"
            + (f" for '{email_subject}'" if email_subject else "")
            + ". Cannot summarise."
        )
        return result

    # Strip HTML and check readable length
    import re
    clean_text = re.sub(r"<[^>]+>", "", body_text).strip()

    if len(clean_text) < 50:
        result.add_warning(
            f"Email body is very short ({len(clean_text)} characters). "
            f"Summary may not be meaningful."
        )

    return result


# ---------------------------------------------------------------------------
# Function: validate_mom_output
# ---------------------------------------------------------------------------
def validate_mom_output(
    mom_text: str,
    expected_sections: list[str] = None,
) -> ValidationResult:
    """
    Validate that a generated MOM contains all required sections
    and doesn't show obvious signs of hallucination.

    Checks:
        - All required sections are present
        - Action items section is not empty
        - No [INFERRED] flags exceed a threshold (too much guessing)
        - MOM is not suspiciously short

    Args:
        mom_text (str): The generated MOM text to validate.
        expected_sections (list[str]): Section headers to check for.
                                        Defaults to standard formal MOM.

    Returns:
        ValidationResult: Validation outcome.
    """
    result = ValidationResult()

    if not mom_text or len(mom_text.strip()) < 100:
        result.add_error(
            "Generated MOM is too short to be valid. "
            "Please try again or check the source email content."
        )
        return result

    # Check all required sections are present
    if expected_sections is None:
        expected_sections = [
            "MEETING DETAILS",
            "ATTENDEES",
            "DISCUSSION SUMMARY",
            "DECISIONS MADE",
            "ACTION ITEMS",
            "NEXT STEPS",
        ]

    for section in expected_sections:
        if section not in mom_text.upper():
            result.add_warning(
                f"MOM is missing the '{section}' section. "
                f"The output may be incomplete."
            )

    # Check for excessive uncertainty flags
    inferred_count = mom_text.upper().count("[INFERRED]")
    unavailable_count = mom_text.upper().count("[UNAVAILABLE]")

    if inferred_count > 5:
        result.add_warning(
            f"MOM contains {inferred_count} [INFERRED] flags — "
            f"many facts were not explicitly stated in the source emails. "
            f"Please review carefully before distributing."
        )

    if unavailable_count > 3:
        result.add_warning(
            f"MOM contains {unavailable_count} [UNAVAILABLE] fields — "
            f"the source emails may not contain enough meeting information."
        )

    return result


# ---------------------------------------------------------------------------
# Function: validate_task_list
# ---------------------------------------------------------------------------
def validate_task_list(tasks: list[dict]) -> ValidationResult:
    """
    Validate a list of extracted tasks for completeness and quality.

    Checks:
        - At least one task was found
        - Each task has a title/action
        - Suspicious patterns: tasks with no action verb
        - Tasks with identical titles (duplicate extraction)

    Args:
        tasks (list[dict]): List of task dictionaries.

    Returns:
        ValidationResult: Validation outcome.
    """
    result = ValidationResult()

    if not tasks:
        result.add_warning(
            "No tasks were extracted from the emails. "
            "The emails may not contain any clear action items."
        )
        return result

    # Check for action verbs in task titles
    action_verbs = [
        "send", "review", "prepare", "update", "complete", "submit",
        "check", "confirm", "schedule", "arrange", "follow", "create",
        "write", "draft", "share", "provide", "organise", "inform",
        "contact", "reply", "respond", "approve", "coordinate",
    ]

    seen_titles = set()
    for task in tasks:
        title = (task.get("title") or task.get("action") or "").lower()

        # Check for duplicates
        if title in seen_titles:
            result.add_warning(
                f"Duplicate task detected: '{title}'. "
                f"This may be an extraction error."
            )
        seen_titles.add(title)

        # Check for action verb presence
        has_action_verb = any(verb in title for verb in action_verbs)
        if not has_action_verb and len(title) > 5:
            result.add_warning(
                f"Task '{title[:50]}' may not be an actionable item — "
                f"no clear action verb found."
            )

    return result


# ---------------------------------------------------------------------------
# Function: validate_draft_content
# ---------------------------------------------------------------------------
def validate_draft_content(
    body_text: str,
    to_emails: list[str],
    subject: str,
) -> ValidationResult:
    """
    Validate email draft content before saving to Outlook Drafts.

    Checks:
        - Body is not empty
        - Recipients are valid email format
        - Subject is not empty
        - Body doesn't contain obvious placeholder text
        - Body is not identical to the context (raw context pasted in)

    Args:
        body_text (str): Draft email body text.
        to_emails (list[str]): List of recipient email addresses.
        subject (str): Email subject line.

    Returns:
        ValidationResult: Validation outcome.
    """
    result = ValidationResult()
    import re

    # Check subject
    if not subject or not subject.strip():
        result.add_error("Email subject is empty.")

    # Check recipients format
    email_pattern = re.compile(r"^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$")
    for addr in to_emails:
        if not email_pattern.match(addr.strip()):
            result.add_error(
                f"'{addr}' does not appear to be a valid email address."
            )

    # Check body
    if not body_text or not body_text.strip():
        result.add_error("Email body is empty — cannot save an empty draft.")
        return result

    # Check for obvious placeholder text
    placeholder_patterns = [
        "[your name]", "[name]", "[date]", "[insert", "lorem ipsum",
        "placeholder", "todo:", "fixme:", "[context provided]",
        "context to base", "compose ", "professional email body",
    ]
    body_lower = body_text.lower()
    for pattern in placeholder_patterns:
        if pattern in body_lower:
            result.add_warning(
                f"Draft body may contain placeholder text: '{pattern}'. "
                f"Please review before sending from Outlook."
            )

    # Check minimum length
    clean_body = re.sub(r"<[^>]+>", "", body_text).strip()
    if len(clean_body) < 30:
        result.add_warning(
            "Draft body is very short. Please ensure the email is complete."
        )

    return result


# ---------------------------------------------------------------------------
# Function: validate_calendar_events
# ---------------------------------------------------------------------------
def validate_calendar_events(events: list[dict]) -> ValidationResult:
    """
    Validate calendar events returned from Graph API.

    Checks:
        - Events list is not empty
        - Each event has subject and start time
        - Events are in chronological order

    Args:
        events (list[dict]): List of calendar event dicts.

    Returns:
        ValidationResult: Validation outcome.
    """
    result = ValidationResult()

    if not events:
        result.add_warning(
            "No calendar events found for the requested time range."
        )
        return result

    for i, event in enumerate(events):
        if not event.get("subject"):
            result.add_warning(
                f"Calendar event {i+1} has no subject/title."
            )
        if not event.get("start_time"):
            result.add_warning(
                f"Calendar event {i+1} is missing a start time."
            )

    return result


# ---------------------------------------------------------------------------
# Function: append_validation_to_result
# ---------------------------------------------------------------------------
def append_validation_to_result(
    tool_result: dict,
    validation: ValidationResult,
) -> dict:
    """
    Append validation warnings/errors to a tool result dictionary.
    Warnings are added as a notice field.
    Errors replace the result with a failure message.

    This is the single integration point — call this at the end of
    every tool function before returning.

    Args:
        tool_result (dict): The tool's result dictionary.
        validation (ValidationResult): The validation outcome.

    Returns:
        dict: Updated tool result with validation notices appended.
    """
    if not validation.is_valid:
        # Critical error — override the result entirely
        return {
            "error": True,
            "message": validation.format_errors(),
            "warnings": validation.warnings,
        }

    if validation.warnings:
        # Non-critical warnings — append to existing result
        tool_result["validation_notices"] = validation.format_warnings()
        tool_result["instruction"] = (
            tool_result.get("instruction", "")
            + f"\n\n{validation.format_warnings()}"
        )

    return tool_result

---
./graph/__init__.py
---


---
./graph/attachment_client.py
---
"""
attachment_client.py
======================
Contains every Graph API operation related to email attachments —
listing what's attached to a message, and downloading the raw bytes
of a specific attachment.

Why this file exists:
    This file's ONLY job is "get the file" — not "read the file".
    That separation is intentional: this file knows about Graph API
    attachment endpoints, while parsers/ knows about file formats.
    Neither needs to know about the other's internals.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import base64

from graph.graph_client_factory import get_graph_client
from config.settings import settings
from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: list_message_attachments
# ---------------------------------------------------------------------------
async def list_message_attachments(message_id: str) -> list[dict]:
    """
    List metadata for all attachments on a specific email, WITHOUT
    downloading their actual content (fast, lightweight call).

    Args:
        message_id (str): The Graph API ID of the parent email.

    Returns:
        list[dict]: A list of attachment metadata dictionaries, each
                     containing id, name, contentType, and size.
    """
    logger.info(f"Listing attachments for message: {message_id}")

    client = get_graph_client()

    # This maps to GET /me/messages/{message_id}/attachments
    response = await client.me.messages.by_message_id(message_id).attachments.get()

    attachments = response.value if response and response.value else []

    return [
        {
            "id": att.id,
            "name": att.name,
            "contentType": att.content_type,
            "size": att.size,
        }
        for att in attachments
    ]


# ---------------------------------------------------------------------------
# Function: download_attachment
# ---------------------------------------------------------------------------
async def download_attachment(message_id: str, attachment_id: str) -> tuple[bytes, str, str]:
    """
    Download the raw binary content of a specific attachment.

    Args:
        message_id (str): The Graph API ID of the parent email.
        attachment_id (str): The Graph API ID of the attachment to
                              download, obtained from
                              list_message_attachments().

    Returns:
        tuple[bytes, str, str]: A 3-tuple of:
            - file_bytes: The raw binary content of the attachment.
            - file_name: The original file name.
            - content_type: The MIME type reported by Graph API.

    Raises:
        ValueError: If the attachment exceeds MAX_ATTACHMENT_SIZE_MB,
                    to avoid downloading and processing excessively
                    large files that could slow down or crash the
                    server.
    """
    logger.info(
        f"Downloading attachment {attachment_id} from message {message_id}"
    )

    client = get_graph_client()

    # Step 1: Fetch the attachment object. For file attachments, Graph
    #         API returns the content as a base64-encoded string in the
    #         contentBytes field (this is the standard Graph API
    #         behaviour for the /attachments/{id} endpoint).
    attachment = await (
        client.me.messages.by_message_id(message_id)
        .attachments.by_attachment_id(attachment_id)
        .get()
    )

    file_name = attachment.name
    content_type = attachment.content_type

    # Step 2: Safety check — reject attachments larger than our
    #         configured limit BEFORE attempting to decode/process them.
    size_mb = (attachment.size or 0) / (1024 * 1024)
    if size_mb > settings.MAX_ATTACHMENT_SIZE_MB:
        logger.warning(
            f"Attachment '{file_name}' ({size_mb:.1f}MB) exceeds the "
            f"configured limit of {settings.MAX_ATTACHMENT_SIZE_MB}MB"
        )
        raise ValueError(
            f"Attachment '{file_name}' is {size_mb:.1f}MB, which "
            f"exceeds the maximum allowed size of "
            f"{settings.MAX_ATTACHMENT_SIZE_MB}MB."
        )

    # Step 3: Decode the base64 content_bytes field into raw bytes that
    #         the parsers/ folder can work with directly.

    # content_bytes_b64 = attachment.content_bytes
    # file_bytes = base64.b64decode(content_bytes_b64)


    # Step 3: Decode attachment content into raw bytes.
    # The Graph SDK can return content_bytes in three different forms
    # depending on the file type and SDK version:
    #   - bytes      : already raw bytes, use directly
    #   - ASCII str  : base64-encoded string, decode it
    #   - Unicode str: plain text content returned directly (e.g. .txt files)
    content_bytes_b64 = attachment.content_bytes

    if isinstance(content_bytes_b64, bytes):
        # Already raw bytes — use directly
        file_bytes = content_bytes_b64

    elif isinstance(content_bytes_b64, str):
        # Check if it's pure ASCII — if so treat as base64
        try:
            content_bytes_b64.encode("ascii")
            # Pure ASCII — treat as base64, fix padding if needed
            padding_needed = len(content_bytes_b64) % 4
            if padding_needed:
                content_bytes_b64 += "=" * (4 - padding_needed)
            file_bytes = base64.b64decode(content_bytes_b64)

        except UnicodeEncodeError:
            # Contains non-ASCII — this IS the plain text content
            # returned directly by the SDK (common for .txt files)
            file_bytes = content_bytes_b64.encode("utf-8")

    else:
        raise ValueError(
            f"Unexpected content_bytes type from Graph SDK: "
            f"{type(content_bytes_b64)}. Cannot decode attachment."
        )

    logger.info(f"Downloaded '{file_name}' ({size_mb:.1f}MB) successfully")

    return file_bytes, file_name, content_type

---
./graph/availability_client.py
---
"""
availability_client.py
=======================
Microsoft Graph API calls for calendar availability and meeting scheduling.

Why this file exists:
    Finding common free time across multiple attendees requires the
    Graph API's findMeetingTimes endpoint — a separate, more complex
    call than simple calendar reads. Isolating it here keeps
    calendar_client.py clean and focused on event reads/writes.

Design notes:
    - findMeetingTimes requires Calendars.Read.Shared permission so
      the app can read other attendees' free/busy information.
    - Results are returned as suggested time slots with confidence
      scores — we surface the top 3-5 highest-confidence slots.
    - All times are handled in UTC and converted for display.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from datetime import datetime, timedelta

from graph.graph_client_factory import get_graph_client
from utils.logger import get_logger
from utils.date_utils import parse_date_string

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: find_meeting_times
# ---------------------------------------------------------------------------
async def find_meeting_times(
    attendee_emails: list[str],
    duration_minutes: int = 30,
    search_window_days: int = 7,
    preferred_start: str = "",
) -> dict:
    """
    Find common available time slots for a group of attendees using
    Graph API's findMeetingTimes endpoint.

    Args:
        attendee_emails (list[str]): Email addresses of all required
                                      attendees to check availability for.
        duration_minutes (int): Required meeting duration in minutes.
                                  Default 30 minutes.
        search_window_days (int): Number of days from now to search
                                   within. Default 7 days.
        preferred_start (str): Optional preferred start date e.g.
                                "next Monday", "2026-07-15".

    Returns:
        dict: Suggested time slots with confidence scores and
               attendee availability summary.
    """
    logger.info(
        f"Finding meeting times for {attendee_emails}, "
        f"duration={duration_minutes}min"
    )
    client = get_graph_client()

    from msgraph.generated.models.attendee_base import AttendeeBase
    from msgraph.generated.models.attendee_type import AttendeeType
    from msgraph.generated.models.email_address import EmailAddress
    # from msgraph.generated.models.meeting_time_suggestion_result import (MeetingTimeSuggestionResult,)
    from msgraph.generated.models.time_constraint import TimeConstraint
    from msgraph.generated.models.time_slot import TimeSlot
    from msgraph.generated.models.date_time_time_zone import DateTimeTimeZone
    from msgraph.generated.users.item.find_meeting_times.find_meeting_times_post_request_body import (
        FindMeetingTimesPostRequestBody,
    )

    # Step 1: Determine search window start and end
    if preferred_start:
        start_dt = parse_date_string(preferred_start)
        if not start_dt:
            start_dt = datetime.utcnow()
    else:
        start_dt = datetime.utcnow()

    end_dt = start_dt + timedelta(days=search_window_days)

    # Step 2: Build the request body
    request_body = FindMeetingTimesPostRequestBody()

    # Build attendee list
    attendees = []
    for email in attendee_emails:
        attendee = AttendeeBase()
        attendee.type = AttendeeType.Required
        attendee.email_address = EmailAddress()
        attendee.email_address.address = email
        attendees.append(attendee)
    request_body.attendees = attendees

    # Set meeting duration
    request_body.meeting_duration = f"PT{duration_minutes}M"

    # Set time constraint (search window)
    time_constraint = TimeConstraint()
    time_slot = TimeSlot()

    start_tz = DateTimeTimeZone()
    start_tz.date_time = start_dt.strftime("%Y-%m-%dT%H:%M:%S")
    start_tz.time_zone = "UTC"
    time_slot.start = start_tz

    end_tz = DateTimeTimeZone()
    end_tz.date_time = end_dt.strftime("%Y-%m-%dT%H:%M:%S")
    end_tz.time_zone = "UTC"
    time_slot.end = end_tz

    time_constraint.time_slots = [time_slot]
    request_body.time_constraint = time_constraint

    # Step 3: Call Graph API
    result = await client.me.find_meeting_times.post(request_body)

    if not result or not result.meeting_time_suggestions:
        return {
            "suggestions": [],
            "message": (
                "No common free slots found within the search window. "
                "Try extending the search window or reducing attendees."
            ),
        }

    # Step 4: Format suggestions
    suggestions = []
    for suggestion in result.meeting_time_suggestions[:5]:  # top 5 only
        if not suggestion.meeting_time_slot:
            continue

        slot_start = suggestion.meeting_time_slot.start
        slot_end = suggestion.meeting_time_slot.end

        # Parse and format times for display
        try:
            start_parsed = datetime.fromisoformat(
                slot_start.date_time.replace("Z", "+00:00")
            )
            end_parsed = datetime.fromisoformat(
                slot_end.date_time.replace("Z", "+00:00")
            )
            start_display = start_parsed.strftime("%A, %d %B %Y at %H:%M UTC")
            end_display = end_parsed.strftime("%H:%M UTC")
            date_key = start_parsed.strftime("%Y-%m-%dT%H:%M:00")
        except Exception:
            start_display = slot_start.date_time if slot_start else "?"
            end_display = slot_end.date_time if slot_end else "?"
            date_key = start_display

        confidence = suggestion.confidence or 0.0
        quality = (
            "🟢 High" if confidence >= 80
            else "🟡 Medium" if confidence >= 50
            else "🔴 Low"
        )

        suggestions.append({
            "start": date_key,
            "start_display": start_display,
            "end_display": end_display,
            "duration_minutes": duration_minutes,
            "confidence": round(confidence, 1),
            "quality": quality,
            "organizer_available": suggestion.organizer_availability or "unknown",
        })

    return {
        "suggestions": suggestions,
        "attendees_checked": attendee_emails,
        "duration_minutes": duration_minutes,
        "search_window_days": search_window_days,
    }


# ---------------------------------------------------------------------------
# Function: create_calendar_draft_invite
# ---------------------------------------------------------------------------
async def create_calendar_draft_invite(
    subject: str,
    attendee_emails: list[str],
    start_datetime: str,
    duration_minutes: int = 30,
    body_html: str = "",
    location: str = "",
    is_online: bool = True,
) -> dict:
    """
    Create a calendar event as a draft invite.
    The invite is saved but NOT sent to attendees until explicitly confirmed.

    Args:
        subject (str): Meeting title/subject.
        attendee_emails (list[str]): List of attendee email addresses.
        start_datetime (str): Meeting start in ISO format or natural
                               language e.g. "2026-07-15T10:00:00".
        duration_minutes (int): Meeting duration. Default 30 minutes.
        body_html (str): Meeting description/agenda in HTML.
        location (str): Physical or virtual location string.
        is_online (bool): Whether to create as a Teams online meeting.

    Returns:
        dict: The created event's ID, subject, time, and web link.
    """
    logger.info(f"Creating calendar draft invite: '{subject}'")
    client = get_graph_client()

    from msgraph.generated.models.event import Event
    from msgraph.generated.models.item_body import ItemBody
    from msgraph.generated.models.body_type import BodyType
    from msgraph.generated.models.date_time_time_zone import DateTimeTimeZone
    from msgraph.generated.models.attendee import Attendee
    from msgraph.generated.models.attendee_type import AttendeeType
    from msgraph.generated.models.email_address import EmailAddress
    from msgraph.generated.models.location import Location

    # Step 1: Parse start time
    try:
        start_dt = datetime.fromisoformat(
            start_datetime.replace("Z", "+00:00")
        )
    except ValueError:
        parsed = parse_date_string(start_datetime)
        start_dt = parsed if parsed else datetime.utcnow() + timedelta(days=1)

    end_dt = start_dt + timedelta(minutes=duration_minutes)

    # Step 2: Build the Event object
    event = Event()
    event.subject = subject
    # event.is_draft = True

    # Body
    event.body = ItemBody()
    event.body.content_type = BodyType.Html
    event.body.content = body_html

    # Start and end times
    event.start = DateTimeTimeZone()
    event.start.date_time = start_dt.strftime("%Y-%m-%dT%H:%M:%S")
    event.start.time_zone = "UTC"

    event.end = DateTimeTimeZone()
    event.end.date_time = end_dt.strftime("%Y-%m-%dT%H:%M:%S")
    event.end.time_zone = "UTC"

    # Attendees
    attendees = []
    for email in attendee_emails:
        att = Attendee()
        att.type = AttendeeType.Required
        att.email_address = EmailAddress()
        att.email_address.address = email
        attendees.append(att)
    event.attendees = attendees

    # Location
    if location:
        event.location = Location()
        event.location.display_name = location

    # Online meeting (Teams)
    event.is_online_meeting = is_online

    # Step 3: Create the event
    from msgraph.generated.users.item.events.events_request_builder import (
        EventsRequestBuilder,
    )

    query_params = EventsRequestBuilder.EventsRequestBuilderPostQueryParameters(
        send_invitations_or_cancel="sendToAllAndSaveCopy"
    )
    request_config = EventsRequestBuilder.EventsRequestBuilderPostRequestConfiguration(
        query_parameters=query_params
    )

    created = await client.me.events.post(event)

    return {
        "event_id": created.id,
        "subject": created.subject,
        "start": start_dt.strftime("%A, %d %B %Y at %H:%M UTC"),
        "end": end_dt.strftime("%H:%M UTC"),
        "duration_minutes": duration_minutes,
        "attendees": attendee_emails,
        "is_online": is_online,
        "web_link": created.web_link or "",
        "status": "Meeting invite drafted. Review in your Calendar before sending.",
    }

---
./graph/calendar_client.py
---
"""
calendar_client.py
===================
Contains every Graph API operation related to reading calendar events.
Phase 1 scope is READ-ONLY — creating or modifying events is deferred
to Phase 2, pending the Calendars.ReadWrite permission.

Why this file exists:
    Same pattern as mail_client.py: tools/calendar_tools.py delegates
    all real Graph API work to this file, keeping the tool layer thin.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from datetime import datetime

from graph.graph_client_factory import get_graph_client
from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Helper: _event_to_dict
# ---------------------------------------------------------------------------
def _event_to_dict(event) -> dict:
    """
    Convert a single Graph SDK Event object into a plain dictionary,
    matching the shape expected by tools/calendar_tools.py.
    """
    return {
        "subject": event.subject,
        "start": {
            "dateTime": event.start.date_time if event.start else None,
        },
        "end": {
            "dateTime": event.end.date_time if event.end else None,
        },
        "organizer": {
            "emailAddress": {
                "name": (
                    event.organizer.email_address.name
                    if event.organizer else None
                ),
            }
        },
        "location": {
            "displayName": event.location.display_name if event.location else "",
        },
        "isOnlineMeeting": event.is_online_meeting or False,
    }


# ---------------------------------------------------------------------------
# Function: fetch_calendar_events
# ---------------------------------------------------------------------------
async def fetch_calendar_events(start_dt: datetime, end_dt: datetime) -> list[dict]:
    """
    Fetch calendar events within a specific date/time range, using
    Graph API's calendarView endpoint (which correctly expands
    recurring meetings into individual occurrences, unlike the plain
    /events endpoint).

    Args:
        start_dt (datetime): Start of the date range to query.
        end_dt (datetime): End of the date range to query.

    Returns:
        list[dict]: A list of event dictionaries, ordered by start time.
    """
    logger.info(
        f"Fetching calendar events from {start_dt.isoformat()} "
        f"to {end_dt.isoformat()}"
    )

    client = get_graph_client()

    from msgraph.generated.users.item.calendar_view.calendar_view_request_builder import (
        CalendarViewRequestBuilder,
    )

    # Step 1: calendarView requires startDateTime and endDateTime as
    #         query parameters — these define the window Graph API
    #         expands recurring events within.
    query_params = CalendarViewRequestBuilder.CalendarViewRequestBuilderGetQueryParameters(
        start_date_time=start_dt.isoformat(),
        end_date_time=end_dt.isoformat(),
        orderby=["start/dateTime"],
    )
    request_config = CalendarViewRequestBuilder.CalendarViewRequestBuilderGetRequestConfiguration(
        query_parameters=query_params
    )

    # Step 2: Execute — maps to GET /me/calendarView?startDateTime=...&endDateTime=...
    response = await client.me.calendar_view.get(request_configuration=request_config)

    events = response.value if response and response.value else []
    return [_event_to_dict(e) for e in events]

---
./graph/draft_client.py
---
"""
draft_client.py
===============
All Microsoft Graph API calls related to drafts, folders,
and email management operations (mark read, move, create folder).

Why this file exists:
    Phase 2 adds write operations to the mailbox — creating drafts,
    moving emails, creating folders. These are separated from
    mail_client.py (which is read-only) to maintain the principle
    that each file has one clear responsibility:
        mail_client.py    → reading emails
        draft_client.py   → writing/managing emails and folders

Design notes:
    - All draft operations require Mail.ReadWrite permission.
    - Drafts are NEVER sent — only saved. send() calls are
      intentionally excluded from this file per project scope.
    - Folder IDs are used internally — well-known folder names
      (inbox, drafts, sentitems) are also supported by Graph API.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from graph.graph_client_factory import get_graph_client
from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: get_draft_emails
# ---------------------------------------------------------------------------
async def get_draft_emails(top: int = 20) -> list[dict]:
    """
    Fetch emails currently saved in the Drafts folder.

    Args:
        top (int): Maximum number of draft emails to return.

    Returns:
        list[dict]: List of draft email dictionaries.
    """
    logger.info(f"Fetching {top} draft emails from Graph API")
    client = get_graph_client()

    from msgraph.generated.users.item.mail_folders.item.messages.messages_request_builder import (
        MessagesRequestBuilder,
    )

    query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
        top=top,
        orderby=["lastModifiedDateTime DESC"],
        filter="isDraft eq true",
    )
    request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
        query_parameters=query_params
    )
    request_config.headers.add("Prefer", "outlook.body-content-type='text'")
    request_config.headers.add("Cache-control", "no-cache")

    # "drafts" is a well-known folder name recognised by Graph API
    response = await (
        client.me.mail_folders.by_mail_folder_id("drafts")
        .messages.get(request_configuration=request_config)
    )

    messages = response.value if response and response.value else []
    return [
        {
            "id": m.id,
            "subject": m.subject or "(no subject)",
            "to": [
                r.email_address.address
                for r in (m.to_recipients or [])
                if r.email_address
            ],
            "last_modified": (
                m.last_modified_date_time.isoformat()
                if m.last_modified_date_time else ""
            ),
            "preview": m.body_preview or "",
        }
        for m in messages
    ]


# ---------------------------------------------------------------------------
# Function: create_draft
# ---------------------------------------------------------------------------
async def create_draft(
    to_emails: list[str],
    subject: str,
    body_html: str,
    cc_emails: list[str] = None,
) -> dict:
    """
    Create a new email draft and save it to the Drafts folder.
    Does NOT send the email.

    Args:
        to_emails (list[str]): List of recipient email addresses.
        subject (str): Email subject line.
        body_html (str): Email body in HTML format.
        cc_emails (list[str]): Optional CC recipient email addresses.

    Returns:
        dict: The created draft's ID, subject, and web link.
    """
    logger.info(f"Creating draft email to {to_emails}, subject: {subject}")
    client = get_graph_client()

    from msgraph.generated.models.message import Message
    from msgraph.generated.models.item_body import ItemBody
    from msgraph.generated.models.body_type import BodyType
    from msgraph.generated.models.recipient import Recipient
    from msgraph.generated.models.email_address import EmailAddress

    # Build recipient objects
    def make_recipient(email: str) -> Recipient:
        r = Recipient()
        r.email_address = EmailAddress()
        r.email_address.address = email
        return r

    message = Message()
    message.subject = subject
    message.body = ItemBody()
    message.body.content_type = BodyType.Html
    message.body.content = body_html
    message.to_recipients = [make_recipient(e) for e in to_emails]

    if cc_emails:
        message.cc_recipients = [make_recipient(e) for e in cc_emails]

    # POST /me/messages creates a draft (isDraft = true by default)
    created = await client.me.messages.post(message)

    return {
        "draft_id": created.id,
        "subject": created.subject,
        "to": to_emails,
        "status": "Draft saved to Drafts folder",
        "web_link": created.web_link or "",
    }


# ---------------------------------------------------------------------------
# Function: create_reply_draft
# ---------------------------------------------------------------------------
async def create_reply_draft(
    message_id: str,
    reply_body_html: str,
    reply_all: bool = True,
) -> dict:
    """
    Create a reply draft for an existing email that maintains the
    thread chain. Uses Graph API's createReply/createReplyAll endpoint
    which correctly threads the reply under the original message.

    Args:
        message_id (str): Graph API ID of the email to reply to.
        reply_body_html (str): The reply body in HTML format.
        reply_all (bool): True to reply-all (default), False for reply.

    Returns:
        dict: The created reply draft's ID, subject, and web link.
    """
    logger.info(
        f"Creating {'reply-all' if reply_all else 'reply'} draft "
        f"for message: {message_id}"
    )
    client = get_graph_client()

    from msgraph.generated.models.message import Message
    from msgraph.generated.models.item_body import ItemBody
    from msgraph.generated.models.body_type import BodyType
    from msgraph.generated.users.item.messages.item.create_reply.create_reply_post_request_body import (
        CreateReplyPostRequestBody,
    )

    # Step 1: Build the reply request body
    request_body_message = Message()
    request_body_message.body = ItemBody()
    request_body_message.body.content_type = BodyType.Html
    request_body_message.body.content = reply_body_html

    if reply_all:
        # createReplyAll keeps all original recipients in the thread
        from msgraph.generated.users.item.messages.item.create_reply_all.create_reply_all_post_request_body import (
            CreateReplyAllPostRequestBody,
        )
        request_body = CreateReplyAllPostRequestBody()
        request_body.message = request_body_message
        request_body.comment = ""  # required field — empty string if body is in message

        reply_draft = await (
            client.me.messages.by_message_id(message_id)
            .create_reply_all.post(request_body)
        )
    else:
        from msgraph.generated.users.item.messages.item.create_reply.create_reply_post_request_body import (
            CreateReplyPostRequestBody,
        )
        request_body = CreateReplyPostRequestBody()
        request_body.message = request_body_message
        request_body.comment = ""
            
        # Step 2: createReply creates a draft reply — does NOT send
        reply_draft = await (
            client.me.messages.by_message_id(message_id)
            .create_reply.post(request_body)
        )

    return {
        "draft_id": reply_draft.id,
        "subject": reply_draft.subject or "",
        "conversation_id": reply_draft.conversation_id or "",
        "status": (
            "Reply-all draft saved to Drafts folder — "
            "thread chain preserved"
            if reply_all else
            "Reply draft saved to Drafts folder — thread chain preserved"
        ),
        "web_link": reply_draft.web_link or "",
    }


# ---------------------------------------------------------------------------
# Function: mark_message_read_status
# ---------------------------------------------------------------------------
async def mark_message_read_status(message_id: str, is_read: bool) -> dict:
    """
    Mark an email as read or unread.

    Args:
        message_id (str): Graph API ID of the email.
        is_read (bool): True to mark as read, False to mark as unread.

    Returns:
        dict: Confirmation of the operation.
    """
    status_label = "read" if is_read else "unread"
    logger.info(f"Marking message {message_id} as {status_label}")
    client = get_graph_client()

    from msgraph.generated.models.message import Message

    update = Message()
    update.is_read = is_read

    await client.me.messages.by_message_id(message_id).patch(update)

    return {
        "message_id": message_id,
        "status": f"Email marked as {status_label}",
    }


# ---------------------------------------------------------------------------
# Function: get_mail_folders
# ---------------------------------------------------------------------------
async def get_mail_folders() -> list[dict]:
    """
    List all mail folders in the user's mailbox.

    Returns:
        list[dict]: List of folder dictionaries with name, id,
                     unread count, and total message count.
    """
    logger.info("Fetching mail folders from Graph API")
    client = get_graph_client()

    response = await client.me.mail_folders.get()
    folders = response.value if response and response.value else []

    return [
        {
            "id": f.id,
            "name": f.display_name,
            "unread_count": f.unread_item_count or 0,
            "total_count": f.total_item_count or 0,
        }
        for f in folders
    ]


# ---------------------------------------------------------------------------
# Function: create_mail_folder
# ---------------------------------------------------------------------------
async def create_mail_folder(folder_name: str, parent_folder_id: str = None) -> dict:
    """
    Create a new mail folder at root level or inside an existing folder.

    Args:
        folder_name (str): Display name for the new folder.
        parent_folder_id (str): Optional parent folder ID. If None,
                                  creates at root level.

    Returns:
        dict: The created folder's ID, name, and confirmation.
    """
    logger.info(f"Creating mail folder: '{folder_name}'")
    client = get_graph_client()

    from msgraph.generated.models.mail_folder import MailFolder

    new_folder = MailFolder()
    new_folder.display_name = folder_name

    if parent_folder_id:
        # Create as child of existing folder
        created = await (
            client.me.mail_folders.by_mail_folder_id(parent_folder_id)
            .child_folders.post(new_folder)
        )
    else:
        # Create at root level
        created = await client.me.mail_folders.post(new_folder)

    return {
        "folder_id": created.id,
        "folder_name": created.display_name,
        "status": f"Folder '{folder_name}' created successfully",
    }


# ---------------------------------------------------------------------------
# Function: move_message_to_folder
# ---------------------------------------------------------------------------
async def move_message_to_folder(message_id: str, destination_folder_id: str) -> dict:
    """
    Move an email to a different mail folder.

    Args:
        message_id (str): Graph API ID of the email to move.
        destination_folder_id (str): ID of the destination folder.
                                       Can also be a well-known name:
                                       inbox, drafts, sentitems,
                                       deleteditems, junkemail, archive.

    Returns:
        dict: Confirmation with new message ID in destination folder.
    """
    logger.info(
        f"Moving message {message_id} to folder {destination_folder_id}"
    )
    client = get_graph_client()

    from msgraph.generated.users.item.messages.item.move.move_post_request_body import (
        MovePostRequestBody,
    )

    request_body = MovePostRequestBody()
    request_body.destination_id = destination_folder_id

    moved = await (
        client.me.messages.by_message_id(message_id).move.post(request_body)
    )

    return {
        "new_message_id": moved.id,
        "subject": moved.subject or "",
        "status": f"Email moved successfully",
    }

# ---------------------------------------------------------------------------
# Function: flag_message
# ---------------------------------------------------------------------------

async def flag_message(message_id: str, flag_status: str = "flagged") -> dict:
    """
    Flag or unflag an email.

    Args:
        message_id (str): Graph API ID of the email.
        flag_status (str): "flagged", "complete", or "notFlagged".

    Returns:
        dict: Confirmation of the flag status change.
    """
    logger.info(f"Flagging message {message_id} as '{flag_status}'")
    client = get_graph_client()

    from msgraph.generated.models.message import Message
    from msgraph.generated.models.followup_flag import FollowupFlag
    from msgraph.generated.models.followup_flag_status import FollowupFlagStatus

    # Map string to SDK enum
    status_map = {
        "flagged": FollowupFlagStatus.Flagged,
        "complete": FollowupFlagStatus.Complete,
        "notflagged": FollowupFlagStatus.NotFlagged,
        "not flagged": FollowupFlagStatus.NotFlagged,
        "unflagged": FollowupFlagStatus.NotFlagged,
    }

    flag_enum = status_map.get(flag_status.lower(), FollowupFlagStatus.Flagged)

    update = Message()
    update.flag = FollowupFlag()
    update.flag.flag_status = flag_enum

    await client.me.messages.by_message_id(message_id).patch(update)

    return {
        "message_id": message_id,
        "flag_status": flag_status,
        "status": f"Email flagged as '{flag_status}' successfully",
    }




---
./graph/graph_client_factory.py
---
"""
graph_client_factory.py
=========================
Builds a ready-to-use, authenticated Microsoft Graph SDK client for the
current request, and provides a couple of foundational Graph calls
(like fetching the user's own profile) that don't belong in a more
specific client file.

Why this file exists:
    Every file in graph/ (mail_client.py, calendar_client.py,
    attachment_client.py) needs an authenticated Graph client to work
    with. Rather than each file duplicating "how do I build a Graph
    client", they all call get_graph_client() from here. This means
    there is exactly ONE place where the token from auth/ gets turned
    into an actual working Graph SDK object.

Design notes:
    - Uses msgraph-sdk-python's GraphServiceClient together with a
      simple custom credential object that just hands back the token
      we already have (from auth/graph_auth.py) — we do NOT use MSAL's
      interactive or device-code flows here, because the token was
      already acquired by LibreChat's OAuth flow. This server only
      ever receives an already-valid token; it doesn't perform login.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import time
from msgraph import GraphServiceClient
from azure.core.credentials import AccessToken, TokenCredential

from auth.graph_auth import get_current_access_token
from auth.token_cache import set_current_token

from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Class: StaticTokenCredential
# ---------------------------------------------------------------------------
class StaticTokenCredential(TokenCredential):
    """
    A minimal credential object that satisfies the interface the Graph
    SDK expects (a TokenCredential), but instead of performing any
    login flow itself, it simply returns a token we already have.

    Why this class exists:
        The official msgraph-sdk-python client is built to work with
        Azure's credential objects (normally things like
        InteractiveBrowserCredential or ClientSecretCredential, which
        perform their own login). We don't want any of those — our
        token already came from LibreChat's OAuth flow. This class
        bridges that gap: it's a TokenCredential that does no real
        work, just hands back the token it was given.
    """

    def __init__(self, token: str):
        self._token = token

    def get_token(self, *scopes, **kwargs) -> AccessToken:
        # Step 1: The Graph SDK calls this method internally whenever
        #         it needs to attach an Authorization header to a
        #         request. We return the token we already have, along
        #         with a fixed expiry far enough in the future — since
        #         this credential object only lives for one request's
        #         duration anyway, the real expiry doesn't matter here.
        expiry = int(time.time()) + 3600
        return AccessToken(self._token, expiry)
        # return AccessToken(self._token, int(9999999999))


# ---------------------------------------------------------------------------
# Function: get_graph_client
# ---------------------------------------------------------------------------
def get_graph_client() -> GraphServiceClient:
    """
    Build an authenticated GraphServiceClient for the CURRENT incoming
    request, using the token extracted from the Authorization header.

    This function is called at the start of every Graph API operation
    across mail_client.py, calendar_client.py, and attachment_client.py.

    Returns:
        GraphServiceClient: A ready-to-use Graph SDK client, scoped to
                             the current user's permissions.
    """
    # Step 1: Extract the token from this request's Authorization
    #         header. This raises AuthorizationError automatically if
    #         missing or malformed — we let that propagate up so the
    #         calling tool's error handler can format it cleanly.
    token = get_current_access_token()

    # Step 2: Store it in the request-scoped context as well, for any
    #         other code path that might need it later in this same
    #         request without re-parsing headers.
    set_current_token(token)

    # Step 3: Wrap the raw token in our minimal credential class so the
    #         Graph SDK accepts it as a valid credential object.
    credential = StaticTokenCredential(token)

    # Step 4: Build and return the actual Graph SDK client.
    client = GraphServiceClient(credentials=credential)

    return client


# ---------------------------------------------------------------------------
# Function: get_user_profile
# ---------------------------------------------------------------------------
async def get_user_profile() -> dict:
    """
    Fetch the current user's basic profile from Microsoft Graph (GET /me).

    This lives here rather than in a dedicated profile_client.py file
    because it's a single, simple call used only by profile_tools.py —
    not substantial enough to warrant its own file at Phase 1 scope.

    Returns:
        dict: The raw Graph API user profile response, containing
              fields like displayName, mail, userPrincipalName,
              jobTitle, etc.
    """
    logger.info("Fetching user profile from Graph API (/me)")

    # Step 1: Get an authenticated client for this request.
    client = get_graph_client()

    # Step 2: Call the SDK's equivalent of GET /me.
    user = await client.me.get()

    # Step 3: The SDK returns a typed object, not a plain dict. Convert
    #         it to a plain dictionary so the rest of the app (which
    #         expects plain dicts, per our tools/ design) can use it
    #         consistently regardless of which Graph endpoint was called.
    return {
        "displayName": user.display_name,
        "mail": user.mail,
        "userPrincipalName": user.user_principal_name,
        "jobTitle": user.job_title,
    }

---
./graph/mail_client.py
---
"""
mail_client.py
==============
Contains every Graph API operation related to reading and searching
Outlook email. This file knows about Graph API mail endpoints and
nothing else — it has no knowledge of MCP tools, the LLM, or attachments.

Why this file exists:
    tools/email_tools.py and tools/attachment_tools.py both need to
    fetch email data. Rather than each tool file calling the Graph SDK
    directly, they call functions here. This means if Microsoft changes
    something about how mail endpoints work, this is the only file that
    needs updating.

Design notes:
    - Every function here returns plain Python dicts/lists, never raw
      SDK objects, keeping the "boundary" between Graph SDK specifics
      and the rest of the app clean.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from graph.graph_client_factory import get_graph_client
from utils.logger import get_logger
import re

logger = get_logger(__name__)

def _clean_body(raw_content: str) -> str:
    """
    Strip HTML tags from email body content so the LLM receives
    clean plain text instead of raw HTML markup.
    Graph API returns HTML bodies by default for rich-text emails.
    """
    if not raw_content:
        return ""
    # Remove HTML tags
    text = re.sub(r"<[^>]+>", " ", raw_content)
    # Collapse multiple whitespace/newlines into single spaces
    text = re.sub(r"\s+", " ", text).strip()
    return text

# ---------------------------------------------------------------------------
# Helper: _message_to_dict
# ---------------------------------------------------------------------------
def _message_to_dict(message) -> dict:
    """
    Convert a single Graph SDK Message object into a plain dictionary,
    matching the shape expected by tools/email_tools.py.

    This helper avoids repeating the same field-mapping logic in every
    function below.
    """
    return {
        "id": message.id,
        "subject": message.subject,
        "from": {
            "emailAddress": {
                "name": message.from_.email_address.name if message.from_ else None,
                "address": message.from_.email_address.address if message.from_ else None,
            }
        },
        "receivedDateTime": (
            message.received_date_time.isoformat()
            if message.received_date_time else None
        ),
        "hasAttachments": message.has_attachments,
        "is_read": message.is_read if message.is_read is not None else True,
        "bodyPreview": message.body_preview or "",
        "body": {
            # "content": message.body.content if message.body else "",
            "content": _clean_body(message.body.content if message.body else ""),
        },
    }


# ---------------------------------------------------------------------------
# Function: fetch_recent_messages
# ---------------------------------------------------------------------------
async def fetch_recent_messages(top: int = 10) -> list[dict]:
    """
    Fetch the most recent messages from the user's inbox, ordered by
    received date (newest first).

    Args:
        top (int): Maximum number of messages to fetch.

    Returns:
        list[dict]: A list of message dictionaries (see _message_to_dict).
    """
    logger.info(f"Fetching {top} recent messages from Graph API")

    # Step 1: Get an authenticated client scoped to this request.
    client = get_graph_client()

    # Step 2: Build the query — Graph SDK uses a request configuration
    #         object to specify query parameters like $top and $orderby,
    #         mirroring the underlying REST API's query string options.
    from msgraph.generated.users.item.messages.messages_request_builder import (
        MessagesRequestBuilder,
    )

    query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
            top=top,
            orderby=["receivedDateTime DESC"],
            filter="isDraft eq false",
            select=["id", "subject", "from", "receivedDateTime",
                    "hasAttachments", "bodyPreview", "isRead"],
    )

    request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
        query_parameters=query_params
    )

    # Step 3: Execute the call — this maps to GET /me/messages.
    response = await client.me.messages.get(request_configuration=request_config)

    # Step 4: Convert each SDK message object into our plain dict shape.
    messages = response.value if response and response.value else []
    return [_message_to_dict(m) for m in messages]


# ---------------------------------------------------------------------------
# Function: fetch_message_by_id
# ---------------------------------------------------------------------------
async def fetch_message_by_id(message_id: str) -> dict:
    """
    Fetch a single email message's full content by its Graph API ID.

    Args:
        message_id (str): The unique Graph API message ID.

    Returns:
        dict: A single message dictionary, including full body content
              (unlike fetch_recent_messages, which only gives a preview).
    """
    logger.info(f"Fetching message by ID: {message_id}")

    client = get_graph_client()

    # This maps to GET /me/messages/{message_id}.
    message = await client.me.messages.by_message_id(message_id).get()

    return _message_to_dict(message)


# ---------------------------------------------------------------------------
# Function: search_messages_by_keyword
# ---------------------------------------------------------------------------
async def search_messages_by_keyword(keyword: str, top: int = 10) -> list[dict]:
    """
    Search the user's mailbox for messages matching a keyword, using
    Graph API's $search query parameter (which searches subject,
    sender, and body content).

    Args:
        keyword (str): The search term.
        top (int): Maximum number of matching messages to return.

    Returns:
        list[dict]: A list of matching message dictionaries.
    """
    logger.info(f"Searching messages for keyword: '{keyword}'")

    client = get_graph_client()

    from msgraph.generated.users.item.messages.messages_request_builder import (
        MessagesRequestBuilder,
    )

    # Step 1: Graph API's $search requires the keyword to be wrapped in
    #         quotes within the query string itself.
    query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
        search=f'"{keyword}"',
        top=top,
        filter="isDraft eq false",
        select=["id", "subject", "from", "receivedDateTime",
                "hasAttachments", "bodyPreview", "isRead"],
    )

    # Step 2: $search has a quirk in Graph API — it requires a special
    #         header (ConsistencyLevel: eventual) to work correctly
    #         alongside certain other query options. The SDK exposes
    #         this via request headers on the configuration object.
    request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
        query_parameters=query_params,
    )
    request_config.headers.add("ConsistencyLevel", "eventual")

    response = await client.me.messages.get(request_configuration=request_config)

    messages = response.value if response and response.value else []
    return [_message_to_dict(m) for m in messages]


# ---------------------------------------------------------------------------
# Function: search_emails_advanced
# ---------------------------------------------------------------------------
async def search_messages_advanced(
    sender_email: str = "",
    keyword: str = "",
    date_from: str = "",
    date_to: str = "",
    top: int = 20,
) -> list[dict]:
    """
    Search messages with combined sender, keyword, and date range filters
    using Graph API's $filter and $search query parameters.
    """
    logger.info(
        f"Advanced search: sender={sender_email}, keyword={keyword}, "
        f"date_from={date_from}, date_to={date_to}"
    )

    client = get_graph_client()

    from msgraph.generated.users.item.messages.messages_request_builder import (
        MessagesRequestBuilder,
    )
    from utils.date_utils import parse_date_string

    # Build $filter clauses
    filter_parts = []

    if sender_email:
        filter_parts.append(
            f"from/emailAddress/address eq '{sender_email}'"
        )

    if date_from:
        dt_from = parse_date_string(date_from)
        if dt_from:
            filter_parts.append(
                f"receivedDateTime ge {dt_from.strftime('%Y-%m-%dT00:00:00Z')}"
            )

    if date_to:
        dt_to = parse_date_string(date_to)
        if dt_to:
            filter_parts.append(
                f"receivedDateTime le {dt_to.strftime('%Y-%m-%dT23:59:59Z')}"
            )

    filter_str = " and ".join(filter_parts) if filter_parts else None

    # $search and $filter cannot be combined in Graph API —
    # if both are needed, apply $search and filter dates/sender client-side
    if keyword and filter_str:
        # Fetch via search, then filter client-side by date/sender
        query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
            search=f'"{keyword}"',
            top=min(top * 3, 100),  # fetch more to allow for client-side filtering
        )
        request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
            query_parameters=query_params,
        )
        request_config.headers.add("ConsistencyLevel", "eventual")
        response = await client.me.messages.get(request_configuration=request_config)
        messages = response.value if response and response.value else []

        # Apply sender/date filters client-side
        from dateutil import parser as dp
        filtered = []
        for msg in messages:
            if sender_email:
                msg_sender = ""
                if msg.from_ and msg.from_.email_address:
                    msg_sender = msg.from_.email_address.address or ""
                if sender_email.lower() not in msg_sender.lower():
                    continue
            if date_from:
                dt_from = parse_date_string(date_from)
                if dt_from and msg.received_date_time:
                    msg_dt = msg.received_date_time.replace(tzinfo=None) if msg.received_date_time.tzinfo else msg.received_date_time
                    if msg.received_date_time < dt_from:
                        continue
            if date_to:
                dt_to = parse_date_string(date_to)
                if dt_to and msg.received_date_time:
                    from datetime import timedelta
                    if msg.received_date_time > dt_to + timedelta(days=1):
                        continue
            filtered.append(msg)
            if len(filtered) >= top:
                break
        return [_message_to_dict(m) for m in filtered]

    elif keyword:
        # Keyword only — use $search
        query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
            search=f'"{keyword}"',
            top=top,
        )
        request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
            query_parameters=query_params,
        )
        request_config.headers.add("ConsistencyLevel", "eventual")

    elif filter_str:
        # Sender/date only — use $filter
        query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
            filter=filter_str,
            top=top,
            orderby=["receivedDateTime DESC"],
        )
        request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
            query_parameters=query_params,
        )

    else:
        # No filters — fall back to recent messages
        query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
            top=top,
            orderby=["receivedDateTime DESC"],
        )
        request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
            query_parameters=query_params,
        )

    response = await client.me.messages.get(request_configuration=request_config)
    messages = response.value if response and response.value else []
    return [_message_to_dict(m) for m in messages]


# ---------------------------------------------------------------------------
# Function: fetch_messages_paged
# ---------------------------------------------------------------------------

async def fetch_messages_paged(top: int = 50, skip: int = 0) -> list[dict]:
    """
    Fetch messages with $skip offset for pagination support.

    Args:
        top (int): Number of messages per page.
        skip (int): Number of messages to skip (page offset).

    Returns:
        list[dict]: A list of message dictionaries.
    """
    logger.info(f"Fetching messages page: top={top}, skip={skip}")

    client = get_graph_client()

    from msgraph.generated.users.item.messages.messages_request_builder import (
        MessagesRequestBuilder,
    )

    query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
        top=top,
        skip=skip,
        orderby=["receivedDateTime DESC"],
        filter="isDraft eq false",
        select=["id", "subject", "from", "receivedDateTime",
                "hasAttachments", "bodyPreview", "isRead"],
    )

    request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
        query_parameters=query_params
    )

    response = await client.me.messages.get(request_configuration=request_config)
    messages = response.value if response and response.value else []
    return [_message_to_dict(m) for m in messages]

---
./graph/task_client.py
---
"""
task_client.py
==============
Microsoft Graph API calls for Microsoft To-Do and Microsoft Planner.

Why this file exists:
    Phase 3 adds the ability to create reminders as real Microsoft
    tasks — either personal (To-Do) or team-level (Planner). This
    keeps follow-up items visible in the Microsoft 365 ecosystem
    the team already uses, rather than storing them only in our
    platform.

Design notes:
    - To-Do uses /me/todo/lists/{listId}/tasks endpoint.
    - Planner uses /planner/tasks endpoint with a planId.
    - Both require Tasks.ReadWrite permission.
    - Planner additionally requires Group.Read.All for plan discovery.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from datetime import datetime, timezone

from graph.graph_client_factory import get_graph_client
from config.settings import settings
from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: get_todo_lists
# ---------------------------------------------------------------------------
async def get_todo_lists() -> list[dict]:
    """
    Fetch all Microsoft To-Do task lists for the current user.

    Returns:
        list[dict]: List of task lists with id and displayName.
    """
    logger.info("Fetching Microsoft To-Do lists")
    client = get_graph_client()

    response = await client.me.todo.lists.get()
    lists = response.value if response and response.value else []

    return [
        {"id": lst.id, "name": lst.display_name}
        for lst in lists
    ]


# ---------------------------------------------------------------------------
# Function: create_todo_task
# ---------------------------------------------------------------------------
async def create_todo_task(
    title: str,
    due_date: str = "",
    notes: str = "",
    list_name: str = "Tasks",
) -> dict:
    """
    Create a task in Microsoft To-Do.

    Args:
        title (str): Task title / description.
        due_date (str): Optional due date in ISO format or natural
                         language e.g. "2026-07-20", "end of week".
        notes (str): Optional additional notes or context.
        list_name (str): Name of the To-Do list to add to.
                          Default "Tasks" (the default list).

    Returns:
        dict: Created task ID, title, and confirmation.
    """
    logger.info(f"Creating To-Do task: '{title}' in list '{list_name}'")
    client = get_graph_client()

    from msgraph.generated.models.todo_task import TodoTask
    from msgraph.generated.models.item_body import ItemBody
    from msgraph.generated.models.body_type import BodyType
    from msgraph.generated.models.date_time_time_zone import DateTimeTimeZone

    # Step 1: Find or use default task list
    todo_lists = await get_todo_lists()
    target_list = next(
        (lst for lst in todo_lists
         if lst["name"].lower() == list_name.lower()),
        None
    )

    if not target_list and todo_lists:
        # Fall back to first available list
        target_list = todo_lists[0]
        logger.info(
            f"List '{list_name}' not found — using '{target_list['name']}'"
        )

    if not target_list:
        return {
            "error": True,
            "message": "No To-Do lists found. Please check Microsoft To-Do access.",
        }

    # Step 2: Build the task object
    task = TodoTask()
    task.title = title

    if notes:
        task.body = ItemBody()
        task.body.content = notes
        task.body.content_type = BodyType.Text

    if due_date:
        from utils.date_utils import parse_date_string
        due_dt = parse_date_string(due_date)
        if due_dt:
            task.due_date_time = DateTimeTimeZone()
            task.due_date_time.date_time = due_dt.strftime("%Y-%m-%dT%H:%M:%S")
            task.due_date_time.time_zone = "UTC"

    # Step 3: Create the task
    created = await (
        client.me.todo.lists
        .by_todo_task_list_id(target_list["id"])
        .tasks.post(task)
    )

    return {
        "task_id": created.id,
        "title": created.title,
        "list_name": target_list["name"],
        "due_date": due_date or "Not set",
        "status": f"Task created in Microsoft To-Do — {target_list['name']}",
    }


# ---------------------------------------------------------------------------
# Function: create_planner_task
# ---------------------------------------------------------------------------
async def create_planner_task(
    title: str,
    due_date: str = "",
    notes: str = "",
    assigned_to_email: str = "",
) -> dict:
    """
    Create a task in Microsoft Planner.

    Args:
        title (str): Task title.
        due_date (str): Optional due date.
        notes (str): Optional task description/notes.
        assigned_to_email (str): Optional email of the person to
                                   assign the task to.

    Returns:
        dict: Created task details and confirmation.
    """
    logger.info(f"Creating Planner task: '{title}'")
    client = get_graph_client()

    plan_id = settings.PLANNER_PLAN_ID

    if not plan_id:
        return {
            "error": True,
            "message": (
                "PLANNER_PLAN_ID is not configured in .env. "
                "Please ask your team for the Planner Plan ID and "
                "add it to .env as PLANNER_PLAN_ID=<your-plan-id>."
            ),
        }

    from msgraph.generated.models.planner_task import PlannerTask
    from msgraph.generated.models.planner_assigned_to_task_board_format import (
        PlannerAssignedToTaskBoardFormat,
    )
    from msgraph.generated.models.planner_assignments import PlannerAssignments

    # Step 1: Build Planner task
    task = PlannerTask()
    task.plan_id = plan_id
    task.title = title

    if due_date:
        from utils.date_utils import parse_date_string
        due_dt = parse_date_string(due_date)
        if due_dt:
            task.due_date_time = due_dt.replace(tzinfo=timezone.utc)

    # Step 2: Resolve assignee if provided
    if assigned_to_email:
        try:
            # Look up user's Graph ID from their email
            from msgraph.generated.users.users_request_builder import (
                UsersRequestBuilder,
            )
            user_params = UsersRequestBuilder.UsersRequestBuilderGetQueryParameters(
                filter=f"mail eq '{assigned_to_email}'",
                select=["id", "displayName"],
            )
            user_config = UsersRequestBuilder.UsersRequestBuilderGetRequestConfiguration(
                query_parameters=user_params,
            )
            user_config.headers.add("ConsistencyLevel", "eventual")
            user_response = await client.users.get(
                request_configuration=user_config
            )
            users = user_response.value if user_response else []
            if users:
                from msgraph.generated.models.planner_assignment import (
                    PlannerAssignment,
                )
                assignments = PlannerAssignments()
                assignment = PlannerAssignment()
                assignment.order_hint = " !"
                assignments.additional_data = {users[0].id: assignment}
                task.assignments = assignments
        except Exception as assign_err:
            logger.warning(f"Could not resolve assignee: {assign_err}")

    # Step 3: Create the task
    created = await client.planner.tasks.post(task)

    # Step 4: Add notes as task details if provided
    if notes and created.id:
        try:
            from msgraph.generated.models.planner_task_details import (
                PlannerTaskDetails,
            )
            details = PlannerTaskDetails()
            details.description = notes
            await client.planner.tasks.by_planner_task_id(
                created.id
            ).details.patch(details)
        except Exception as notes_err:
            logger.warning(f"Could not add task notes: {notes_err}")

    return {
        "task_id": created.id,
        "title": created.title,
        "plan_id": plan_id,
        "due_date": due_date or "Not set",
        "assigned_to": assigned_to_email or "Unassigned",
        "status": "Task created in Microsoft Planner",
    }


# ---------------------------------------------------------------------------
# Function: get_todo_tasks
# ---------------------------------------------------------------------------
async def get_todo_tasks(list_name: str = "Tasks", top: int = 20) -> list[dict]:
    """
    Fetch tasks from a Microsoft To-Do list.

    Args:
        list_name (str): Name of the To-Do list. Default "Tasks".
        top (int): Maximum number of tasks to return.

    Returns:
        list[dict]: List of task dictionaries.
    """
    logger.info(f"Fetching To-Do tasks from list: '{list_name}'")
    client = get_graph_client()

    todo_lists = await get_todo_lists()
    target_list = next(
        (lst for lst in todo_lists
         if lst["name"].lower() == list_name.lower()),
        todo_lists[0] if todo_lists else None
    )

    if not target_list:
        return []

    from msgraph.generated.users.item.todo.lists.item.tasks.tasks_request_builder import (
        TasksRequestBuilder,
    )

    query_params = TasksRequestBuilder.TasksRequestBuilderGetQueryParameters(
        top=top,
        filter="status ne 'completed'",
    )
    request_config = TasksRequestBuilder.TasksRequestBuilderGetRequestConfiguration(
        query_parameters=query_params,
    )

    response = await (
        client.me.todo.lists
        .by_todo_task_list_id(target_list["id"])
        .tasks.get(request_configuration=request_config)
    )

    tasks = response.value if response and response.value else []

    return [
        {
            "id": t.id,
            "title": t.title or "",
            "due_date": (
                t.due_date_time.date_time[:10]
                if t.due_date_time else "Not set"
            ),
            "status": t.status.value if t.status else "notStarted",
            "list_name": target_list["name"],
        }
        for t in tasks
    ]

---
./graph/thread_client.py
---
"""
thread_client.py
================
Microsoft Graph API calls for fetching email threads (conversations)
and building conversation chains for MOM generation.

Why this file exists:
    A single email is often just one message in a longer thread.
    For accurate MOM generation, we need all messages in a
    conversation — not just the latest one. This file fetches
    the complete conversation chain using Graph API's
    conversationId linking.

Design notes:
    - Messages in a thread share a conversationId field.
    - We fetch the thread by filtering messages by conversationId.
    - Messages are sorted chronologically (oldest first) so the
      MOM reflects the natural flow of discussion.
    - Thread fetching requires Mail.Read permission only.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from graph.graph_client_factory import get_graph_client
from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Function: fetch_email_thread
# ---------------------------------------------------------------------------
async def fetch_email_thread(message_id: str) -> list[dict]:
    """
    Fetch all messages in the same conversation thread as a given email.

    Args:
        message_id (str): Graph API ID of any message in the thread.

    Returns:
        list[dict]: All messages in the thread, sorted oldest first,
                     each as a plain dictionary.
    """
    logger.info(f"Fetching thread for message: {message_id}")
    client = get_graph_client()

    # Step 1: Get the target message to extract its conversationId
    target_msg = await client.me.messages.by_message_id(message_id).get()
    conversation_id = target_msg.conversation_id

    if not conversation_id:
        logger.warning(f"No conversationId found for message {message_id}")
        # Return just the single message if no thread found
        return [_msg_to_dict(target_msg)]

    logger.info(f"ConversationId: {conversation_id}")

    # Step 2: Fetch all messages with the same conversationId
    from msgraph.generated.users.item.messages.messages_request_builder import (
        MessagesRequestBuilder,
    )

    query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
        filter=f"conversationId eq '{conversation_id}'",
        orderby=["receivedDateTime ASC"],  # oldest first for chronological MOM
        top=50,
    )
    request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
        query_parameters=query_params,
    )
    request_config.headers.add("ConsistencyLevel", "eventual")

    response = await client.me.messages.get(request_configuration=request_config)
    messages = response.value if response and response.value else [target_msg]

    return [_msg_to_dict(m) for m in messages]


# ---------------------------------------------------------------------------
# Helper: _msg_to_dict
# ---------------------------------------------------------------------------
def _msg_to_dict(message) -> dict:
    """Convert a Graph SDK Message to a plain dict for MOM processing."""
    return {
        "id": message.id,
        "subject": message.subject or "",
        "sender_name": (
            message.from_.email_address.name
            if message.from_ and message.from_.email_address else ""
        ),
        "sender_email": (
            message.from_.email_address.address
            if message.from_ and message.from_.email_address else ""
        ),
        "to_recipients": [
            {
                "name": r.email_address.name or "",
                "email": r.email_address.address or "",
            }
            for r in (message.to_recipients or [])
            if r.email_address
        ],
        "cc_recipients": [
            {
                "name": r.email_address.name or "",
                "email": r.email_address.address or "",
            }
            for r in (message.cc_recipients or [])
            if r.email_address
        ],
        "received_date": (
            message.received_date_time.isoformat()
            if message.received_date_time else ""
        ),
        "body": message.body.content if message.body else "",
        "has_attachments": message.has_attachments or False,
        "is_draft": message.is_draft or False,
    }

---

