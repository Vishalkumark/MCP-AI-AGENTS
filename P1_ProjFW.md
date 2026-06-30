# Phase 1 — Project Framework Structure

**Organisation:** Fichtner GmbH & Co. KG
**Project:** Enterprise Agentic AI Platform — Outlook Email & Attachment Assistant
**Document Purpose:** Define the complete folder/file structure, the responsibility of every file, the reasoning behind this design, and the step-by-step VS Code setup procedure.
**Scope of this document:** Structure and documentation only — no code.

-----

## PART A — The Complete Folder Structure

```
agentic-ai-platform/
│
├── .env                              ← All secrets, keys, endpoints (never committed)
├── .env.example                      ← Safe template version of .env (committed)
├── .gitignore                        ← Tells Git/Azure DevOps what to ignore
├── requirements.txt                  ← Full list of Python dependencies, pinned
├── README.md                         ← Project overview, setup quick-start
│
├── config/
│   ├── __init__.py
│   └── settings.py                   ← Loads .env, exposes typed config objects
│
├── auth/
│   ├── __init__.py
│   ├── graph_auth.py                 ← Token extraction & validation logic
│   └── token_cache.py                ← In-memory token handling per request
│
├── graph/
│   ├── __init__.py
│   ├── graph_client_factory.py       ← Builds an authenticated Graph SDK client
│   ├── mail_client.py                ← All email-related Graph API calls
│   ├── calendar_client.py            ← All calendar-related Graph API calls
│   └── attachment_client.py          ← Downloads raw attachment bytes from Graph
│
├── parsers/
│   ├── __init__.py
│   ├── pdf_parser.py                 ← Extracts text from PDF attachments
│   ├── docx_parser.py                ← Extracts text from Word attachments
│   ├── pptx_parser.py                ← Extracts text from PowerPoint attachments
│   ├── xlsx_parser.py                ← Extracts text/data from Excel attachments
│   ├── ocr_fallback.py               ← OCR for scanned/image-based documents
│   └── parser_router.py              ← Picks the right parser based on file type
│
├── tools/
│   ├── __init__.py
│   ├── profile_tools.py              ← MCP tool: get_my_profile
│   ├── email_tools.py                ← MCP tools: list/read/search/summarise emails
│   ├── calendar_tools.py             ← MCP tool: list_calendar_events
│   └── attachment_tools.py           ← MCP tools: list/read/summarise attachments
│
├── llm/
│   ├── __init__.py
│   └── requesty_client.py            ← Wrapper around Requesty.AI API calls
│
├── utils/
│   ├── __init__.py
│   ├── logger.py                     ← Centralised logging configuration
│   ├── error_handler.py              ← Common error formatting for tool responses
│   └── date_utils.py                 ← Date parsing helpers (uses dateutil)
│
├── server.py                         ← FastMCP server entry point — ties everything together
│
├── tests/
│   ├── __init__.py
│   ├── test_auth.py
│   ├── test_mail_tools.py
│   ├── test_calendar_tools.py
│   ├── test_attachment_tools.py
│   └── test_parsers.py
│
└── temp_attachments/                 ← Scratch space for downloaded attachments (auto-cleaned)
    └── .gitkeep
```

-----

## PART B — Detailed Responsibility of Every Folder and File

### Root-level files

**`.env`**
Holds every secret, key, and endpoint your code needs to run — Azure credentials, Requesty.AI key, server ports, attachment limits. Nothing in the codebase ever hardcodes a secret; everything is read from here. This file lives only on your machine and is never pushed to Azure DevOps.

**`.env.example`**
An identical copy of `.env` but with all real values replaced by placeholders (e.g. `AZURE_CLIENT_ID=your-client-id-here`). This is the file that does get committed, so that anyone setting up the project (including future-you on FGSV297) knows exactly what variables are needed without ever seeing the actual secrets.

**`.gitignore`**
A plain text list of files and folders Git/Azure DevOps should never track — `.env`, `venv/`, `__pycache__/`, `temp_attachments/*`. Prevents secrets and junk files from accidentally being pushed.

**`requirements.txt`**
The exact list of Python packages and versions the project needs. Running `pip install -r requirements.txt` on any machine — your VM today, FGSV297 tomorrow — recreates the identical environment in one command.

**`README.md`**
The front door of the project. Explains what the project does, how to set it up, and how to run it, for anyone (including future-you) opening this folder cold.

-----

### `config/` — Centralised Configuration

**Purpose of this folder:** No other file in the project should ever call `os.getenv()` directly. Every setting flows through here once, in one place, so that if a variable name changes, it changes in exactly one file.

**`settings.py`**
Reads the `.env` file on startup and exposes clean, organised configuration values to the rest of the app — grouped logically (Azure settings, Graph settings, Requesty settings, server settings, attachment settings). Every other module imports from here, never from `.env` directly.

-----

### `auth/` — Authentication Handling

**Purpose of this folder:** Everything related to proving who the user is, and safely passing that identity through to Microsoft Graph, lives here and nowhere else. This isolation matters because authentication is the most security-sensitive part of the whole platform.

**`graph_auth.py`**
Responsible for two things: extracting the OAuth access token that LibreChat sends with every request, and validating that token is well-formed before it’s used. This is the file that implements the “delegated auth” model we agreed on — the token always belongs to you, the logged-in user, never a shared service account.

**`token_cache.py`**
Holds the token only for the lifetime of a single incoming request — nothing is persisted to disk. This file exists to formalise “we never store credentials on this VM,” which was a hard requirement from our architecture discussion.

-----

### `graph/` — Microsoft Graph API Communication

**Purpose of this folder:** Every single network call to Microsoft Graph happens through this folder. No tool file ever calls Graph directly — they all go through here. This means if Microsoft changes an API detail, we fix it in one place.

**`graph_client_factory.py`**
Builds a ready-to-use, authenticated Graph SDK client object using the token from `auth/`. Every other file in this folder asks this factory for a client rather than building one themselves — avoids duplicated connection logic.

**`mail_client.py`**
Contains the actual functions that talk to your Outlook mailbox: get a list of messages, get one message by ID, search messages by keyword. This file knows about Graph API mail endpoints and nothing else — it doesn’t know what an MCP tool is.

**`calendar_client.py`**
Same pattern as `mail_client.py`, but for calendar events — fetching your upcoming meetings within a date range.

**`attachment_client.py`**
Downloads the raw bytes of an email attachment from Graph API and saves it temporarily to `temp_attachments/` so the parsers can read it. This file’s only job is “get the file,” not “read the file” — that separation is intentional (see Part C, design rationale).

-----

### `parsers/` — Attachment Reading Engines

**Purpose of this folder:** Each file format gets its own dedicated, focused parser. This directly implements your requirement for PDF, DOC, PPT, Excel, and CV reading capability.

**`pdf_parser.py`**
Extracts readable text from `.pdf` files using PyMuPDF. Detects whether a PDF page has real selectable text or is just a scanned image, and flags it for OCR if needed.

**`docx_parser.py`**
Extracts text from `.docx` Word documents — paragraphs and tables. This is the same engine used whether the document is a project report or a CV submitted as a Word file.

**`pptx_parser.py`**
Extracts text from each slide of a `.pptx` PowerPoint file, preserving slide order so a summary can reference “Slide 3 mentions…”

**`xlsx_parser.py`**
Reads `.xlsx` Excel spreadsheets — sheet names, headers, and row data — and converts them into a clean text/table format the LLM can reason over.

**`ocr_fallback.py`**
Used only when a PDF or image has no extractable digital text (i.e. it’s a scanned document, like a scanned CV or signed contract). Runs OCR to pull text out of the image. This file is only invoked when the other parsers detect they have nothing to extract.

**`parser_router.py`**
The traffic controller. Looks at a file’s extension or content type and decides which parser file above should handle it. This is the single place that knows “`.pdf` goes here, `.xlsx` goes there” — no other file needs that knowledge.

-----

### `tools/` — The MCP Tool Definitions

**Purpose of this folder:** This is the only folder the AI model actually “sees.” Everything in `graph/`, `parsers/`, `auth/` is invisible plumbing — these tool files are the public menu of capabilities the agent can choose from, each with a name and description the LLM reads to decide what to call.

**`profile_tools.py`**
Defines `get_my_profile` — confirms who is logged in. Small, but useful as a connectivity sanity check.

**`email_tools.py`**
Defines `list_emails`, `read_email`, `search_emails`, and `summarise_email`. Each function here is thin — it calls into `graph/mail_client.py` to get data, then optionally calls `llm/requesty_client.py` to summarise it, and returns clean output. No Graph API or auth logic lives here directly — it’s all delegated.

**`calendar_tools.py`**
Defines `list_calendar_events`. Same thin-wrapper pattern, delegating to `graph/calendar_client.py`.

**`attachment_tools.py`**
Defines the new attachment capability: `list_attachments` (what’s attached to an email), `read_attachment` (extract its text via `parsers/parser_router.py`), and `summarise_attachment` (extract + summarise via the LLM). This is where your CV/PDF/DOC/PPT/Excel requirement becomes an actual callable AI capability.

-----

### `llm/` — Requesty.AI Communication

**Purpose of this folder:** Isolates every call to the LLM gateway in one place, so that the day Requesty.AI is replaced by native Azure OpenAI or Gemini (which your team flagged as a future possibility), this is the only folder that changes.

**`requesty_client.py`**
A thin wrapper function like “send this text, get a summary back.” Reads model name and API key from `config/settings.py`. Implements the primary/fallback model logic (GPT-4.1-nano first, Mistral-small as fallback).

-----

### `utils/` — Shared Helpers

**Purpose of this folder:** Small, reusable pieces of logic that many other files need, so they’re written once instead of copy-pasted everywhere.

**`logger.py`**
One consistent logging setup for the whole project — so every file logs in the same format, making debugging on FGSV297 later much easier.

**`error_handler.py`**
Converts raw Python exceptions (which can look messy and expose internal details) into clean, safe error messages that get shown back through LibreChat — directly addresses the Phase 1 “Definition of Done” requirement that no raw stack traces reach the UI.

**`date_utils.py`**
Wraps `python-dateutil` for parsing dates that show up in different formats across emails and calendar data, so every other file gets dates in one consistent format.

-----

### `server.py` — The Entry Point

This is the single file you actually run (`python server.py`). It does only one job: start the FastMCP server, register every tool from the `tools/` folder, and start listening on the configured host and port. It contains no business logic of its own — it’s pure wiring.

-----

### `tests/`

**Purpose of this folder:** Each test file mirrors a real file elsewhere in the project, letting you verify each piece works in isolation before testing the whole chain end-to-end through LibreChat. For example, `test_attachment_tools.py` lets you confirm attachment reading works correctly using a sample PDF, without needing LibreChat or even a live mailbox.

-----

### `temp_attachments/`

A scratch folder where attachment files are temporarily saved while being parsed, then deleted. The `.gitkeep` file is a zero-byte placeholder so the empty folder still gets created during setup (Git/Azure DevOps doesn’t track empty folders otherwise).

-----

## PART C — Why This Framework? Design Rationale

**Principle 1 — Each folder has exactly one responsibility.**
`auth/` only knows about identity. `graph/` only knows about talking to Microsoft. `parsers/` only knows about reading file formats. `tools/` only knows about exposing capabilities to the AI. This is called “separation of concerns” — when something breaks, you know exactly which folder to open, because each one does precisely one job and nothing else.

**Principle 2 — The AI only sees `tools/`, never the plumbing underneath.**
This matters specifically for Agentic AI design. The LLM reads tool names and descriptions to decide what to call — it never needs to know that `attachment_tools.py` internally calls `attachment_client.py` then `parser_router.py` then possibly `requesty_client.py`. Keeping that chain hidden behind a clean tool function is what makes the system genuinely agentic rather than a tangle of exposed internals.

**Principle 3 — Swappable by design.**
Three things were flagged in our conversations as likely to change later: the LLM gateway (Requesty.AI today, possibly Azure OpenAI tomorrow), the deployment host (your VM today, FGSV297 tomorrow), and the auth model (single-user today, multi-user later). Because `llm/`, `config/`, and `auth/` are isolated folders, each of those changes touches only one folder — not the entire codebase.

**Principle 4 — New tools are additive, not invasive.**
When Phase 2 adds “send email reply” or “create calendar event,” that’s a new function added to an existing tools file, calling a new function in the matching `graph/` client file. Nothing already working has to be touched or risk breaking. This directly satisfies your original requirement: “extensible so new tools, agents, and workflows can be added with minimal changes.”

**Principle 5 — Testable in isolation.**
Because `parsers/` doesn’t know about `auth/`, and `graph/` doesn’t know about `tools/`, you can test “does my PDF parser correctly read a CV” completely independently of “is my Microsoft login working.” This matters enormously for debugging — when something fails, the layered structure tells you almost immediately which layer to check first.

**Principle 6 — Matches how FastMCP and Microsoft Graph are meant to be used.**
This isn’t an arbitrary structure — it mirrors the natural shape of the official `msgraph-sdk-python` (client objects, separated by service) and FastMCP’s own pattern (tools as thin, declarative functions). We are not fighting either framework’s grain.

-----

## PART D — Step-by-Step Setup in VS Code (Structure Only, No Code Yet)

This section creates every folder and empty file so the skeleton exists and is ready for code in the next step.

### Step D.1 — Open the project folder in VS Code

1. Open VS Code
1. File → Open Folder
1. Create and select a new folder, e.g. `C:\Projects\agentic-ai-platform`
1. Open the integrated terminal: Terminal → New Terminal (it should default to PowerShell)

### Step D.2 — Create the virtual environment

```powershell
python -m venv venv
```

This creates an isolated Python environment inside a `venv` folder, so packages installed for this project never conflict with anything else on your VM.

### Step D.3 — Activate the virtual environment

```powershell
venv\Scripts\activate
```

Your terminal prompt should now show `(venv)` at the start of the line — confirming you’re working inside the isolated environment.

### Step D.4 — Create the full folder structure

Run this single block in the terminal (PowerShell) to create every folder at once:

```powershell
mkdir config, auth, graph, parsers, tools, llm, utils, server_root, tests, temp_attachments
```

(Note: `server.py` is a file, not a folder — it gets created in Step D.6.)

### Step D.5 — Create `__init__.py` in each Python package folder

Each of these folders needs an empty `__init__.py` file so Python recognises them as importable packages:

```powershell
New-Item config\__init__.py, auth\__init__.py, graph\__init__.py, parsers\__init__.py, tools\__init__.py, llm\__init__.py, utils\__init__.py, tests\__init__.py -ItemType File
```

### Step D.6 — Create all empty module files

```powershell
New-Item config\settings.py -ItemType File
New-Item auth\graph_auth.py, auth\token_cache.py -ItemType File
New-Item graph\graph_client_factory.py, graph\mail_client.py, graph\calendar_client.py, graph\attachment_client.py -ItemType File
New-Item parsers\pdf_parser.py, parsers\docx_parser.py, parsers\pptx_parser.py, parsers\xlsx_parser.py, parsers\ocr_fallback.py, parsers\parser_router.py -ItemType File
New-Item tools\profile_tools.py, tools\email_tools.py, tools\calendar_tools.py, tools\attachment_tools.py -ItemType File
New-Item llm\requesty_client.py -ItemType File
New-Item utils\logger.py, utils\error_handler.py, utils\date_utils.py -ItemType File
New-Item server.py -ItemType File
New-Item tests\test_auth.py, tests\test_mail_tools.py, tests\test_calendar_tools.py, tests\test_attachment_tools.py, tests\test_parsers.py -ItemType File
New-Item temp_attachments\.gitkeep -ItemType File
```

### Step D.7 — Create root-level files

```powershell
New-Item .env, .env.example, .gitignore, requirements.txt, README.md -ItemType File
```

### Step D.8 — Verify the structure

In VS Code’s left sidebar (Explorer panel), you should now see the complete folder tree exactly as shown in Part A. Every file will be empty — that’s expected and correct for this step.

### Step D.9 — Add `.gitignore` content (structure-safe, no project code yet)

Open `.gitignore` and add:

```
venv/
__pycache__/
*.pyc
.env
temp_attachments/*
!temp_attachments/.gitkeep
```

This is configuration, not code — it’s safe to add now since it only governs what Azure DevOps tracks later.

### Step D.10 — Confirm Python recognises the virtual environment in VS Code

1. Press `Ctrl+Shift+P` → type “Python: Select Interpreter”
1. Choose the interpreter located inside your `venv` folder (it will show as something like `.\venv\Scripts\python.exe`)

This ensures VS Code’s terminal, debugger, and IntelliSense all use the correct isolated Python environment for this project.

-----

## What This Step Achieves

At the end of Part D, you will have a complete, correctly named, fully organised project skeleton — every folder and file in the right place, with no code written yet. This is intentionally the pause point: you can open every file, see its name, and understand its single responsibility before any logic is written into it.

-----

*Review this framework. Once you confirm you understand the structure and are satisfied with it, let me know and I will wait for your instruction to begin writing the actual code inside these files.*