# “””
test_mom_followup.py

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
“””

# —————————————————————————

# Imports

# —————————————————————————

import pytest
from unittest.mock import AsyncMock, patch
from datetime import datetime, timedelta, timezone

# ===========================================================================

# Language Utility Tests

# ===========================================================================

def test_detect_english():
“””
detect_language() should correctly identify English text.
“””
from utils.language_utils import detect_language
code, name = detect_language(
“Please find attached the quarterly report for your review.”
)
assert code == “en”
assert name == “English”

def test_detect_german():
“””
detect_language() should correctly identify German text.
“””
from utils.language_utils import detect_language
code, name = detect_language(
“Sehr geehrte Damen und Herren, anbei finden Sie den Bericht.”
)
assert code == “de”
assert name == “German”

def test_short_text_defaults_to_english():
“””
detect_language() should return English for text that is
too short for reliable detection.
“””
from utils.language_utils import detect_language
code, name = detect_language(“Hi”)
assert code == “en”
assert name == “English”

def test_empty_text_defaults_to_english():
“””
detect_language() should return English for empty string.
“””
from utils.language_utils import detect_language
code, name = detect_language(””)
assert code == “en”
assert name == “English”

def test_language_instruction_english():
“””
get_language_instruction() should return a short string for English.
“””
from utils.language_utils import get_language_instruction
instruction = get_language_instruction(“en”, “English”)
assert instruction == “Write in English.”

def test_language_instruction_german():
“””
get_language_instruction() should return a German-specific
instruction string for code ‘de’.
“””
from utils.language_utils import get_language_instruction
instruction = get_language_instruction(“de”, “German”)
assert “German” in instruction

# ===========================================================================

# MOM Generation Tests

# ===========================================================================

def _make_fake_thread():
“”“Build a minimal fake email thread for testing.”””
return [
{
“id”: “msg-001”,
“subject”: “Project Kickoff Meeting”,
“sender_name”: “John Smith”,
“sender_email”: “john@company.com”,
“to_recipients”: [
{“name”: “Jane Doe”, “email”: “jane@company.com”}
],
“cc_recipients”: [],
“received_date”: “2026-07-01T09:00:00+00:00”,
“body”: (
“Hi team, let us kick off the project. “
“Action: John to prepare the timeline by Friday. “
“Decision: We will use Python for the backend.”
),
“has_attachments”: False,
“is_draft”: False,
},
{
“id”: “msg-002”,
“subject”: “Re: Project Kickoff Meeting”,
“sender_name”: “Jane Doe”,
“sender_email”: “jane@company.com”,
“to_recipients”: [
{“name”: “John Smith”, “email”: “john@company.com”}
],
“cc_recipients”: [],
“received_date”: “2026-07-01T10:00:00+00:00”,
“body”: “Agreed. I will handle the design mockups by next Wednesday.”,
“has_attachments”: False,
“is_draft”: False,
},
]

@pytest.mark.asyncio
async def test_generate_mom_from_thread():
“””
generate_mom() with use_thread=True should return a structured
instruction containing MOM format, attendees, and email digest.
“””
fake_thread = _make_fake_thread()

```
with patch(
    "tools.mom_tools.fetch_email_thread",
    new=AsyncMock(return_value=fake_thread)
):
    from tools.mom_tools import generate_mom
    result = await generate_mom(
        email_id="msg-001",
        use_thread=True,
    )

assert result["email_count"] == 2
assert "john@company.com" in result["participants"]
assert "jane@company.com" in result["participants"]
assert result["meeting_subject"] == "Project Kickoff Meeting"
assert "MINUTES OF MEETING" in result["instruction"]
assert "ATTENDEES" in result["instruction"]
assert "ACTION ITEMS" in result["instruction"]
assert "DECISIONS MADE" in result["instruction"]
```

@pytest.mark.asyncio
async def test_generate_mom_from_single_email():
“””
generate_mom() with use_thread=False should use only the
single email and label source as ‘Single Email’.
“””
fake_single = {
“id”: “msg-001”,
“subject”: “Team Sync Notes”,
“from”: {
“emailAddress”: {
“name”: “Manager”,
“address”: “mgr@company.com”
}
},
“receivedDateTime”: “2026-07-01T09:00:00+00:00”,
“body”: {“content”: “We discussed the roadmap and agreed on Q3 targets.”},
“hasAttachments”: False,
“bodyPreview”: “We discussed the roadmap…”,
}

```
with patch(
    "tools.mom_tools.fetch_message_by_id",
    new=AsyncMock(return_value=fake_single)
):
    from tools.mom_tools import generate_mom
    result = await generate_mom(
        email_id="msg-001",
        use_thread=False,
    )

assert result["email_count"] == 1
assert result["source"] == "Single Email"
assert "MINUTES OF MEETING" in result["instruction"]
```

@pytest.mark.asyncio
async def test_generate_mom_handles_error():
“””
generate_mom() should return a clean error dict if the
Graph API call fails.
“””
with patch(
“tools.mom_tools.fetch_email_thread”,
new=AsyncMock(side_effect=Exception(“Graph API error”))
):
from tools.mom_tools import generate_mom
result = await generate_mom(email_id=“msg-001”)

```
assert result.get("error") is True
assert "message" in result
```

@pytest.mark.asyncio
async def test_saves_mom_draft_successfully():
“””
save_mom_as_draft() should call create_draft and return
a confirmation with draft ID.
“””
fake_draft_result = {
“draft_id”: “draft-mom-001”,
“subject”: “MOM — Project Kickoff”,
“to”: [“john@company.com”, “jane@company.com”],
“status”: “Draft saved to Drafts folder”,
“web_link”: “”,
}

```
with patch(
    "tools.mom_tools.create_draft",
    new=AsyncMock(return_value=fake_draft_result)
):
    from tools.mom_tools import save_mom_as_draft
    result = await save_mom_as_draft(
        to_emails="john@company.com, jane@company.com",
        subject="MOM — Project Kickoff",
        mom_content="MINUTES OF MEETING\n1. Details...",
    )

assert result["draft_id"] == "draft-mom-001"
assert "MOM" in result["instruction"]
assert "Outlook" in result["instruction"]
```

@pytest.mark.asyncio
async def test_save_mom_returns_error_for_empty_recipients():
“””
save_mom_as_draft() should return an error when no
recipients are provided.
“””
from tools.mom_tools import save_mom_as_draft
result = await save_mom_as_draft(
to_emails=””,
subject=“MOM”,
mom_content=“Content here”,
)

```
assert result.get("error") is True
assert "recipient" in result["message"].lower()
```

# ===========================================================================

# Follow-up Tracking Tests

# ===========================================================================

def _make_old_email(days_old: int = 5) -> dict:
“”“Create a fake email that is N days old.”””
received = (
datetime.now(timezone.utc) - timedelta(days=days_old)
).isoformat()
return {
“id”: “msg-old-001”,
“subject”: “Pending Approval Request”,
“from”: {
“emailAddress”: {
“name”: “External Sender”,
“address”: “external@other.com”
}
},
“receivedDateTime”: received,
“hasAttachments”: False,
“bodyPreview”: “Please approve the attached request.”,
“is_draft”: False,
}

@pytest.mark.asyncio
async def test_identifies_overdue_emails():
“””
track_followups() should flag emails older than the threshold
that were not sent by the current user.
“””
fake_profile = {
“mail”: “me@company.com”,
“userPrincipalName”: “me@company.com”,
}
fake_messages = [_make_old_email(days_old=5)]

```
with patch(
    "tools.followup_tools.get_user_profile",
    new=AsyncMock(return_value=fake_profile)
), patch(
    "tools.followup_tools.fetch_recent_messages",
    new=AsyncMock(return_value=fake_messages)
):
    from tools.followup_tools import track_followups
    result = await track_followups(days_threshold=3, count=10)

assert result["followup_count"] == 1
assert "Pending Approval Request" in result["display_table"]
```

@pytest.mark.asyncio
async def test_returns_clear_when_no_followups():
“””
track_followups() should return a positive confirmation
when all emails are recent.
“””
fake_profile = {
“mail”: “me@company.com”,
“userPrincipalName”: “me@company.com”,
}
fake_messages = [_make_old_email(days_old=1)]

```
with patch(
    "tools.followup_tools.get_user_profile",
    new=AsyncMock(return_value=fake_profile)
), patch(
    "tools.followup_tools.fetch_recent_messages",
    new=AsyncMock(return_value=fake_messages)
):
    from tools.followup_tools import track_followups
    result = await track_followups(days_threshold=3, count=10)

assert result["followup_count"] == 0
assert "No follow-ups" in result["display_table"]
```

@pytest.mark.asyncio
async def test_track_followups_handles_empty_inbox():
“””
track_followups() should handle an empty inbox gracefully.
“””
fake_profile = {
“mail”: “me@company.com”,
“userPrincipalName”: “me@company.com”,
}

```
with patch(
    "tools.followup_tools.get_user_profile",
    new=AsyncMock(return_value=fake_profile)
), patch(
    "tools.followup_tools.fetch_recent_messages",
    new=AsyncMock(return_value=[])
):
    from tools.followup_tools import track_followups
    result = await track_followups()

assert result["followup_count"] == 0
```

@pytest.mark.asyncio
async def test_detects_replied_email():
“””
check_email_replied() should correctly detect when the
current user has sent a reply in the thread.
“””
fake_profile = {
“mail”: “me@company.com”,
“userPrincipalName”: “me@company.com”,
}
fake_thread = [
{
“id”: “msg-001”,
“subject”: “Budget Request”,
“sender_name”: “John”,
“sender_email”: “john@company.com”,
“to_recipients”: [],
“cc_recipients”: [],
“received_date”: “2026-07-01T09:00:00+00:00”,
“body”: “Please approve the budget.”,
“has_attachments”: False,
“is_draft”: False,
},
{
“id”: “msg-002”,
“subject”: “Re: Budget Request”,
“sender_name”: “Me”,
“sender_email”: “me@company.com”,
“to_recipients”: [],
“cc_recipients”: [],
“received_date”: “2026-07-02T09:00:00+00:00”,
“body”: “Approved.”,
“has_attachments”: False,
“is_draft”: False,
},
]

```
with patch(
    "tools.followup_tools.get_user_profile",
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
```

@pytest.mark.asyncio
async def test_detects_unreplied_email():
“””
check_email_replied() should correctly detect when no reply
has been sent by the current user.
“””
fake_profile = {
“mail”: “me@company.com”,
“userPrincipalName”: “me@company.com”,
}
fake_thread = [
{
“id”: “msg-001”,
“subject”: “Pending Request”,
“sender_name”: “John”,
“sender_email”: “john@company.com”,
“to_recipients”: [],
“cc_recipients”: [],
“received_date”: “2026-06-25T09:00:00+00:00”,
“body”: “Please confirm.”,
“has_attachments”: False,
“is_draft”: False,
}
]

```
with patch(
    "tools.followup_tools.get_user_profile",
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
```

@pytest.mark.asyncio
async def test_compose_followup_returns_instruction_with_sender():
“””
compose_followup() should return an instruction containing
the original sender’s name and email.
“””
fake_email = {
“id”: “msg-001”,
“subject”: “Quarterly Review”,
“from”: {
“emailAddress”: {
“name”: “Hans Mueller”,
“address”: “hans@company.de”
}
},
“receivedDateTime”: “2026-06-28T09:00:00+00:00”,
“body”: {
“content”: (
“Dear colleague, please send the quarterly figures “
“at your earliest convenience.”
)
},
“hasAttachments”: False,
}

```
with patch(
    "tools.followup_tools.fetch_message_by_id",
    new=AsyncMock(return_value=fake_email)
):
    from tools.followup_tools import compose_followup
    result = await compose_followup(email_id="msg-001")

assert "Hans Mueller" in result["instruction"]
assert "hans@company.de" in result["instruction"]
assert result["original_subject"] == "Quarterly Review"
assert "instruction" in result
```

# ===========================================================================

# Task Tool Tests

# ===========================================================================

@pytest.mark.asyncio
async def test_creates_todo_task_successfully():
“””
add_task_todo() should call create_todo_task and return
a confirmation with task details.
“””
fake_result = {
“task_id”: “todo-task-001”,
“title”: “Follow up on budget email”,
“list_name”: “Tasks”,
“due_date”: “2026-07-18”,
“status”: “Task created in Microsoft To-Do — Tasks”,
}

```
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
assert "✅" in result["instruction"]
```

@pytest.mark.asyncio
async def test_todo_task_handles_creation_error():
“””
add_task_todo() should return a clean error dict if the
Graph API call fails.
“””
with patch(
“tools.followup_tools.create_todo_task”,
new=AsyncMock(side_effect=Exception(“Tasks API unavailable”))
):
from tools.followup_tools import add_task_todo
result = await add_task_todo(title=“Test task”)

```
assert result.get("error") is True
assert "message" in result
```

@pytest.mark.asyncio
async def test_planner_returns_error_when_plan_id_missing():
“””
add_task_planner() should return a clear error when
PLANNER_PLAN_ID is not configured.
“””
fake_result = {
“error”: True,
“message”: “PLANNER_PLAN_ID is not configured in .env.”,
}

```
with patch(
    "tools.followup_tools.create_planner_task",
    new=AsyncMock(return_value=fake_result)
):
    from tools.followup_tools import add_task_planner
    result = await add_task_planner(title="Team task")

assert result.get("error") is True
```

@pytest.mark.asyncio
async def test_creates_planner_task_successfully():
“””
add_task_planner() should return confirmation when a
Planner task is created successfully.
“””
fake_result = {
“task_id”: “planner-task-001”,
“title”: “Infrastructure review”,
“plan_id”: “plan-abc-123”,
“due_date”: “2026-07-20”,
“assigned_to”: “jane@company.com”,
“status”: “Task created in Microsoft Planner”,
}

```
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
```

@pytest.mark.asyncio
async def test_list_tasks_returns_table():
“””
list_tasks() with source=‘todo’ should return a formatted
table of pending tasks.
“””
fake_tasks = [
{
“id”: “t1”,
“title”: “Review budget proposal”,
“due_date”: “2026-07-18”,
“status”: “notStarted”,
“list_name”: “Tasks”,
},
{
“id”: “t2”,
“title”: “Send MOM to team”,
“due_date”: “Not set”,
“status”: “notStarted”,
“list_name”: “Tasks”,
},
]

```
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
```

@pytest.mark.asyncio
async def test_list_tasks_returns_empty_message():
“””
list_tasks() should return a clear message when the
task list is empty.
“””
with patch(
“tools.followup_tools.get_todo_tasks”,
new=AsyncMock(return_value=[])
):
from tools.followup_tools import list_tasks
result = await list_tasks(source=“todo”)

```
assert result["count"] == 0
assert (
    "empty" in result["display_table"].lower()
    or "No pending" in result["display_table"]
)
```