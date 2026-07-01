# Phase 1 — Code Section 3

**Project:** outlook-ai-agent-mcp
**Covers:** `utils/`, `tests/`, `temp_attachments/`, `server.py`
**Status:** Code only — do not execute until all 3 sections are reviewed

---

## 1. `utils/` Folder

Small, reusable pieces of logic that many other files need. Written once
here, imported everywhere — never copy-pasted across the codebase.

---

### 1.1 `utils/logger.py`

```python
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
```

---

### 1.2 `utils/error_handler.py`

```python
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
        # Authentication/token issues — tell the user to re-authenticate.
        user_message = (
            "Authentication failed. Your Microsoft session may have "
            "expired. Please re-authenticate via LibreChat's MCP "
            "server settings."
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
```

---

### 1.3 `utils/date_utils.py`

```python
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
```

---

## 2. `tests/` Folder

Each test file mirrors one real module in the project. Tests verify each
layer works in isolation — Graph calls, parsers, tools — using mocked
dependencies so no live Microsoft account or API key is needed to run them.

---

### 2.1 `tests/test_auth.py`

```python
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
```

---

### 2.2 `tests/test_mail_tools.py`

```python
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
                    "address": "h.mueller@fichtner.de"
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
    assert isinstance(result, list)
    assert len(result) == 1

    first = result[0]
    assert first["id"] == "msg-001"
    assert first["subject"] == "Q3 Budget Review"
    assert first["sender"] == "Hans Mueller"
    assert first["sender_email"] == "h.mueller@fichtner.de"
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
    called_with_top = mock_fetch.call_args.kwargs.get("top") or mock_fetch.call_args.args[0]
    assert called_with_top <= 50


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
        "from": {"emailAddress": {"name": "Anna Schmidt", "address": "a.schmidt@fichtner.de"}},
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
```

---

### 2.3 `tests/test_calendar_tools.py`

```python
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
```

---

### 2.4 `tests/test_attachment_tools.py`

```python
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
    assert "No readable text" in result["summary"]
```

---

### 2.5 `tests/test_parsers.py`

```python
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

    # get_pdf_data() returns the PDF as bytes without writing to disk.
    pdf_bytes = doc.get_pdf_data()
    doc.close()
    return pdf_bytes


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
```

---

## 3. `temp_attachments/` Folder

### 3.1 `temp_attachments/.gitkeep`

```
This file intentionally left empty.

Purpose:
    Git does not track empty folders. Without this placeholder file,
    the temp_attachments/ folder would not exist after a fresh clone
    or checkout, which would cause attachment_client.py to fail when
    it tries to write to this directory.

    The .gitignore file contains the rule:
        temp_attachments/*
        !temp_attachments/.gitkeep

    This combination means:
        - All downloaded attachment files in this folder ARE ignored
          (never committed to Azure DevOps)
        - This .gitkeep file is NOT ignored — keeping the folder tracked
          and therefore always present after a checkout
"""
```

---

## 4. `server.py` — The Entry Point

This is the only file you run to start the entire platform. It does nothing
except wire everything together — no business logic lives here.

```python
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
mcp = FastMCP(settings.MCP_SERVER_NAME)


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
    # Profile tools — get_my_profile
    from tools import profile_tools  # noqa: F401

    # Email tools — list_emails, read_email, search_emails, summarise_email
    from tools import email_tools  # noqa: F401

    # Calendar tools — list_calendar_events
    from tools import calendar_tools  # noqa: F401

    # Attachment tools — list_attachments, read_attachment, summarise_attachment
    from tools import attachment_tools  # noqa: F401

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
async def health_check(request) -> dict:
    """
    Simple health check endpoint. Returns server status and the list of
    registered tools. Not an MCP tool — a plain HTTP GET endpoint.

    Test with:
        curl http://localhost:8000/health
    """
    tool_names = [tool.name for tool in mcp.list_tools()]

    return {
        "status": "healthy",
        "server": settings.MCP_SERVER_NAME,
        "transport": settings.MCP_TRANSPORT,
        "tools_registered": len(tool_names),
        "tools": tool_names,
    }


# ---------------------------------------------------------------------------
# Step 4: Start the server.
# ---------------------------------------------------------------------------
if __name__ == "__main__":
    logger.info(f"Starting {settings.MCP_SERVER_NAME} on {settings.MCP_HOST}:{settings.MCP_PORT}")
    logger.info(f"Transport: {settings.MCP_TRANSPORT}")
    logger.info(f"MCP endpoint: http://{settings.MCP_HOST}:{settings.MCP_PORT}/mcp")
    logger.info(f"Health check: http://{settings.MCP_HOST}:{settings.MCP_PORT}/health")
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
```

---

## 5. Complete File Checklist — All 3 Sections

Use this as your final verification before running anything:

### Section 1 (delivered previously)
| File | Purpose |
|---|---|
| `.env` | Real secrets — fill in your values |
| `.env.example` | Safe template — commit this one |
| `.gitignore` | Excludes secrets and cache from version control |
| `requirements.txt` | All Python dependencies |
| `tools/profile_tools.py` | MCP tool: get_my_profile |
| `tools/email_tools.py` | MCP tools: list/read/search/summarise emails |
| `tools/calendar_tools.py` | MCP tool: list_calendar_events |
| `tools/attachment_tools.py` | MCP tools: list/read/summarise attachments |
| `parsers/pdf_parser.py` | Extract text from PDF bytes |
| `parsers/docx_parser.py` | Extract text from Word bytes |
| `parsers/pptx_parser.py` | Extract text from PowerPoint bytes |
| `parsers/xlsx_parser.py` | Extract data from Excel bytes |
| `parsers/ocr_fallback.py` | OCR for scanned/image PDFs |
| `parsers/parser_router.py` | Routes file type to correct parser |

### Section 2 (delivered previously)
| File | Purpose |
|---|---|
| `config/settings.py` | Loads .env, exposes typed config object |
| `auth/graph_auth.py` | Extracts OAuth token from request headers |
| `auth/token_cache.py` | Request-scoped in-memory token holder |
| `graph/graph_client_factory.py` | Builds authenticated Graph SDK client |
| `graph/mail_client.py` | All email Graph API calls |
| `graph/calendar_client.py` | All calendar Graph API calls |
| `graph/attachment_client.py` | Attachment listing and download |
| `llm/requesty_client.py` | Requesty.AI gateway wrapper with fallback |

### Section 3 (this document)
| File | Purpose |
|---|---|
| `utils/logger.py` | Centralised logging for the whole project |
| `utils/error_handler.py` | Safe error formatting for tool responses |
| `utils/date_utils.py` | Date range parsing for calendar tools |
| `tests/test_auth.py` | Tests for auth layer |
| `tests/test_mail_tools.py` | Tests for email tools |
| `tests/test_calendar_tools.py` | Tests for calendar tools |
| `tests/test_attachment_tools.py` | Tests for attachment tools |
| `tests/test_parsers.py` | Tests for all parsers and router |
| `temp_attachments/.gitkeep` | Keeps empty folder tracked in version control |
| `server.py` | FastMCP server entry point — ties everything together |

**Total: 32 files. All of Phase 1 is now written.**

---

*All 3 sections are complete. Review everything, then confirm when ready to proceed to execution.*
