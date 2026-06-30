# Enterprise Agentic AI Platform — Phase 1 Prerequisites

**Organisation:** Fichtner GmbH & Co. KG
**Scope:** Everything required before implementation begins
**Status:** Pending Your Review — v2 (Attachments capability added)

-----

## 1. AI Model & LLM Gateway

|Item              |Value                                                                                |Status         |
|------------------|-------------------------------------------------------------------------------------|---------------|
|Gateway           |Requesty.AI                                                                          |Have access    |
|Primary model     |`azure/gpt-4.1-nano@swedencentral`                                                   |Confirmed      |
|Fallback model    |`mistral/mistral-small-2603`                                                         |Confirmed      |
|API Key           |Requesty.AI API key                                                                  |You have it    |
|Base URL          |`https://router.requesty.ai/v1`                                                      |You have it    |
|Future flexibility|Code structured so swapping to Azure OpenAI / OpenAI / Gemini is a `.env` change only|Already planned|

No further action needed here — covered in Section 6 below (`.env` structure).

-----

## 2. Python Libraries

### 2.1 Core MCP / Graph / Auth stack

|Library        |Purpose                                                   |Install Command            |
|---------------|----------------------------------------------------------|---------------------------|
|`fastmcp`      |MCP server framework                                      |`pip install fastmcp`      |
|`msal`         |Microsoft authentication (token handling)                 |`pip install msal`         |
|`msgraph-sdk`  |Official Microsoft Graph API Python SDK                   |`pip install msgraph-sdk`  |
|`python-dotenv`|Loads `.env` config into Python                           |`pip install python-dotenv`|
|`httpx`        |Async HTTP client (used by msgraph-sdk and Requesty calls)|`pip install httpx`        |
|`pydantic`     |Data validation for tool inputs/outputs                   |`pip install pydantic`     |

### 2.2 Attachment Reading — NEW (PDF, DOC/DOCX, PPT/PPTX, XLSX, CV parsing)

This is a new capability added to Phase 1: reading and extracting text from email attachments so the AI can summarise CVs, reports, slide decks, and spreadsheets sent via Outlook.

|Library                       |Purpose                                                                                         |Install Command                 |
|------------------------------|------------------------------------------------------------------------------------------------|--------------------------------|
|`pymupdf` (imported as `fitz`)|Fast, reliable PDF text extraction — also handles scanned/image-based PDF detection             |`pip install pymupdf`           |
|`python-docx`                 |Reads `.docx` Word documents (paragraphs, tables) — this is also how CVs in Word format are read|`pip install python-docx`       |
|`python-pptx`                 |Reads `.pptx` PowerPoint slides (text per slide)                                                |`pip install python-pptx`       |
|`openpyxl`                    |Reads `.xlsx` Excel spreadsheets (cells, sheets)                                                |`pip install openpyxl`          |
|`pytesseract` + `Pillow`      |OCR fallback for scanned PDFs/images with no extractable text                                   |`pip install pytesseract pillow`|

**A note on legacy formats:** `.doc` and `.ppt` (the old pre-2007 binary formats, without the `x`) are not reliably readable by the libraries above. If your organisation’s emails commonly contain these older formats, we will need `LibreOffice` installed on the host machine as a conversion step (`.doc` → `.docx`) before reading. This is a system-level install, not a `pip` package — flagging it now so it’s not a surprise later. Let me know if old-format `.doc`/`.ppt` attachments are common in your inbox; if rare, we skip this and handle it as an edge case with a clear “unsupported format” message instead.

**A note on Tesseract OCR:** `pytesseract` is a Python wrapper — the actual OCR engine (`Tesseract`) is a separate system-level install, not a `pip` package.

- Windows: download installer from `https://github.com/UB-Mannheim/tesseract/wiki`
- This is only needed as a fallback for scanned/image-only PDFs (e.g. a CV that was scanned rather than typed). If most attachments are native digital files, this can be deferred.

### 2.3 Why specialised libraries instead of one all-in-one parser

There are heavier “do everything” document parsers available (e.g. `unstructured`, `docling`) that handle all formats through one interface. We are deliberately not using them for Phase 1: they pull in large dependency trees (some require GBs of ML models), which is unnecessary weight for a single-user pilot and harder to deploy cleanly on FGSV297 later. The libraries listed above are small, fast, and each does exactly one job — easier to install, debug, and reason about. We can revisit a unified parser in a later phase if attachment variety grows significantly.

### 2.4 Full install command for Phase 1

```bash
pip install fastmcp msal msgraph-sdk python-dotenv httpx pydantic pymupdf python-docx python-pptx openpyxl pytesseract pillow
```

We will also generate a `requirements.txt` file so this becomes one command (`pip install -r requirements.txt`) on any future machine, including FGSV297.

-----

## 3. About the Code Snippet You Pasted

The Python imports you pasted (`pystache`, `openpyxl.styles`, `fastmcp.server.middleware`, `unittest.mock`, etc.) look like they’re from an existing FastMCP server’s internals — possibly a reference implementation or example you came across — rather than a list of libraries to newly install. A few notes:

- `openpyxl` — already included above (Section 2.2) for Excel reading. Good catch, it’s genuinely needed.
- `pystache` — a templating library (Mustache templates), used for generating formatted output (e.g. MOM documents). Not needed in Phase 1 (no document generation yet) — relevant for Phase 3.
- `fastmcp.server.middleware`, `fastmcp.server.dependencies`, `fastmcp.exceptions` — these aren’t separate installs, they’re internal FastMCP modules, included automatically when you `pip install fastmcp`.
- `unittest.mock` — built into Python standard library, no install needed. Used for writing tests (mocking API calls).
- `dateutil` — useful utility for flexible date parsing (e.g. parsing dates in different formats from emails). Worth adding now since calendar tools will need it.

**One addition based on this:** `python-dateutil` is genuinely useful for Phase 1 calendar tools. Adding it to the install list.

```bash
pip install python-dateutil
```

If this snippet came from a specific reference repo you want us to model our server structure after, let me know which one and I’ll review it properly before we write `server.py`.

-----

## 4. Licenses

|Item                                |License Type                   |Cost                                    |Status                                                                                           |
|------------------------------------|-------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------|
|FastMCP                             |Apache 2.0 (open source)       |Free                                    |No action                                                                                        |
|msal / msgraph-sdk                  |MIT (open source)              |Free                                    |No action                                                                                        |
|pymupdf                             |AGPL / commercial dual license |Free for internal, non-redistributed use|Confirm with your legal/IT team if AGPL is acceptable for internal enterprise use — flagged below|
|python-docx / python-pptx / openpyxl|MIT / open source              |Free                                    |No action                                                                                        |
|pytesseract / Tesseract OCR         |Apache 2.0                     |Free                                    |No action                                                                                        |
|Python                              |PSF License (open source)      |Free                                    |No action                                                                                        |
|Microsoft 365 / Graph API access    |Org-provided enterprise license|Already covered by org                  |No action                                                                                        |
|Requesty.AI                         |Org subscription               |Already covered by org                  |No action — already confirmed                                                                    |
|LibreChat                           |MIT (open source)              |Free                                    |No action, team manages it                                                                       |

**Flag on pymupdf licensing:** PyMuPDF is distributed under AGPL-3.0 by default, with a commercial license available from Artifex if AGPL terms aren’t suitable for your organisation (AGPL has implications if software is distributed/networked — typically not an issue for purely internal tooling, but worth a quick check with whoever handles open-source licensing compliance at Fichtner before production deployment). Not a blocker for local development and testing.

-----

## 5. Endpoints & URLs

|Endpoint                                    |URL                                                              |Purpose                                       |
|--------------------------------------------|-----------------------------------------------------------------|----------------------------------------------|
|Microsoft Graph API                         |`https://graph.microsoft.com/v1.0/`                              |All Outlook/Calendar/Attachment data calls    |
|Microsoft identity platform (token endpoint)|`https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token`|OAuth token exchange                          |
|Requesty.AI Gateway                         |`https://router.requesty.ai/v1`                                  |LLM inference calls                           |
|Your FastMCP server (local dev)             |`http://localhost:8000/mcp`                                      |Local testing before deployment               |
|Your FastMCP server (future, FGSV297)       |`http://<fgsv297-address>:8000/mcp`                              |Pending IP/domain from team                   |
|LibreChat OAuth callback                    |`https://<librechat-domain>/api/mcp/outlook-ai/oauth/callback`   |Needed in Azure App Registration redirect URIs|

### Still missing

- **LibreChat domain** — ask your team for the exact LibreChat URL. Needed to register the OAuth callback.
- **FGSV297 address** — ask your team once SSH key is ready.

-----

## 6. The `.env` File — Neatly Categorised

This is the single source of truth for every secret, key, and endpoint. Nothing is hardcoded anywhere else in the codebase — every module reads from this file via `python-dotenv`. This file is never committed to Azure DevOps (add it to `.gitignore` / exclude from check-in).

```env
# =================================================================
# AZURE / ENTRA ID — APP REGISTRATION (Authentication)
# =================================================================
AZURE_CLIENT_ID=your-client-id-here
AZURE_CLIENT_SECRET=your-client-secret-here
AZURE_TENANT_ID=your-tenant-id-here
AZURE_AUTHORITY=https://login.microsoftonline.com/your-tenant-id-here

# =================================================================
# MICROSOFT GRAPH API
# =================================================================
GRAPH_API_BASE_URL=https://graph.microsoft.com/v1.0
GRAPH_API_SCOPES=Mail.Read,Calendars.Read,User.Read

# =================================================================
# REQUESTY.AI — LLM GATEWAY
# =================================================================
REQUESTY_API_KEY=your-requesty-api-key-here
REQUESTY_BASE_URL=https://router.requesty.ai/v1
REQUESTY_PRIMARY_MODEL=azure/gpt-4.1-nano@swedencentral
REQUESTY_FALLBACK_MODEL=mistral/mistral-small-2603

# =================================================================
# FASTMCP SERVER CONFIG
# =================================================================
MCP_HOST=0.0.0.0
MCP_PORT=8000
MCP_SERVER_NAME=outlook-ai
MCP_TRANSPORT=streamable-http

# =================================================================
# LIBRECHAT INTEGRATION
# =================================================================
LIBRECHAT_DOMAIN=https://your-librechat-domain-here
LIBRECHAT_OAUTH_CALLBACK=${LIBRECHAT_DOMAIN}/api/mcp/outlook-ai/oauth/callback

# =================================================================
# ATTACHMENT PROCESSING
# =================================================================
MAX_ATTACHMENT_SIZE_MB=25
ATTACHMENT_TEMP_DIR=./temp_attachments
OCR_ENABLED=false
TESSERACT_CMD_PATH=

# =================================================================
# FUTURE / DEPLOYMENT (FGSV297) — fill in once provisioned
# =================================================================
DEPLOY_HOST=
DEPLOY_SSH_KEY_PATH=
DEPLOY_PYTHON_PATH=

# =================================================================
# LOGGING
# =================================================================
LOG_LEVEL=INFO
```

**Why this structure matters:** Every section has a single responsibility. When Requesty.AI is later swapped for native Azure OpenAI, only the “REQUESTY.AI” block changes — nothing else in the code touches it, because every module reads these values by name, never hardcoded.

-----

## 7. Access & Credentials Required

|Access Item                                                 |Status                       |How to Get It                                                   |
|------------------------------------------------------------|-----------------------------|----------------------------------------------------------------|
|Azure App Registration — Client ID, Tenant ID, Client Secret|Already have                 |Azure Portal -> App Registrations -> your app                   |
|Graph API Permissions — Mail.Read, Calendars.Read, User.Read|Already granted              |Confirmed via screenshot                                        |
|Graph API Permissions — Mail.ReadWrite, Mail.Send           |Configured, not yet consented|Ask admin to click “Grant admin consent” (Phase 2, not blocking)|
|LibreChat login + MCP Settings access                       |Have                         |No action                                                       |
|Requesty.AI API key + Base URL                              |Have                         |No action — confirmed by you                                    |
|SSH key for FGSV297                                         |Pending from team            |Not a Phase 1 blocker                                           |
|FGSV297 server address                                      |Pending from team            |Ask once SSH key ready                                          |
|Python on FGSV297                                           |Unconfirmed                  |Ask team, or request install when SSH access granted            |

-----

## 8. Local Development Environment (Your Windows VM)

|Item                                                           |Status                  |How to Get It                                                   |
|---------------------------------------------------------------|------------------------|----------------------------------------------------------------|
|Python 3.11+                                                   |Confirm installed       |`https://www.python.org/downloads/` — check “Add Python to PATH”|
|VS Code                                                        |Confirm installed       |`https://code.visualstudio.com/`                                |
|curl                                                           |Built into Windows 11   |No action                                                       |
|Tesseract OCR (only if scanned attachments expected)           |Optional                |`https://github.com/UB-Mannheim/tesseract/wiki`                 |
|LibreOffice (only if legacy `.doc`/`.ppt` attachments expected)|Optional                |`https://www.libreoffice.org/download/download/`                |
|Git                                                            |Not required for Phase 1|Skipped per your direction                                      |

-----

## 9. Summary — What You Need To Do Before Step 4 (Implementation)

1. Confirm Python 3.11+ is installed on your Windows VM (or install it)
1. Confirm whether scanned/image-based attachments are common in your inbox (determines if Tesseract OCR setup is needed now or can be deferred)
1. Confirm whether legacy `.doc`/`.ppt` (non-x) attachments are common (determines if LibreOffice conversion step is needed now or can be deferred)
1. Get the exact LibreChat domain/URL from your team (for OAuth callback registration)
1. Flag pymupdf’s AGPL license to whoever handles open-source compliance at Fichtner, for awareness before production deployment

Everything else (SSH key, FGSV297 address, admin consent for Mail.Send/ReadWrite) is not a blocker. Phase 1 — including attachment reading — can be fully built and tested locally on your Windows VM without them.

-----

Review this list, confirm or amend, and let me know once ready. I will wait for your Step 4 instruction before proceeding.