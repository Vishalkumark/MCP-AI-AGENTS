# Phase 1 — Code Section 2

**Project:** outlook-ai-agent-mcp
**Covers:** `llm/`, `graph/`, `config/`, `auth/`
**Status:** Code only — do not execute until all 3 sections are delivered and reviewed

---

## 1. `config/` Folder

This is the only folder allowed to read directly from `.env`. Every other file in the project imports configuration values from here, never from `os.getenv()` directly. This means if a variable name ever changes, exactly one file needs updating.

---

### 1.1 `config/settings.py`

```python
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
        self.MCP_HOST: str = _optional_env("MCP_HOST", "0.0.0.0")
        self.MCP_PORT: int = int(_optional_env("MCP_PORT", "8000"))
        self.MCP_SERVER_NAME: str = _optional_env(
            "MCP_SERVER_NAME", "outlook-ai-agent-mcp"
        )
        self.MCP_TRANSPORT: str = _optional_env(
            "MCP_TRANSPORT", "streamable-http"
        )

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


# ---------------------------------------------------------------------------
# Step 6: Create ONE shared instance, imported everywhere else as:
#         from config.settings import settings
# ---------------------------------------------------------------------------
settings = Settings()
```

---

## 2. `auth/` Folder

Everything related to proving identity and safely handling the OAuth token lives here, isolated from the rest of the codebase.

---

### 2.1 `auth/graph_auth.py`

```python
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

    # Step 2: Look for the standard Authorization header. HTTP headers
    #         are case-insensitive, but we check the conventional
    #         casing first, then fall back to lowercase as a safety net
    #         depending on how the headers dict is populated.
    auth_header = headers.get("Authorization") or headers.get("authorization")

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
```

---

### 2.2 `auth/token_cache.py`

```python
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
```

---

## 3. `graph/` Folder

Every network call to Microsoft Graph happens through this folder, and nowhere else in the project.

---

### 3.1 `graph/graph_client_factory.py`

```python
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
        return AccessToken(self._token, int(9999999999))


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
```

---

### 3.2 `graph/mail_client.py`

```python
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

logger = get_logger(__name__)


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
        "bodyPreview": message.body_preview or "",
        "body": {
            "content": message.body.content if message.body else "",
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
```

---

### 3.3 `graph/calendar_client.py`

```python
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
```

---

### 3.4 `graph/attachment_client.py`

```python
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
    content_bytes_b64 = attachment.content_bytes
    file_bytes = base64.b64decode(content_bytes_b64)

    logger.info(f"Downloaded '{file_name}' ({size_mb:.1f}MB) successfully")

    return file_bytes, file_name, content_type
```

---

## 4. `llm/` Folder

Isolates every call to the LLM gateway in one place. This is the folder that changes if Requesty.AI is ever replaced by a different provider.

---

### 4.1 `llm/requesty_client.py`

```python
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
        "max_tokens": 500,
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
```

---

*End of Section 2. Next: Section 3 covers `utils/`, `tests/`, `temp_attachments/`, and `server.py`.*
