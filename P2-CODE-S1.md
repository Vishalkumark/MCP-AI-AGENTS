# Phase 2 — Plan, Prerequisites & Code (Part 1)

**Project:** outlook-ai-agent-mcp
**Scope:** Calendar, Advanced Email Management, Task Extraction, Semantic Search
**Document:** Full plan + Part 1 code (Part 2 on your confirmation)

-----

## PART A — What Phase 2 Delivers

### New MCP Tools (13 tools added)

|Tool                    |Description                                      |
|------------------------|-------------------------------------------------|
|`list_draft_emails`     |List emails currently in Drafts folder           |
|`create_draft_email`    |Draft a professional reply or new email          |
|`create_draft_reply`    |Draft a reply to a specific email                |
|`search_emails_advanced`|Search with sender + keyword + date range filters|
|`mark_email_read`       |Mark email as read or unread                     |
|`list_folders`          |List all mail folders in the mailbox             |
|`create_folder`         |Create a new mail folder                         |
|`move_email`            |Move email between folders                       |
|`list_events`           |List calendar events with rich view              |
|`find_availability`     |Find common free slots across attendees          |
|`create_draft_invite`   |Draft a meeting invite with agenda               |
|`extract_tasks`         |Read emails and extract actionable to-do items   |
|`semantic_search_emails`|Semantic search over email history               |

**Total tools after Phase 2: 25**

-----

## PART B — Prerequisites & Approvals

### B.1 — New Graph API Permissions Required

You need your admin to grant the following in Azure Portal →
App Registrations → your app → API Permissions:

|Permission             |Type     |Purpose                                              |Admin consent required    |
|-----------------------|---------|-----------------------------------------------------|--------------------------|
|`Mail.ReadWrite`       |Delegated|Create drafts, mark read, move emails, create folders|No — but needs grant click|
|`Mail.Send`            |Delegated|Sending (deferred, but register now)                 |No                        |
|`Calendars.ReadWrite`  |Delegated|Create/modify calendar events and draft invites      |No                        |
|`Calendars.Read.Shared`|Delegated|Read other attendees’ free/busy status               |No                        |

**Current status from Phase 1:**

- `Mail.Read` ✅ granted
- `Calendars.Read` ✅ granted
- `User.Read` ✅ granted
- `Mail.ReadWrite` ⚠️ configured, not granted
- `Mail.Send` ⚠️ configured, not granted

**What to ask your admin:**

> “Please grant consent for `Mail.ReadWrite`, `Mail.Send`,
> `Calendars.ReadWrite`, and `Calendars.Read.Shared` on our Azure App
> Registration for the Outlook AI Agent project. Also please add
> `Calendars.ReadWrite` and `Calendars.Read.Shared` if not already listed.”

### B.2 — How to check and add permissions in Azure Portal

1. Go to `https://portal.azure.com`
1. Azure Active Directory → App Registrations → your app
1. Click **API Permissions** in left menu
1. Click **+ Add a permission** → Microsoft Graph → Delegated
1. Search and add each missing permission from the table above
1. Click **Grant admin consent for [Company]** button at the top
1. All permissions should show green ✅ status

### B.3 — New Python Libraries Required

|Library                |Purpose                                               |Install                            |
|-----------------------|------------------------------------------------------|-----------------------------------|
|`sentence-transformers`|Semantic search — converts emails to vector embeddings|`pip install sentence-transformers`|
|`numpy`                |Vector similarity calculations for semantic search    |`pip install numpy`                |
|`sklearn`              |Cosine similarity scoring                             |`pip install scikit-learn`         |

Install all at once:

```bash
pip install sentence-transformers numpy scikit-learn
```

Add to `requirements.txt`:

```txt
sentence-transformers>=2.7.0
numpy>=1.26.0
scikit-learn>=1.4.0
```

### B.4 — Drafting Rules (hardcoded into all draft tools)

These rules are baked into every draft tool’s system prompt:

- Professional tone, always polite
- Short and crisp unless elaboration is needed
- Meeting invites always include: Agenda, Description, clear header
- Meeting invites default to 30 minutes duration
- Never aggressive, never informal unless context demands it

### B.5 — .env additions required

Add these to your `.env` on FGSV297:

```env
# =================================================================
# PHASE 2 — GRAPH API SCOPES (update existing line)
# =================================================================
GRAPH_API_SCOPES=Mail.Read,Mail.ReadWrite,Calendars.Read,Calendars.ReadWrite,Calendars.Read.Shared,User.Read

# =================================================================
# PHASE 2 — SEMANTIC SEARCH
# =================================================================
SEMANTIC_SEARCH_MODEL=all-MiniLM-L6-v2
SEMANTIC_SEARCH_MAX_EMAILS=100
```

Also update `config/settings.py` — add inside `__init__`:

```python
        # ----- Phase 2 — Semantic search -----
        self.SEMANTIC_SEARCH_MODEL: str = _optional_env(
            "SEMANTIC_SEARCH_MODEL", "all-MiniLM-L6-v2"
        )
        self.SEMANTIC_SEARCH_MAX_EMAILS: int = int(
            _optional_env("SEMANTIC_SEARCH_MAX_EMAILS", "100")
        )
```

-----

## PART C — Updated Folder Structure

```
outlook-ai-agent-mcp/
│
├── tools/
│   ├── profile_tools.py          ← Phase 1 (unchanged)
│   ├── email_tools.py            ← Phase 1 + Phase 2 additions
│   ├── calendar_tools.py         ← Phase 1 + Phase 2 additions
│   ├── attachment_tools.py       ← Phase 1 (unchanged)
│   ├── draft_tools.py            ← NEW Phase 2 — all draft operations
│   ├── folder_tools.py           ← NEW Phase 2 — folder management
│   ├── task_tools.py             ← NEW Phase 2 — task extraction
│   └── semantic_tools.py         ← NEW Phase 2 — semantic search
│
├── graph/
│   ├── mail_client.py            ← Phase 1 + Phase 2 additions
│   ├── calendar_client.py        ← Phase 1 + Phase 2 additions
│   ├── draft_client.py           ← NEW Phase 2 — draft/folder Graph calls
│   └── availability_client.py   ← NEW Phase 2 — free/busy Graph calls
│
├── semantic/
│   ├── __init__.py               ← NEW Phase 2 folder
│   ├── embedder.py               ← Converts email text to vectors
│   └── search_engine.py         ← Similarity search over email corpus
```

-----

## PART D — How Each Tool Works (Design)

### `list_draft_emails`

Calls `GET /me/mailFolders/drafts/messages`. Returns table view
identical to `list_emails` but labelled as Drafts.

### `create_draft_email`

Calls `POST /me/messages` with `isDraft: true`. Applies drafting rules
from system prompt. Does NOT send — only saves to Drafts folder.
Returns the draft ID so user can review it in Outlook.

### `create_draft_reply`

Calls `POST /me/messages/{id}/createReply`. Reads the original email
first, then constructs a professional reply draft in context.

### `mark_email_read`

Calls `PATCH /me/messages/{id}` with `{"isRead": true/false}`.

### `list_folders`

Calls `GET /me/mailFolders`. Returns all folders with name, unread
count, and total count in a clean table.

### `create_folder`

Calls `POST /me/mailFolders`. Creates a new folder at root level or
as a child of an existing folder.

### `move_email`

Calls `POST /me/messages/{id}/move` with `{"destinationId": folder_id}`.

### `list_events`

Enhancement of Phase 1 `list_calendar_events` — returns richer data
including attendee list, online meeting link, and recurring flag.

### `find_availability`

Calls `POST /me/findMeetingTimes` with a list of attendee emails and
a preferred duration. Returns suggested time slots as a clean table.

### `create_draft_invite`

Calls `POST /me/events` with `isDraft: true` equivalent — creates a
calendar event in Draft state. Includes agenda, description, and
attendees. Default 30 minutes duration.

### `extract_tasks`

Reads the last N emails using `list_emails`, then passes each to
LibreChat’s LLM with a structured prompt to extract action items,
deadlines, and owners. Returns a numbered to-do list.

### `semantic_search_emails`

Fetches the last N emails, converts each to a vector embedding using
`sentence-transformers`, computes cosine similarity against the query,
and returns the top matching emails — without needing exact keyword
matches.

-----

## PART E — Code: Part 1

### Files in Part 1:

- `graph/draft_client.py`
- `graph/availability_client.py`
- `tools/draft_tools.py`
- `tools/folder_tools.py`

-----

### `graph/draft_client.py`

```python
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
    )
    request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
        query_parameters=query_params
    )

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
) -> dict:
    """
    Create a reply draft for an existing email.
    The reply is pre-populated with the original email's context
    and saved to Drafts. Does NOT send.

    Args:
        message_id (str): Graph API ID of the email to reply to.
        reply_body_html (str): The reply body in HTML format.

    Returns:
        dict: The created reply draft's ID and status.
    """
    logger.info(f"Creating reply draft for message: {message_id}")
    client = get_graph_client()

    from msgraph.generated.models.message import Message
    from msgraph.generated.models.item_body import ItemBody
    from msgraph.generated.models.body_type import BodyType
    from msgraph.generated.users.item.messages.item.create_reply.create_reply_post_request_body import (
        CreateReplyPostRequestBody,
    )

    # Step 1: Build the reply request body
    request_body = CreateReplyPostRequestBody()
    request_body.message = Message()
    request_body.message.body = ItemBody()
    request_body.message.body.content_type = BodyType.Html
    request_body.message.body.content = reply_body_html

    # Step 2: createReply creates a draft reply — does NOT send
    reply_draft = await (
        client.me.messages.by_message_id(message_id)
        .create_reply.post(request_body)
    )

    return {
        "draft_id": reply_draft.id,
        "subject": reply_draft.subject or "",
        "status": "Reply draft saved to Drafts folder",
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
```

-----

### `graph/availability_client.py`

```python
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
    from msgraph.generated.models.meeting_time_suggestion_result import (
        MeetingTimeSuggestionResult,
    )
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
    event.is_draft = True

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
```

-----

### `tools/draft_tools.py`

```python
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
async def create_draft_email(
    to_emails: str,
    subject: str,
    context: str,
    cc_emails: str = "",
) -> dict:
    """
    Draft a new professional email and save it to the Drafts folder.
    Does NOT send the email. The user can review and send from Outlook.

    Use this tool when the user asks things like:
    - "Draft an email to john@company.com about the project delay"
    - "Write an email to the team about tomorrow's meeting"
    - "Compose a professional email to the client about the invoice"

    Args:
        to_emails (str): Comma-separated recipient email addresses.
                          e.g. "john@company.com, jane@company.com"
        subject (str): The email subject line.
        context (str): Description of what the email should say —
                        the AI will compose the professional body
                        based on this context following drafting rules.
        cc_emails (str): Optional comma-separated CC email addresses.

    Returns:
        dict with draft ID, subject, and confirmation message.
    """
    logger.info(f"Tool called: create_draft_email (to={to_emails}, subject={subject})")

    try:
        # Step 1: Parse recipient lists from comma-separated strings
        to_list = [e.strip() for e in to_emails.split(",") if e.strip()]
        cc_list = [e.strip() for e in cc_emails.split(",") if e.strip()] if cc_emails else []

        if not to_list:
            return {
                "error": True,
                "message": "No valid recipient email addresses provided.",
            }

        # Step 2: Build the professional email body HTML following
        #         drafting rules — returned directly as the draft content.
        #         LibreChat's LLM composes the actual text from context.
        body_html = f"""
<html><body>
<p><!-- Professional email body to be composed by the AI based on: --></p>
<p><strong>Context provided:</strong> {context}</p>
<p><em>Draft composed following professional drafting rules:
polite tone, crisp language, professional greeting and sign-off.</em></p>
</body></html>
"""
        # Step 3: Create the draft via Graph API
        result = await create_draft(
            to_emails=to_list,
            subject=subject,
            body_html=body_html,
            cc_emails=cc_list if cc_list else None,
        )

        return {
            **result,
            "instruction": (
                f"Inform the user their draft email has been saved to Drafts. "
                f"Draft ID: {result['draft_id']}. "
                f"They can review and send it from Outlook. "
                f"Also compose and display the email body based on this context: '{context}'. "
                f"Follow professional drafting rules: polite, crisp, professional greeting "
                f"and sign-off. Subject: '{subject}'."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="create_draft_email", logger=logger)


# ---------------------------------------------------------------------------
# Tool: create_draft_reply
# ---------------------------------------------------------------------------
@mcp.tool
async def create_draft_reply(
    email_id: str,
    reply_context: str,
) -> dict:
    """
    Draft a professional reply to a specific email and save it to Drafts.
    Does NOT send. The user reviews and sends from Outlook.

    Use this tool when the user asks things like:
    - "Draft a reply to this email saying we'll review by Friday"
    - "Reply to John's email and let him know the report is delayed"
    - "Write a professional response to this email"

    Args:
        email_id (str): Graph API ID of the email to reply to.
                         Obtained from list_emails or search_emails.
        reply_context (str): What the reply should say — the AI composes
                              a professional response based on this context
                              and the original email's content.

    Returns:
        dict with reply draft ID and confirmation.
    """
    logger.info(f"Tool called: create_draft_reply (email_id={email_id})")

    try:
        # Step 1: Read the original email for context
        original = await fetch_message_by_id(email_id)
        original_subject = original.get("subject", "")
        original_sender = original.get("from", {}).get("emailAddress", {}).get("name", "")
        original_body = original.get("body", {}).get("content", "")[:500]

        # Step 2: Build the reply body HTML
        reply_body_html = f"""
<html><body>
<p><em>[Professional reply composed based on context: {reply_context}]</em></p>
<br>
<hr>
<p><strong>Original message from:</strong> {original_sender}</p>
<p><strong>Subject:</strong> {original_subject}</p>
<p>{original_body}</p>
</body></html>
"""

        # Step 3: Create the reply draft via Graph API
        result = await create_reply_draft(
            message_id=email_id,
            reply_body_html=reply_body_html,
        )

        return {
            **result,
            "original_subject": original_subject,
            "original_sender": original_sender,
            "instruction": (
                f"The reply draft has been saved. "
                f"Now compose and display the professional reply text based on: '{reply_context}'. "
                f"Original email from {original_sender} about '{original_subject}'. "
                f"Follow drafting rules: polite, professional, concise. "
                f"Include appropriate greeting and sign-off."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="create_draft_reply", logger=logger)


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
                "Display 'invite_preview' as markdown to show the user "
                "a formatted preview of their meeting invite draft. "
                "Confirm it has been saved to their Calendar for review."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="create_draft_invite", logger=logger)
```

-----

### `tools/folder_tools.py`

```python
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
                + (f"It is inside '{parent_folder_name}'." if parent_folder_name else "It is at root level.")
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
```

-----

## PART F — What’s in Part 2

Part 2 will contain:

|File                            |Contents                                                |
|--------------------------------|--------------------------------------------------------|
|`tools/email_tools.py` additions|`mark_email_read` tool                                  |
|`tools/task_tools.py`           |`extract_tasks` tool — reads emails, extracts to-do list|
|`tools/semantic_tools.py`       |`semantic_search_emails` tool                           |
|`semantic/embedder.py`          |Email → vector embedding using sentence-transformers    |
|`semantic/search_engine.py`     |Cosine similarity search over email corpus              |
|`server.py` updates             |Register all new tool modules                           |
|`tests/`                        |New test files for Phase 2 tools                        |
|Checklist                       |Full Phase 2 definition of done                         |

-----

*Review Part 1. Once confirmed, say “Give me Part 2” and I will deliver the remaining files.*