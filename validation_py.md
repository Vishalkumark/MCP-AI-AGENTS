~~~python
# “””
email_tools.py

MCP tools for listing, reading, searching, summarising emails,
and managing email flags and read status.

Governance layer:
- Every tool call is audit-logged via utils/audit_logger.py
- Summarisation tools have governance rules injected via utils/governance.py
- Output validation applied via utils/validator.py before returning

Phase 1 tools:
list_emails, read_email, search_emails, summarise_email

Phase 1 enhancements:
list_emails_paged, search_emails_advanced, export_emails_markdown

Phase 2 additions:
mark_email_read, flag_email
“””

# —————————————————————————

# Imports

# —————————————————————————

from tools.mcp_instance import mcp

from graph.mail_client import (
fetch_recent_messages,
fetch_message_by_id,
search_messages_by_keyword,
fetch_messages_paged,
search_messages_advanced,
)

from utils.error_handler import format_tool_error
from utils.logger import get_logger

logger = get_logger(**name**)

# —————————————————————————

# Tool: list_emails

# —————————————————————————

@mcp.tool
async def list_emails(count: int = 10) -> dict:
“””
List the most recent emails in the user’s Outlook inbox.
Excludes drafts automatically.

```
Use this tool when the user asks things like:
- "Show me my recent emails"
- "What's in my inbox?"
- "List my last 5 emails"

Args:
    count (int): How many recent emails to return. Defaults to 10.
                  Maximum 50.

Returns:
    dict with emails list and a formatted display_table.
"""
logger.info(f"Tool called: list_emails (count={count})")

try:
    safe_count = min(max(count, 1), 50)

    # ── Audit log ────────────────────────────────────────────────
    from utils.audit_logger import log_tool_call, get_user_email_from_headers
    user_email = get_user_email_from_headers()
    log_tool_call(
        tool_name="list_emails",
        user_email=user_email,
        inputs={"count": safe_count},
    )

    # ── Fetch data ───────────────────────────────────────────────
    raw_messages = await fetch_recent_messages(top=safe_count)

    # ── Build result ─────────────────────────────────────────────
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

    # ── Build display table ───────────────────────────────────────
    table_lines = [
        "| # | Date | From | Subject | Preview | 📎 |",
        "|---|------|------|---------|---------|-----|",
    ]
    for i, m in enumerate(result, 1):
        att = "✅" if m["has_attachments"] else "—"
        subject_col = m["subject"][:50] + ("…" if len(m["subject"]) > 50 else "")
        preview_col = m["preview"][:80] + ("…" if len(m["preview"]) > 80 else "")
        table_lines.append(
            f"| {i} | {m['date_display']} | {m['sender']} "
            f"| {subject_col} | {preview_col} | {att} |"
        )

    final_result = {
        "emails": result,
        "count": len(result),
        "display_table": "\n".join(table_lines),
        "instruction": (
            "Display the 'display_table' field as a markdown table. "
            "Do not reformat or summarise it — show it exactly as given. "
            "Then offer to read, summarise, or search any specific email."
        ),
    }

    # ── Validate output ───────────────────────────────────────────
    from utils.validator import validate_email_list, append_validation_to_result
    validation = validate_email_list(result)
    return append_validation_to_result(final_result, validation)

except Exception as exc:
    return format_tool_error(exc, tool_name="list_emails", logger=logger)
```

# —————————————————————————

# Tool: read_email

# —————————————————————————

@mcp.tool
async def read_email(email_id: str) -> dict:
“””
Read the full content of a specific email by its ID.
Displays in Outlook-style inbox card view.

```
Use this tool when the user asks things like:
- "Open this email and tell me what it says"
- "Read the full content of this email"
- "What does this email say in detail?"

Args:
    email_id (str): The unique Graph API ID of the email to read.
                     Obtained from list_emails or search_emails.

Returns:
    dict with full email content and a formatted inbox_view.
"""
logger.info(f"Tool called: read_email (email_id={email_id[:20]}...)")

try:
    # ── Audit log ────────────────────────────────────────────────
    from utils.audit_logger import log_tool_call, get_user_email_from_headers
    user_email = get_user_email_from_headers()
    log_tool_call(
        tool_name="read_email",
        user_email=user_email,
        inputs={"email_id": email_id[:12]},
    )

    # ── Fetch data ───────────────────────────────────────────────
    msg = await fetch_message_by_id(email_id)

    subject = msg.get("subject") or "(no subject)"
    sender_name = msg.get("from", {}).get("emailAddress", {}).get("name") or "—"
    sender_addr = msg.get("from", {}).get("emailAddress", {}).get("address") or "—"
    received_raw = msg.get("receivedDateTime", "")
    body = msg.get("body", {}).get("content", "")
    has_att = msg.get("hasAttachments", False)

    # ── Validate email body ───────────────────────────────────────
    from utils.validator import validate_email_body, append_validation_to_result
    body_validation = validate_email_body(body, subject)

    # ── Format date ───────────────────────────────────────────────
    try:
        from dateutil import parser as dp
        dt = dp.parse(received_raw)
        date_display = dt.strftime("%A, %d %B %Y at %H:%M")
    except Exception:
        date_display = received_raw

    # ── Build Outlook-style inbox view ────────────────────────────
    inbox_view = f"""
```

-----

### 📧 {subject}

|Field      |Details                         |
|-----------|--------------------------------|
|**From**   |{sender_name} <{sender_addr}>   |
|**Date**   |{date_display}                  |
|**Subject**|{subject}                       |
|**Attach.**|{“📎 Yes” if has_att else “None”}|

-----

{body}

-----

“””

```
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
```

# —————————————————————————

# Tool: search_emails

# —————————————————————————

@mcp.tool
async def search_emails(keyword: str, count: int = 10) -> dict:
“””
Search the user’s mailbox for emails matching a keyword.

```
Use this tool when the user asks things like:
- "Find emails about the Q3 report"
- "Search for emails from John about the budget"
- "Do I have any emails mentioning the project deadline?"

Args:
    keyword (str): The search term — subject, sender, or content.
    count (int): Maximum number of matching emails to return.

Returns:
    dict with matching emails and display_table.
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

    # ── Fetch data ───────────────────────────────────────────────
    raw_messages = await search_messages_by_keyword(keyword, top=safe_count)

    # ── Build result ─────────────────────────────────────────────
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

    # ── Build display table ───────────────────────────────────────
    table_lines = [
        f"**Search Results for '{keyword}' — {len(result)} found**\n",
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
        "keyword": keyword,
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
    return format_tool_error(exc, tool_name="search_emails", logger=logger)
```

# —————————————————————————

# Tool: summarise_email

# —————————————————————————

@mcp.tool
async def summarise_email(email_id: str) -> dict:
“””
Read and return a specific email’s content ready for summarisation
by LibreChat’s LLM. The LLM summarises from the returned content —
no internal LLM call is made by this tool.

```
Use this tool when the user asks things like:
- "Summarise this email"
- "Give me the key points of this email"
- "What's the TL;DR of this message?"

Args:
    email_id (str): The unique Graph API ID of the email to summarise.

Returns:
    dict with full email content and summarisation instruction
    for LibreChat's LLM.
"""
logger.info(f"Tool called: summarise_email (email_id={email_id[:20]}...)")

try:
    # ── Audit log ────────────────────────────────────────────────
    from utils.audit_logger import log_tool_call, get_user_email_from_headers
    user_email = get_user_email_from_headers()
    log_tool_call(
        tool_name="summarise_email",
        user_email=user_email,
        inputs={"email_id": email_id[:12]},
    )

    # ── Fetch data ───────────────────────────────────────────────
    msg = await fetch_message_by_id(email_id)

    subject = msg.get("subject", "")
    sender = msg.get("from", {}).get("emailAddress", {}).get("name", "")
    sender_email = msg.get("from", {}).get("emailAddress", {}).get("address", "")
    received_date = msg.get("receivedDateTime", "")
    body_text = msg.get("body", {}).get("content", "")

    # ── Validate email body before summarising ────────────────────
    from utils.validator import validate_email_body, append_validation_to_result
    body_validation = validate_email_body(body_text, subject)
    if not body_validation.is_valid:
        return append_validation_to_result({}, body_validation)

    # ── Governance rules for summarisation ────────────────────────
    from utils.governance import get_email_rules
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
```

# —————————————————————————

# Tool: list_emails_paged

# —————————————————————————

@mcp.tool
async def list_emails_paged(count: int = 50, page: int = 1) -> dict:
“””
List emails in pages of up to 50. Use when the user wants
more than 50 emails or asks for the next page of results.

```
Use this tool when the user asks things like:
- "Show me the next 50 emails"
- "Get page 2 of my inbox"
- "List emails 51 to 100"

Args:
    count (int): Emails per page, max 50. Default 50.
    page (int): Page number starting at 1. Default 1.

Returns:
    dict with paged emails, table, and next page prompt.
"""
logger.info(f"Tool called: list_emails_paged (count={count}, page={page})")

try:
    safe_count = min(max(count, 1), 50)
    skip = (page - 1) * safe_count

    # ── Audit log ────────────────────────────────────────────────
    from utils.audit_logger import log_tool_call, get_user_email_from_headers
    user_email = get_user_email_from_headers()
    log_tool_call(
        tool_name="list_emails_paged",
        user_email=user_email,
        inputs={"count": safe_count, "page": page},
    )

    # ── Fetch data ───────────────────────────────────────────────
    raw_messages = await fetch_messages_paged(top=safe_count, skip=skip)

    # ── Build result ─────────────────────────────────────────────
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
    from utils.validator import validate_email_list, append_validation_to_result
    validation = validate_email_list(result)
    return append_validation_to_result(final_result, validation)

except Exception as exc:
    return format_tool_error(exc, tool_name="list_emails_paged", logger=logger)
```

# —————————————————————————

# Tool: search_emails_advanced

# —————————————————————————

@mcp.tool
async def search_emails_advanced(
sender_email: str = “”,
keyword: str = “”,
date_from: str = “”,
date_to: str = “”,
count: int = 20,
) -> dict:
“””
Advanced email search with sender, keyword, and date range filters.

```
Use this tool when the user asks things like:
- "Get emails from jane@company.com between June 10 and June 18"
- "Show emails from John about budget last week"
- "Find all emails received between 1 July and 7 July"

Args:
    sender_email (str): Filter by sender email address.
    keyword (str): Filter by keyword in subject or body.
    date_from (str): Start date e.g. "June 10", "2026-06-10".
    date_to (str): End date in any format.
    count (int): Maximum results to return. Default 20, max 50.

Returns:
    dict with filtered emails and display_table.
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

    # ── Fetch data ───────────────────────────────────────────────
    raw_messages = await search_messages_advanced(
        sender_email=sender_email,
        keyword=keyword,
        date_from=date_from,
        date_to=date_to,
        top=safe_count,
    )

    # ── Build result ─────────────────────────────────────────────
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

    # ── Build display table ───────────────────────────────────────
    table_lines = [
        f"**Search Results — {len(result)} email(s) found**\n",
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
```

# —————————————————————————

# Tool: export_emails_markdown

# —————————————————————————

@mcp.tool
async def export_emails_markdown(count: int = 200, keyword: str = “”) -> dict:
“””
Export a large list of emails to a downloadable markdown table.

```
Use this tool when the user asks things like:
- "Export all my emails to a file"
- "Give me all 200 emails in a markdown table"
- "Download my inbox as a table"

Args:
    count (int): Total emails to export, max 200.
    keyword (str): Optional keyword filter.

Returns:
    dict with full markdown content for export.
"""
logger.info(f"Tool called: export_emails_markdown (count={count})")

try:
    safe_count = min(max(count, 1), 200)

    # ── Audit log ────────────────────────────────────────────────
    from utils.audit_logger import log_tool_call, get_user_email_from_headers
    user_email = get_user_email_from_headers()
    log_tool_call(
        tool_name="export_emails_markdown",
        user_email=user_email,
        inputs={"count": safe_count, "keyword": keyword},
    )

    # ── Fetch in batches of 50 ───────────────────────────────────
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
            break

    # ── Build markdown table ──────────────────────────────────────
    lines = [
        "# Outlook Inbox Export",
        f"**Total emails:** {len(all_messages)}",
        f"**Filter:** {keyword if keyword else 'None'}",
        "",
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
        sender_email_addr = msg.get("from", {}).get("emailAddress", {}).get("address") or "—"
        subject = (msg.get("subject") or "(no subject)")[:60]
        att = "Yes" if msg.get("hasAttachments") else "No"

        lines.append(
            f"| {i} | {date_display} | {sender} | {sender_email_addr} "
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
```

# —————————————————————————

# Tool: mark_email_read

# —————————————————————————

@mcp.tool
async def mark_email_read(email_id: str, is_read: bool = True) -> dict:
“””
Mark a specific email as read or unread.

```
Use this tool when the user asks things like:
- "Mark this email as read"
- "Mark John's email as unread"
- "Set this message as unread so I remember to come back to it"

Args:
    email_id (str): The unique Graph API ID of the email.
    is_read (bool): True to mark as read, False for unread.

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

    from graph.draft_client import mark_message_read_status
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
```

# —————————————————————————

# Tool: flag_email

# —————————————————————————

@mcp.tool
async def flag_email(email_id: str, flag_status: str = “flagged”) -> dict:
“””
Flag an email for follow-up, mark it complete, or remove the flag.

```
Note: Pinning emails is not supported via Microsoft Graph API —
pinning is a local Outlook client feature only.

Use this tool when the user asks things like:
- "Flag this email for follow-up"
- "Mark this email as complete"
- "Remove the flag from this email"
- "Unflag this message"

Args:
    email_id (str): Graph API ID of the email to flag.
    flag_status (str): "flagged", "complete", or "notFlagged".

Returns:
    dict with confirmation of the flag status change.
"""
logger.info(
    f"Tool called: flag_email "
    f"(email_id={email_id[:20]}..., flag_status={flag_status})"
)

try:
    # ── Audit log ────────────────────────────────────────────────
    from utils.audit_logger import log_tool_call, get_user_email_from_headers
    user_email = get_user_email_from_headers()
    log_tool_call(
        tool_name="flag_email",
        user_email=user_email,
        inputs={"email_id": email_id[:12], "flag_status": flag_status},
    )

    from graph.draft_client import flag_message
    result = await flag_message(
        message_id=email_id,
        flag_status=flag_status,
    )

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
```
~~~



___


~~~python

cat > /mnt/user-data/outputs/tools_update_batch1.md << 'ENDOFFILE'
# Tools Update — Guardrails Integration Batch 1

**Files:** attachment_tools.py, calendar_tools.py, draft_tools.py,
           folder_tools.py, task_tools.py
**Changes:** audit logging, governance rules, output validation added
**Note:** All imports consolidated at top of each file

---

## 1. `tools/attachment_tools.py`

```python
"""
attachment_tools.py
====================
MCP tools for listing, reading, and summarising email attachments
(PDF, Word, PowerPoint, Excel, TXT, CSV, MD, images).

Governance layer:
    - Every tool call is audit-logged via utils/audit_logger.py
    - Summarisation instruction includes governance rules
    - Output validated before returning to LibreChat

Phase 1 tools:
    list_attachments, read_attachment, summarise_attachment
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from tools.mcp_instance import mcp

from graph.mail_client import fetch_message_by_id
from graph.attachment_client import (
    list_message_attachments,
    download_attachment,
)
from parsers.parser_router import parse_attachment

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.audit_logger import log_tool_call, get_user_email_from_headers
from utils.governance import get_email_rules

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# Tool: list_attachments
# ---------------------------------------------------------------------------
@mcp.tool
async def list_attachments(email_id: str) -> list[dict]:
    """
    List all attachments on a specific email without reading their content.

    Use this tool when the user asks things like:
    - "What's attached to this email?"
    - "Does this email have any files?"
    - "Show me the attachments on this message"

    IMPORTANT: Always call list_attachments first in the same conversation
    turn to get a fresh attachment_id before calling read_attachment or
    summarise_attachment. Do not reuse attachment IDs from previous turns
    as they expire.

    Args:
        email_id (str): The unique Graph API ID of the email to inspect.
                         Obtained from list_emails or search_emails.

    Returns:
        list[dict]: Each item contains attachment_id, file_name,
                     content_type, and size_kb.
    """
    logger.info(f"Tool called: list_attachments (email_id={email_id[:20]}...)")

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
    attachment (PDF, Word, PowerPoint, Excel, TXT, CSV, MD, or image).

    Use this tool when the user asks things like:
    - "Read this attached PDF"
    - "What does this Word document say?"
    - "Open the attached CV and tell me what's in it"
    - "Extract the data from this Excel file"

    IMPORTANT: Always call list_attachments first in the same turn to
    get a fresh attachment_id. Stale IDs from previous turns will fail.

    Args:
        email_id (str): The unique Graph API ID of the parent email.
        attachment_id (str): The unique ID of the attachment to read.
                              Obtained from list_attachments output.

    Returns:
        dict with file_name, file_type, extracted_text, and used_ocr flag.
    """
    logger.info(
        f"Tool called: read_attachment "
        f"(email_id={email_id[:20]}..., attachment_id={attachment_id[:20]}...)"
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

        # ── Download and parse ───────────────────────────────────────
        file_bytes, file_name, content_type = await download_attachment(
            email_id, attachment_id
        )
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
    Read an email attachment and return its content with a summarisation
    instruction for LibreChat's LLM. No internal LLM call is made —
    LibreChat's own model produces the summary.

    Use this tool when the user asks things like:
    - "Summarise this attached document"
    - "Give me the key points from this CV"
    - "What's the gist of this attached report?"

    IMPORTANT: Always call list_attachments first in the same turn to
    get a fresh attachment_id before calling this tool.

    Args:
        email_id (str): The unique Graph API ID of the parent email.
        attachment_id (str): The unique ID of the attachment to summarise.

    Returns:
        dict with file_name, file_type, extracted_text, and instruction
        for LibreChat's LLM to produce the summary.
    """
    logger.info(
        f"Tool called: summarise_attachment "
        f"(email_id={email_id[:20]}..., attachment_id={attachment_id[:20]}...)"
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

        # ── Download and parse ───────────────────────────────────────
        file_bytes, file_name, content_type = await download_attachment(
            email_id, attachment_id
        )
        parse_result = await parse_attachment(file_bytes, file_name, content_type)
        extracted_text = parse_result["text"]

        # ── Guard against empty extraction ───────────────────────────
        if not extracted_text or not extracted_text.strip():
            return {
                "file_name": file_name,
                "file_type": parse_result["file_type"],
                "extracted_text": "",
                "instruction": (
                    "No readable text could be extracted from this attachment. "
                    "It may be blank, corrupted, or an image-only file. "
                    "Inform the user accordingly."
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
```

---

## 2. `tools/calendar_tools.py`

```python
"""
calendar_tools.py
=================
MCP tools for viewing calendar events and checking availability.

Governance layer:
    - Every tool call is audit-logged via utils/audit_logger.py
    - Output validated via validate_calendar_events()

Phase 1 tools:
    list_calendar_events
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
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
    List the user's upcoming calendar events within a given time range.

    Use this tool when the user asks things like:
    - "What meetings do I have today?"
    - "Show me my calendar for this week"
    - "Do I have anything scheduled tomorrow?"

    Args:
        date_range (str): A natural-language time range such as:
                           "today", "tomorrow", "this week",
                           "next week", "next 7 days", "next 30 days".
                           Defaults to "this week".

    Returns:
        list[dict]: Each item contains subject, start_time, end_time,
                     organizer, location, and is_online_meeting flag.
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

        # ── Parse date range ─────────────────────────────────────────
        start_dt, end_dt = parse_relative_date_range(date_range)

        # ── Fetch events ─────────────────────────────────────────────
        raw_events = await fetch_calendar_events(start_dt, end_dt)

        # ── Build result ─────────────────────────────────────────────
        result = []
        for event in raw_events:
            result.append({
                "subject": event.get("subject"),
                "start_time": event.get("start", {}).get("dateTime"),
                "end_time": event.get("end", {}).get("dateTime"),
                "organizer": event.get("organizer", {}).get(
                    "emailAddress", {}
                ).get("name"),
                "location": event.get("location", {}).get("displayName", ""),
                "is_online_meeting": event.get("isOnlineMeeting", False),
            })

        # ── Validate output ───────────────────────────────────────────
        validation = validate_calendar_events(result)
        return append_validation_to_result({"events": result}, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="list_calendar_events", logger=logger)
```

---

## 3. `tools/draft_tools.py`

```python
"""
draft_tools.py
==============
MCP tools for composing email drafts, reply drafts, meeting invite
drafts, and finding attendee availability.

Governance layer:
    - Every tool call is audit-logged via utils/audit_logger.py
    - Compose tools have draft governance rules injected into instruction
    - save_draft_to_outlook validates content before saving

Drafting rules applied to all compose tools:
    - Professional tone, always polite
    - Short and crisp unless elaboration adds value
    - Meeting invites always include: agenda, description, clear header
    - Default meeting duration: 30 minutes
    - Never aggressive, never informal

Tools:
    list_draft_emails, compose_email, compose_reply,
    save_draft_to_outlook, find_availability, create_draft_invite
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from tools.mcp_instance import mcp

from graph.draft_client import (
    create_draft,
    get_draft_emails,
)
from graph.availability_client import (
    find_meeting_times,
    create_calendar_draft_invite,
)
from graph.mail_client import fetch_message_by_id

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.audit_logger import log_tool_call, get_user_email_from_headers
from utils.governance import get_draft_rules
from utils.validator import validate_draft_content, append_validation_to_result

logger = get_logger(__name__)


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
        count (int): Number of drafts to return. Default 10, max 50.

    Returns:
        dict with drafts list and a formatted display_table.
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
                f"| {i} | {date_display} | {to_display} "
                f"| {subject_col} | {preview_col} |"
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
# Tool: compose_email
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
    - "Write an email to the team about tomorrow's meeting"
    - "Compose a professional email to the client about the invoice"

    Args:
        to_emails (str): Comma-separated recipient email addresses.
        subject (str): The email subject line.
        context (str): What the email should say — LibreChat's LLM
                        composes the professional body from this.
        cc_emails (str): Optional comma-separated CC addresses.

    Returns:
        dict with header and composition instruction for LibreChat's LLM.
    """
    logger.info(f"Tool called: compose_email (to={to_emails}, subject={subject})")

    try:
        # ── Parse recipients ─────────────────────────────────────────
        to_list = [e.strip() for e in to_emails.split(",") if e.strip()]
        cc_list = [e.strip() for e in cc_emails.split(",") if e.strip()] if cc_emails else []

        if not to_list:
            return {
                "error": True,
                "message": "No valid recipient email addresses provided.",
            }

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="compose_email",
            user_email=user_email,
            inputs={"subject": subject},
        )

        # ── Derive greeting name from first recipient ─────────────────
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
    Read an email and compose a professional reply in chat for review.
    Does NOT save to Outlook yet — use save_draft_to_outlook after approval.

    Use this tool when the user asks things like:
    - "Draft a reply to John's email"
    - "Reply to this email professionally"
    - "Write a response to the client's email"

    Args:
        email_id (str): Graph API ID of the email to reply to.
                         Obtained from list_emails or search_emails.
        reply_context (str): What the reply should say.

    Returns:
        dict with original email context and composition instruction
        for LibreChat's LLM.
    """
    logger.info(f"Tool called: compose_reply (email_id={email_id[:20]}...)")

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="compose_reply",
            user_email=user_email,
            inputs={"email_id": email_id[:12]},
        )

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
                f"If yes, call save_draft_to_outlook with:\n"
                f"- to_emails: '{original_sender_email}'\n"
                f"- subject: 'Re: {original_subject}'\n"
                f"- body_text: the exact composed reply body\n\n"
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
    Call this AFTER compose_email or compose_reply and user approval.
    Pass the exact composed body text from the chat.

    Use this tool when the user says things like:
    - "Yes save it to drafts"
    - "Save that email"
    - "Push it to Outlook"

    Args:
        to_emails (str): Comma-separated recipient email addresses.
        subject (str): The email subject line.
        body_text (str): The exact composed email body text to save —
                          including greeting and sign-off.
        cc_emails (str): Optional comma-separated CC addresses.

    Returns:
        dict with draft ID and save confirmation.
    """
    logger.info(f"Tool called: save_draft_to_outlook (to={to_emails})")

    try:
        to_list = [e.strip() for e in to_emails.split(",") if e.strip()]
        cc_list = [e.strip() for e in cc_emails.split(",") if e.strip()] if cc_emails else []

        if not to_list:
            return {
                "error": True,
                "message": "No valid recipient addresses provided.",
            }

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="save_draft_to_outlook",
            user_email=user_email,
            inputs={"subject": subject},
        )

        # ── Validate draft content before saving ──────────────────────
        validation = validate_draft_content(
            body_text=body_text,
            to_emails=to_list,
            subject=subject,
        )
        if not validation.is_valid:
            return append_validation_to_result({}, validation)

        # ── Convert plain text to clean HTML ─────────────────────────
        paragraphs = [p.strip() for p in body_text.split("\n") if p.strip()]
        body_html = (
            '<html><body style="font-family: Calibri, Arial, sans-serif; '
            'font-size: 14px; color: #333;">'
            + "".join(f"<p>{p}</p>" for p in paragraphs)
            + "</body></html>"
        )

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
                f"**✅ Draft Saved to Outlook**\n\n"
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
                               "next Monday", "15 July".
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
                    "Try extending the date range or reducing attendees."
                ),
                "instruction": (
                    "Inform the user no slots were found and suggest alternatives."
                ),
            }

        table_lines = [
            f"**📅 Available Meeting Slots ({duration_minutes} min)**\n",
            f"*Checking: {', '.join(email_list)}*\n",
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
                "Then ask which slot the user would like and offer to "
                "draft a meeting invite for it."
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
    Draft a professional meeting invite with agenda and save it to
    Calendar. The invite is NOT sent until the user confirms in
    Outlook Calendar.

    Use this tool when the user asks things like:
    - "Draft a meeting invite for the project kickoff"
    - "Create a calendar invite for our review on Monday at 10am"
    - "Schedule a 30-minute call with the team"

    Args:
        subject (str): Meeting title.
        attendee_emails (str): Comma-separated attendee email addresses.
        start_datetime (str): Meeting start in ISO format or natural
                               language e.g. "Monday 10am".
        agenda (str): Meeting agenda points.
        duration_minutes (int): Duration in minutes. Default 30.
        location (str): Location string. Default "Microsoft Teams".
        is_online (bool): Create as online meeting. Default True.

    Returns:
        dict with event ID, invite preview, and confirmation.
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

        # ── Build professional HTML invite body ───────────────────────
        body_html = f"""
<html>
<body style="font-family: Calibri, Arial, sans-serif; font-size: 14px;">
<h2 style="color: #0078D4;">{subject}</h2>
<table style="border-collapse: collapse; margin-bottom: 16px;">
  <tr>
    <td style="padding: 4px 12px 4px 0; font-weight: bold;">Duration:</td>
    <td>{duration_minutes} minutes</td>
  </tr>
  <tr>
    <td style="padding: 4px 12px 4px 0; font-weight: bold;">Location:</td>
    <td>{location}</td>
  </tr>
</table>
<hr style="border: none; border-top: 1px solid #e0e0e0; margin: 16px 0;">
<h3 style="color: #0078D4;">📋 Agenda</h3>
<p style="margin-left: 16px;">{agenda.replace(chr(10), "<br>")}</p>
<hr style="border: none; border-top: 1px solid #e0e0e0; margin: 16px 0;">
<h3 style="color: #0078D4;">ℹ️ Description</h3>
<p>This meeting has been organised to discuss: <strong>{subject}</strong>.</p>
<p>Please review the agenda and come prepared.</p>
<hr style="border: none; border-top: 1px solid #e0e0e0; margin: 16px 0;">
<p style="color: #888; font-size: 12px;">
  Drafted via Outlook AI Agent. Please review before sending.
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

        invite_preview = (
            f"---\n"
            f"### 📅 Meeting Invite Draft\n\n"
            f"| Field | Details |\n"
            f"|---|---|\n"
            f"| **Subject** | {subject} |\n"
            f"| **Time** | {result['start']} – {result['end']} |\n"
            f"| **Duration** | {duration_minutes} minutes |\n"
            f"| **Attendees** | {', '.join(email_list)} |\n"
            f"| **Location** | {location} |\n\n"
            f"**Agenda:**\n{agenda}\n\n"
            f"---\n"
            f"✅ *Draft saved to Calendar. Review and send from Outlook.*"
        )

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
```

---

## 4. `tools/folder_tools.py`

```python
"""
folder_tools.py
================
MCP tools for Outlook mail folder management — listing folders,
creating new folders, and moving emails between folders.

Governance layer:
    - Every tool call is audit-logged via utils/audit_logger.py

Tools:
    list_folders, create_folder, move_email
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

# Well-known folder name mapping
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
        dict with folder list and a formatted display_table.
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
    - "Create a subfolder called Q3 inside Projects"

    Args:
        folder_name (str): Name for the new folder.
        parent_folder_name (str): Optional parent folder name.
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

        if parent_folder_name:
            parent_lower = parent_folder_name.strip().lower()
            if parent_lower in WELL_KNOWN_FOLDERS:
                parent_id = WELL_KNOWN_FOLDERS[parent_lower]
            else:
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
        destination_folder (str): Destination folder name.
                                    Accepts well-known names:
                                    inbox, drafts, sent, deleted,
                                    junk, archive — or any custom
                                    folder name from list_folders.

    Returns:
        dict with confirmation and new email location.
    """
    logger.info(
        f"Tool called: move_email "
        f"(email_id={email_id[:20]}..., destination='{destination_folder}')"
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

        if dest_lower in WELL_KNOWN_FOLDERS:
            folder_id = WELL_KNOWN_FOLDERS[dest_lower]
        else:
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

---

## 5. `tools/task_tools.py`

```python
"""
task_tools.py
=============
MCP tool for extracting actionable tasks and to-do items from emails.

Governance layer:
    - Every tool call is audit-logged via utils/audit_logger.py
    - Task extraction instruction includes governance rules
    - Output validated via validate_task_list()

Tools:
    extract_tasks
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from tools.mcp_instance import mcp

from graph.mail_client import (
    fetch_recent_messages,
    search_messages_by_keyword,
)

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.audit_logger import log_tool_call, get_user_email_from_headers
from utils.governance import get_task_rules

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
        count (int): Number of recent emails to analyse.
                      Default 10, max 25.
        keyword (str): Optional keyword to filter emails before
                        extracting tasks e.g. "project", "deadline".
        sender_email (str): Optional filter by sender email address.

    Returns:
        dict with email digest and task extraction instruction
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

        # ── Fetch emails ─────────────────────────────────────────────
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

        # ── Filter by sender if specified ────────────────────────────
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
                    f"Inform the user and suggest they check the address."
                ),
            }

        # ── Build structured email digest ────────────────────────────
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

        extraction_instruction = (
            f"Extract all actionable tasks, to-do items, deadlines, and "
            f"action owners from the email digest below.\n\n"
            f"Format the output as:\n\n"
            f"## 📋 Action Items\n\n"
            f"| # | Task | Owner | Deadline | Source Email |\n"
            f"|---|------|-------|----------|--------------|\n"
            f"| 1 | [action] | [person] | [date or 'Not specified'] "
            f"| [email subject] |\n\n"
            f"Then add:\n"
            f"## ⚠️ Deadlines to Watch\n"
            f"List only items with explicit deadlines.\n\n"
            f"## ℹ️ Notes\n"
            f"Any important context that doesn't fit as a task.\n\n"
            f"{governance}\n\n"
            f"Email digest to analyse:\n\n{full_digest}"
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
```

---

## Summary of changes per file

| File | Audit log | Governance | Validation |
|---|---|---|---|
| `attachment_tools.py` | ✅ all 3 tools | `summarise_attachment` only | ❌ |
| `calendar_tools.py` | ✅ | ❌ | `validate_calendar_events` |
| `draft_tools.py` | ✅ all 6 tools | `compose_email`, `compose_reply`, `create_draft_invite` | `save_draft_to_outlook` |
| `folder_tools.py` | ✅ all 3 tools | ❌ | ❌ |
| `task_tools.py` | ✅ | `get_task_rules()` in instruction | ❌ |

---

*Copy each code block to its respective file. Restart server after all 5 are updated.*
ENDOFFILE
echo "done"

~~~

---
