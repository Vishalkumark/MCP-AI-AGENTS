# Phase 1 — Step-by-Step Execution Guide

**Project:** outlook-ai-agent-mcp
**Python:** 3.14.6 | **pip:** 26.1.2
**Location:** D:\PROJECTS\outlook-ai-agent-mcp
**Status:** Execute in order — do not skip steps

-----

## STAGE 1 — Install Dependencies

### Step 1.1 — Install all Python libraries

Make sure you are in the project root with `(venv)` active, then run:

```powershell
pip install -r requirements.txt
```

This will download and install all libraries defined in `requirements.txt`.
It may take 2–3 minutes — let it complete fully.

### Step 1.2 — Verify key packages installed correctly

Run each line one at a time and confirm no errors:

```powershell
python -c "import fastmcp; print('fastmcp OK')"
python -c "import msal; print('msal OK')"
python -c "import msgraph; print('msgraph OK')"
python -c "import fitz; print('pymupdf OK')"
python -c "import docx; print('python-docx OK')"
python -c "import pptx; print('python-pptx OK')"
python -c "import openpyxl; print('openpyxl OK')"
python -c "import httpx; print('httpx OK')"
python -c "import dotenv; print('python-dotenv OK')"
python -c "import dateutil; print('python-dateutil OK')"
```

**Expected output for each line:**

```
fastmcp OK
msal OK
msgraph OK
pymupdf OK
python-docx OK
python-pptx OK
openpyxl OK
httpx OK
python-dotenv OK
python-dateutil OK
```

If any line shows an error, run:

```powershell
pip install <library-name>
```

and try the check again.

-----

## STAGE 2 — Verify the .env File

### Step 2.1 — Confirm all required values are filled

Open your `.env` file in VS Code and verify these 4 required fields
have REAL values (not placeholder text):

```
AZURE_CLIENT_ID=        ← must not be empty
AZURE_CLIENT_SECRET=    ← must not be empty
AZURE_TENANT_ID=        ← must not be empty
REQUESTY_API_KEY=       ← must not be empty
```

Also confirm `AZURE_AUTHORITY` looks like this (with your real tenant ID):

```
AZURE_AUTHORITY=https://login.microsoftonline.com/your-actual-tenant-id
```

### Step 2.2 — Test that settings.py loads without errors

```powershell
python -c "from config.settings import settings; print('Settings loaded OK'); print('Server will run on:', settings.MCP_HOST + ':' + str(settings.MCP_PORT))"
```

**Expected output:**

```
Settings loaded OK
Server will run on: 0.0.0.0:8000
```

If you see `EnvironmentError: Missing required environment variable`, open
`.env` and fill in the variable it names.

-----

## STAGE 3 — Run the Tests

Tests verify every layer works correctly in isolation BEFORE you start
the live server. Run them in this order.

### Step 3.1 — Install pytest (test runner)

```powershell
pip install pytest pytest-asyncio
```

### Step 3.2 — Run the parser tests first (no network needed)

```powershell
python -m pytest tests/test_parsers.py -v
```

**Expected output:**

```
tests/test_parsers.py::test_pdf_parser_extracts_text          PASSED
tests/test_parsers.py::test_docx_parser_extracts_text         PASSED
tests/test_parsers.py::test_pptx_parser_extracts_text         PASSED
tests/test_parsers.py::test_xlsx_parser_extracts_data         PASSED
tests/test_parsers.py::test_parser_router_routes_pdf          PASSED
tests/test_parsers.py::test_parser_router_handles_unsupported_doc  PASSED
tests/test_parsers.py::test_parser_router_handles_unknown_type     PASSED
```

### Step 3.3 — Run the auth tests (no network needed)

```powershell
python -m pytest tests/test_auth.py -v
```

**Expected output:**

```
tests/test_auth.py::test_get_current_access_token_valid            PASSED
tests/test_auth.py::test_get_current_access_token_missing_header   PASSED
tests/test_auth.py::test_get_current_access_token_malformed_header PASSED
tests/test_auth.py::test_token_cache_set_and_get                   PASSED
tests/test_auth.py::test_token_cache_clear                         PASSED
```

### Step 3.4 — Run the tools tests (no network needed)

```powershell
python -m pytest tests/test_mail_tools.py tests/test_calendar_tools.py tests/test_attachment_tools.py -v
```

**Expected output:**

```
tests/test_mail_tools.py::test_list_emails_returns_correct_shape        PASSED
tests/test_mail_tools.py::test_list_emails_caps_count                   PASSED
tests/test_mail_tools.py::test_read_email_returns_body                  PASSED
tests/test_mail_tools.py::test_list_emails_handles_exception_gracefully PASSED
tests/test_calendar_tools.py::test_list_calendar_events_returns_correct_shape  PASSED
tests/test_calendar_tools.py::test_list_calendar_events_handles_exception_gracefully PASSED
tests/test_attachment_tools.py::test_list_attachments_returns_correct_shape    PASSED
tests/test_attachment_tools.py::test_read_attachment_returns_extracted_text    PASSED
tests/test_attachment_tools.py::test_summarise_attachment_handles_empty_text   PASSED
```

### Step 3.5 — Run all tests together as a final check

```powershell
python -m pytest tests/ -v
```

All tests must show `PASSED` before moving to Stage 4.

-----

## STAGE 4 — Start the MCP Server

### Step 4.1 — Start the server

```powershell
python server.py
```

**Expected output in terminal:**

```
2025-01-15 10:00:00 | INFO     | server | Starting outlook-ai-agent-mcp on 0.0.0.0:8000
2025-01-15 10:00:00 | INFO     | server | Transport: streamable-http
2025-01-15 10:00:00 | INFO     | server | MCP endpoint: http://0.0.0.0:8000/mcp
2025-01-15 10:00:00 | INFO     | server | Health check: http://0.0.0.0:8000/health
2025-01-15 10:00:00 | INFO     | server | Server is running. Press Ctrl+C to stop.
```

The terminal will stay open and active — this is correct. The server is now
running and listening. Do NOT close this terminal.

### Step 4.2 — Test the health check endpoint

Open a SECOND terminal in VS Code (click the + icon in the terminal panel),
activate venv, then run:

```powershell
curl http://localhost:8000/health
```

**Expected output:**

```json
{
  "status": "healthy",
  "server": "outlook-ai-agent-mcp",
  "transport": "streamable-http",
  "tools_registered": 9,
  "tools": [
    "get_my_profile",
    "list_emails",
    "read_email",
    "search_emails",
    "summarise_email",
    "list_calendar_events",
    "list_attachments",
    "read_attachment",
    "summarise_attachment"
  ]
}
```

If you see `tools_registered: 9` and all 9 tool names listed — your MCP
server is fully running and all tools are registered correctly.

### Step 4.3 — Get your VM’s IP address

In the second terminal, run:

```powershell
ipconfig
```

Look for the line that says `IPv4 Address` under your active network
adapter (usually `Ethernet adapter Ethernet` or similar). Note this IP —
you will need it for Stage 5.

Example:

```
IPv4 Address. . . . . . . . . . . : 192.168.1.45
```

Your MCP server URL for LibreChat will be:

```
http://192.168.1.45:8000/mcp
```

-----

## STAGE 5 — Connect to LibreChat

### Step 5.1 — Add your MCP server in LibreChat UI

1. Open LibreChat in your browser and log in with your corporate Microsoft
   account
1. Click the **settings/tools icon** in the sidebar
1. Navigate to **MCP Servers** or **Plugins / Tools** section
1. Click **+ Add Server** or **+ New MCP Server**
1. Fill in the following:

```
Name:       outlook-ai-agent-mcp
URL:        http://<your-vm-ip>:8000/mcp
Transport:  Streamable HTTP
```

Replace `<your-vm-ip>` with the IPv4 address from Step 4.3.

1. Click **Save** or **Create**

### Step 5.2 — Authenticate with Microsoft

After saving, LibreChat will show an **Authenticate** button next to your
MCP server entry.

1. Click **Authenticate**
1. A Microsoft login popup will open
1. Log in with your Fichtner corporate Microsoft account
1. You will be asked to grant permissions — accept
1. The popup closes and LibreChat shows a green **Connected** status

This completes the OAuth delegated auth flow. Your MCP server now has a
valid token to call Microsoft Graph on your behalf.

### Step 5.3 — Verify the connection from server logs

Switch back to the FIRST terminal (where `server.py` is running). After
you click Authenticate in LibreChat, you should see log lines appear:

```
INFO  | auth.graph_auth    | Access token extracted successfully (preview: eyJ0eXAi...Xk4Q)
INFO  | graph.graph_client_factory | Graph client created for current request
```

This confirms LibreChat is successfully sending your OAuth token to your
MCP server with each request.

-----

## STAGE 6 — End-to-End Testing in LibreChat

Open a new chat in LibreChat. Make sure your MCP server is selected/enabled
in the tools panel. Run these test prompts one at a time:

### Test 1 — Confirm identity (simplest possible test)

```
Who am I logged in as?
```

**Expected:** Your full name, email address, and job title from your
Microsoft 365 profile.

### Test 2 — List recent emails

```
Show me my last 5 emails
```

**Expected:** A list of 5 emails with subject, sender, and received date.

### Test 3 — Search emails

```
Search my emails for anything about budget
```

**Expected:** A list of emails mentioning “budget” in subject or body.

### Test 4 — Read a specific email

```
Read the full content of the first email you found
```

**Expected:** The AI uses the email ID from the previous result and returns
the full email body.

### Test 5 — Summarise an email

```
Summarise that email for me
```

**Expected:** A 3–4 sentence AI-generated summary of the email’s key points.

### Test 6 — Calendar

```
What meetings do I have this week?
```

**Expected:** A list of your calendar events for the current week.

### Test 7 — Attachment listing

```
Does my most recent email have any attachments?
```

**Expected:** Either a list of attachment names and sizes, or confirmation
there are none.

### Test 8 — Full chain (most complete test)

```
Find my latest email with a PDF attachment,
read the PDF, and give me a summary of it.
```

**Expected:** The AI chains list_emails → list_attachments →
read_attachment → summarise_attachment automatically and returns a summary
of the PDF content.

-----

## STAGE 7 — What Good Looks Like (Definition of Done)

Phase 1 is complete when ALL of the following are true:

|Check                                                     |Status|
|----------------------------------------------------------|------|
|`pip install -r requirements.txt` completes with no errors|⬜     |
|All 10 library import checks pass                         |⬜     |
|`settings.py` loads without EnvironmentError              |⬜     |
|All pytest tests pass (16+ tests, all PASSED)             |⬜     |
|`python server.py` starts cleanly                         |⬜     |
|`/health` returns 9 tools registered                      |⬜     |
|LibreChat shows green Connected status                    |⬜     |
|`get_my_profile` returns your real name and email         |⬜     |
|`list_emails` returns your real recent emails             |⬜     |
|`search_emails` returns relevant results                  |⬜     |
|`summarise_email` produces an accurate LLM summary        |⬜     |
|`list_calendar_events` returns your real meetings         |⬜     |
|`list_attachments` correctly lists email attachments      |⬜     |
|`read_attachment` extracts text from a PDF/DOCX           |⬜     |
|No raw stack traces appear in the LibreChat chat UI       |⬜     |

-----

## Troubleshooting Reference

|Symptom                                                  |Likely Cause                                        |Fix                                                                                      |
|---------------------------------------------------------|----------------------------------------------------|-----------------------------------------------------------------------------------------|
|`EnvironmentError: Missing required environment variable`|`.env` value is empty or missing                    |Open `.env`, fill in the named variable                                                  |
|`ModuleNotFoundError` on server start                    |Library not installed                               |Run `pip install -r requirements.txt` again                                              |
|`tools_registered: 0` in health check                    |Tool import failed silently                         |Check terminal for `ImportError` messages after server start                             |
|`curl` returns `Connection refused`                      |Server not running on port 8000                     |Make sure `python server.py` is still running in Terminal 1                              |
|LibreChat shows `Cannot connect to MCP server`           |Wrong IP or port in LibreChat settings              |Recheck `ipconfig` output and re-enter the URL in LibreChat                              |
|LibreChat `Authenticate` button does nothing             |LibreChat OAuth callback URL not registered in Azure|Add the callback URL to Azure Portal → App Registrations → Authentication → Redirect URIs|
|Graph API returns 401                                    |Token expired or wrong permissions                  |Click Authenticate again in LibreChat MCP settings                                       |
|Attachment read returns `unsupported_legacy`             |File is `.doc` not `.docx`                          |Ask sender to resave as `.docx` format                                                   |
|LLM summary returns error                                |Requesty.AI key invalid or model not enabled        |Check `REQUESTY_API_KEY` in `.env` and confirm models are enabled on Requesty dashboard  |

-----

*Execute Stage by Stage. Report back after each Stage with the terminal output and I will guide you through any issues before moving to the next.*