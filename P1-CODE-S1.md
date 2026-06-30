# Phase 1 — Code Section 1

**Project:** outlook-ai-agent-mcp
**Covers:** `.env`, `.env.example`, `.gitignore`, `requirements.txt`, `tools/`, `parsers/`
**Status:** Code only — do not execute until all 3 sections are delivered and reviewed

-----

## 1. `.env`

This is YOUR actual file with real values. Fill in the placeholders with your real credentials. This file is never shared or committed.

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
MCP_SERVER_NAME=outlook-ai-agent-mcp
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

-----

## 2. `.env.example`

Identical structure, safe placeholder values only. This is the file that WOULD be committed to Azure DevOps if versioning is set up later — it tells any future reader what variables exist without exposing secrets.

```env
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
```

-----

## 3. `.gitignore`

```gitignore
# Virtual environment
venv/
env/

# Python cache
__pycache__/
*.pyc
*.pyo

# Secrets — never commit
.env

# Temp attachment files — keep folder, ignore contents
temp_attachments/*
!temp_attachments/.gitkeep

# VS Code
.vscode/

# OS junk files
.DS_Store
Thumbs.db

# Logs
*.log
```

-----

## 4. `requirements.txt`

Pinned-style dependency list. Versions are left as latest-compatible (`>=`) for Phase 1 dev; we can pin exact versions before FGSV297 deployment for reproducibility.

```txt
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
```

-----

## 5. `tools/` Folder

This folder is the only layer the AI model actually sees. Every function here is intentionally thin: it validates input, delegates the real work to `graph/`, `parsers/`, or `llm/`, and returns a clean, predictable result. The docstring on every tool function is not just documentation for humans — **FastMCP uses it as the tool description the LLM reads to decide when to call it.** This is why every docstring below is written carefully and specifically.

-----

### 5.1 `tools/profile_tools.py`

```python
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
from server import mcp

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
```

-----

### 5.2 `tools/email_tools.py`

```python
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
from server import mcp

# Graph layer functions — each does exactly one Graph API operation.
from graph.mail_client import (
    fetch_recent_messages,
    fetch_message_by_id,
    search_messages_by_keyword,
)

# LLM wrapper — used only by summarise_email.
from llm.requesty_client import generate_summary

from utils.error_handler import format_tool_error
from utils.logger import get_logger

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

        # Step 2: Delegate to the Graph layer. This function returns a list
        #         of raw Graph API message objects.
        raw_messages = await fetch_recent_messages(top=safe_count)

        # Step 3: Transform raw Graph objects into the clean shape defined
        #         in this tool's docstring. This keeps the AI-facing output
        #         consistent regardless of what Graph API's raw shape looks like.
        result = []
        for msg in raw_messages:
            result.append({
                "id": msg.get("id"),
                "subject": msg.get("subject"),
                "sender": msg.get("from", {}).get("emailAddress", {}).get("name"),
                "sender_email": msg.get("from", {}).get("emailAddress", {}).get("address"),
                "received_date": msg.get("receivedDateTime"),
                "has_attachments": msg.get("hasAttachments", False),
                "preview": msg.get("bodyPreview", "")[:200],
            })

        return result

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
        # Step 1: Fetch the single full message from Graph API.
        msg = await fetch_message_by_id(email_id)

        # Step 2: Return the full body this time (unlike list_emails,
        #         which only returns a short preview).
        return {
            "subject": msg.get("subject"),
            "sender": msg.get("from", {}).get("emailAddress", {}).get("name"),
            "sender_email": msg.get("from", {}).get("emailAddress", {}).get("address"),
            "received_date": msg.get("receivedDateTime"),
            "body": msg.get("body", {}).get("content", ""),
            "has_attachments": msg.get("hasAttachments", False),
        }

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

        raw_messages = await search_messages_by_keyword(keyword, top=safe_count)

        result = []
        for msg in raw_messages:
            result.append({
                "id": msg.get("id"),
                "subject": msg.get("subject"),
                "sender": msg.get("from", {}).get("emailAddress", {}).get("name"),
                "sender_email": msg.get("from", {}).get("emailAddress", {}).get("address"),
                "received_date": msg.get("receivedDateTime"),
                "has_attachments": msg.get("hasAttachments", False),
                "preview": msg.get("bodyPreview", "")[:200],
            })

        return result

    except Exception as exc:
        return format_tool_error(exc, tool_name="search_emails", logger=logger)


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
        # Step 1: Reuse the existing Graph fetch — no duplicated logic.
        msg = await fetch_message_by_id(email_id)

        subject = msg.get("subject", "")
        sender = msg.get("from", {}).get("emailAddress", {}).get("name", "")
        body_text = msg.get("body", {}).get("content", "")

        # Step 2: Build a focused prompt for the LLM. We pass the subject
        #         and sender as context so the summary can reference them
        #         intelligently if needed.
        prompt = (
            f"Summarise the following email in 3-4 concise sentences. "
            f"Focus on key points, any action items, and deadlines.\n\n"
            f"Subject: {subject}\n"
            f"From: {sender}\n\n"
            f"Email body:\n{body_text}"
        )

        # Step 3: Call the LLM wrapper. This function internally handles
        #         primary/fallback model logic (GPT-4.1-nano -> Mistral).
        summary_text = await generate_summary(prompt)

        return {
            "subject": subject,
            "sender": sender,
            "summary": summary_text,
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="summarise_email", logger=logger)
```

-----

### 5.3 `tools/calendar_tools.py`

```python
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
from server import mcp

from graph.calendar_client import fetch_calendar_events

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.date_utils import parse_relative_date_range

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

        return result

    except Exception as exc:
        return format_tool_error(exc, tool_name="list_calendar_events", logger=logger)
```

-----

### 5.4 `tools/attachment_tools.py`

```python
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
from server import mcp

from graph.mail_client import fetch_message_by_id
from graph.attachment_client import (
    list_message_attachments,
    download_attachment,
)

from parsers.parser_router import parse_attachment

from llm.requesty_client import generate_summary

from utils.error_handler import format_tool_error
from utils.logger import get_logger

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
```

-----

## 6. `parsers/` Folder

Each file in this folder does exactly one job: turn the raw bytes of one specific file format into plain text. None of these files know anything about email, Graph API, or the LLM — they only know “given these bytes, give me text back.”

-----

### 6.1 `parsers/pdf_parser.py`

```python
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
```

-----

### 6.2 `parsers/docx_parser.py`

```python
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
```

-----

### 6.3 `parsers/pptx_parser.py`

```python
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
```

-----

### 6.4 `parsers/xlsx_parser.py`

```python
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
```

-----

### 6.5 `parsers/ocr_fallback.py`

```python
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
```

-----

### 6.6 `parsers/parser_router.py`

```python
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

        # Step 2: If the PDF had no real text layer (i.e. it's scanned),
        #         automatically fall back to OCR. The caller doesn't
        #         need to handle this — it's transparent.
        if not result["has_extractable_text"]:
            logger.info(f"'{file_name}' has no extractable text — attempting OCR")
            ocr_result = extract_text_via_ocr(file_bytes)

            return {
                "text": ocr_result["text"],
                "file_type": "pdf",
                "used_ocr": ocr_result["ocr_succeeded"],
            }

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
```

-----

*End of Section 1. Next: Section 2 covers `llm/`, `graph/`, `config/`, `auth/`.*