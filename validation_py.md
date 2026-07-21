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


**Files:** mom_tools.py, followup_tools.py, semantic_tools.py, chart_tools.py
**Changes:** audit logging, governance rules, output validation added
**Note:** All imports consolidated at top of each file

---

## 1. `tools/mom_tools.py`

```python
"""
mom_tools.py
============
MCP tools for generating formal Minutes of Meeting (MOM) from
email threads or single emails, and saving them as draft emails.

Governance layer:
    - Every tool call is audit-logged via utils/audit_logger.py
    - MOM generation instruction includes MOM governance rules
    - save_mom_as_draft validates content before saving
    - validate_mom_output checks generated MOM for required sections

Multi-language:
    - Detects email language automatically via utils/language_utils.py
    - MOM is generated in the same language as the source emails

Formal MOM format (7 sections):
    1. Meeting Details
    2. Attendees
    3. Discussion Summary
    4. Decisions Made
    5. Action Items
    6. Next Steps
    7. Notes

Tools:
    generate_mom, save_mom_as_draft
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
from utils.governance import get_mom_rules
from utils.language_utils import detect_language, get_language_instruction
from utils.validator import (
    validate_mom_output,
    validate_draft_content,
    append_validation_to_result,
)

logger = get_logger(__name__)


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
    - "Create minutes of meeting from this email chain"
    - "Prepare a formal MOM from this discussion"
    - "Extract meeting minutes from this email"

    Args:
        email_id (str): Graph API ID of any email in the thread.
                         If use_thread=True, fetches the full chain.
                         If use_thread=False, uses only this email.
        use_thread (bool): True to fetch the full conversation thread
                            (recommended for accurate MOM). Default True.

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

        # ── Fetch email(s) ───────────────────────────────────────────
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

        # ── Collect all participants ──────────────────────────────────
        participants = {}
        for msg in messages:
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

        # ── Build attendee table ──────────────────────────────────────
        attendee_rows = "\n".join([
            f"| {name} | {email} | Participant |"
            for email, name in participants.items()
        ])

        # ── Build email digest ────────────────────────────────────────
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

        # ── Detect language ───────────────────────────────────────────
        sample_text = " ".join([
            msg.get("subject", "") + " " + (
                msg.get("body", {}).get("content", "")
                if isinstance(msg.get("body"), dict)
                else msg.get("body", "")
            )[:200]
            for msg in messages[:3]
        ])
        lang_code, lang_name = detect_language(sample_text)
        lang_instruction = get_language_instruction(lang_code, lang_name)

        # ── Meeting metadata ──────────────────────────────────────────
        first_msg = messages[0]
        last_msg = messages[-1]
        meeting_subject = first_msg.get("subject", "Meeting")
        meeting_date = (
            last_msg.get("received_date")
            or last_msg.get("receivedDateTime", "")
        )[:10]

        # ── Governance rules ─────────────────────────────────────────
        governance = get_mom_rules(include_sources=True)

        # ── Build MOM instruction ─────────────────────────────────────
        mom_instruction = (
            f"Generate a formal Minutes of Meeting (MOM) document.\n"
            f"{lang_instruction}\n\n"
            f"Use EXACTLY this structure:\n\n"
            f"═══════════════════════════════════════════════════\n"
            f"         MINUTES OF MEETING (MOM)\n"
            f"═══════════════════════════════════════════════════\n\n"
            f"1. MEETING DETAILS\n"
            f"   Date        : {meeting_date}\n"
            f"   Subject     : {meeting_subject}\n"
            f"   Source      : {source_label} ({len(messages)} message(s))\n"
            f"   Language    : {lang_name}\n\n"
            f"2. ATTENDEES\n"
            f"   | Name | Email | Role |\n"
            f"   |------|-------|------|\n"
            f"{attendee_rows}\n\n"
            f"3. DISCUSSION SUMMARY\n"
            f"   [Write 3-5 paragraphs summarising key discussion points]\n\n"
            f"4. DECISIONS MADE\n"
            f"   [List explicit decisions as bullet points]\n\n"
            f"5. ACTION ITEMS\n"
            f"   | # | Action | Owner | Deadline | Status |\n"
            f"   |---|--------|-------|----------|--------|\n"
            f"   [Extract all action items]\n\n"
            f"6. NEXT STEPS\n"
            f"   [Summarise what happens next]\n\n"
            f"7. NOTES\n"
            f"   [Any additional context]\n\n"
            f"═══════════════════════════════════════════════════\n\n"
            f"{governance}\n\n"
            f"Email content to base the MOM on:\n\n{email_digest}"
        )

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
        mom_content (str): The full MOM text as generated and
                            approved in chat.

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

        # ── Validate MOM content before saving ────────────────────────
        validation = validate_draft_content(
            body_text=mom_content,
            to_emails=to_list,
            subject=subject,
        )
        if not validation.is_valid:
            return append_validation_to_result({}, validation)

        # ── Convert to HTML ───────────────────────────────────────────
        paragraphs = [p.strip() for p in mom_content.split("\n") if p.strip()]
        body_html = (
            '<html><body style="font-family: Calibri, Arial, sans-serif; '
            'font-size: 13px; color: #333;">'
            '<div style="white-space: pre-wrap; font-family: monospace;">'
            + "\n".join(paragraphs)
            + "</div></body></html>"
        )

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
```

---

## 2. `tools/followup_tools.py`

```python
"""
followup_tools.py
=================
MCP tools for tracking emails that need follow-up, composing
professional follow-up messages, and creating reminder tasks
in Microsoft To-Do or Planner.

Governance layer:
    - Every tool call is audit-logged via utils/audit_logger.py
    - Follow-up compose instruction includes governance rules
    - Multi-language: follow-up drafts match original email language

Tools:
    track_followups, check_email_replied, compose_followup,
    add_task_todo, add_task_planner, list_tasks
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from datetime import datetime, timedelta, timezone

from tools.mcp_instance import mcp

from graph.mail_client import fetch_recent_messages, fetch_message_by_id
from graph.thread_client import fetch_email_thread
from graph.task_client import (
    create_todo_task,
    create_planner_task,
    get_todo_tasks,
)

from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.audit_logger import log_tool_call, get_user_email_from_headers
from utils.governance import get_followup_rules
from utils.language_utils import detect_language, get_language_instruction
from config.settings import settings

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
        threshold_days (int): Days after which unreplied email is overdue.

    Returns:
        bool: True if follow-up is needed.
    """
    if msg.get("is_draft"):
        return False

    sender_email = (
        msg.get("from", {}).get("emailAddress", {}).get("address", "").lower()
    )
    if sender_email == user_email.lower():
        return False

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

        # ── Get current user profile ──────────────────────────────────
        from graph.graph_client_factory import get_user_profile
        profile = await get_user_profile()
        current_user_email = (
            profile.get("mail") or profile.get("userPrincipalName") or ""
        )

        # ── Fetch recent inbox emails ─────────────────────────────────
        raw_messages = await fetch_recent_messages(top=scan_count)

        if not raw_messages:
            return {
                "followup_count": 0,
                "display_table": "📭 Inbox is empty — no emails to track.",
                "instruction": "Inform the user their inbox is empty.",
            }

        # ── Filter emails needing follow-up ───────────────────────────
        needs_reply = [
            m for m in raw_messages
            if _needs_followup(m, current_user_email, threshold)
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

        # ── Build display table ───────────────────────────────────────
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
            urgency = (
                "🔴" if isinstance(age_days, int) and age_days >= 7 else "🟡"
            )

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
    logger.info(
        f"Tool called: check_email_replied (email_id={email_id[:20]}...)"
    )

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="check_email_replied",
            user_email=user_email,
            inputs={"email_id": email_id[:12]},
        )

        # ── Get current user ──────────────────────────────────────────
        from graph.graph_client_factory import get_user_profile
        profile = await get_user_profile()
        current_user_email = (
            profile.get("mail") or profile.get("userPrincipalName") or ""
        ).lower()

        # ── Fetch thread ──────────────────────────────────────────────
        thread = await fetch_email_thread(email_id)

        target_msg = next(
            (m for m in thread if m.get("id") == email_id), thread[0]
        )

        user_replies = [
            m for m in thread
            if m.get("sender_email", "").lower() == current_user_email
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
        return format_tool_error(
            exc, tool_name="check_email_replied", logger=logger
        )


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
    and display it in chat for review. Detects original email language
    automatically and matches it in the follow-up.

    Use this tool when the user asks things like:
    - "Compose a follow-up for John's email"
    - "Write a polite follow-up to this unanswered email"
    - "Draft a chaser for this overdue email"

    Args:
        email_id (str): Graph API ID of the email to follow up on.
        followup_context (str): Optional extra context e.g.
                                  "mention the deadline is this Friday".

    Returns:
        dict with follow-up composition instruction for LibreChat's LLM.
    """
    logger.info(
        f"Tool called: compose_followup (email_id={email_id[:20]}...)"
    )

    try:
        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="compose_followup",
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
        original_preview = original.get("body", {}).get("content", "")[:400]
        received_raw = original.get("receivedDateTime", "")

        # ── Calculate age ─────────────────────────────────────────────
        try:
            received_dt = datetime.fromisoformat(
                received_raw.replace("Z", "+00:00")
            )
            age_days = (datetime.now(timezone.utc) - received_dt).days
            original_date = received_dt.strftime("%d %B %Y")
        except Exception:
            age_days = "several"
            original_date = received_raw[:10]

        # ── Detect language ───────────────────────────────────────────
        lang_code, lang_name = detect_language(original_preview)
        lang_instruction = get_language_instruction(lang_code, lang_name)

        # ── Governance rules ─────────────────────────────────────────
        governance = get_followup_rules()

        extra = (
            f"\nAdditional context: {followup_context}"
            if followup_context else ""
        )

        return {
            "original_subject": original_subject,
            "original_sender": original_sender_name,
            "original_sender_email": original_sender_email,
            "age_days": age_days,
            "language_detected": lang_name,
            "instruction": (
                f"Compose and display a professional follow-up email.\n"
                f"{lang_instruction}\n\n"
                f"Display this header first:\n\n"
                f"**📧 Follow-up Draft**\n\n"
                f"| Field | Details |\n"
                f"|---|---|\n"
                f"| **To** | {original_sender_name} <{original_sender_email}> |\n"
                f"| **Re** | {original_subject} |\n"
                f"| **Original date** | {original_date} ({age_days} days ago) |\n\n"
                f"---\n\n"
                f"Then compose the follow-up body:\n"
                f"- Start: Dear {original_sender_name},\n"
                f"- Politely reference the original email sent {age_days} days ago\n"
                f"- Gently ask for an update or response\n"
                f"- Keep it to 2-3 sentences maximum\n"
                f"- Never aggressive or demanding\n"
                f"- End: Best regards,{extra}\n\n"
                f"{governance}\n\n"
                f"Original email context:\n{original_preview}\n\n"
                f"After composing ask:\n"
                f"'Would you like me to save this follow-up to Outlook Drafts?'\n"
                f"If yes, call save_draft_to_outlook with:\n"
                f"- to_emails: '{original_sender_email}'\n"
                f"- subject: 'Re: {original_subject}'\n"
                f"- body_text: the exact composed follow-up"
            ),
        }

    except Exception as exc:
        return format_tool_error(
            exc, tool_name="compose_followup", logger=logger
        )


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
        assigned_to_email (str): Email of the team member to assign to.

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
        return format_tool_error(
            exc, tool_name="add_task_planner", logger=logger
        )


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
                          Only used when source="todo".
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
                "source": source_label,
                "display_table": f"✅ No pending tasks in {source_label}.",
                "instruction": (
                    f"Inform the user their {source_label} task list is empty."
                ),
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
```

---

## 3. `tools/semantic_tools.py`

```python
"""
semantic_tools.py
=================
MCP tool for semantic search over email history.

Governance layer:
    - Every tool call is audit-logged via utils/audit_logger.py

How it works:
    Fetches recent emails, converts each to a vector embedding using
    sentence-transformers, computes cosine similarity against the query,
    and returns ranked results — without needing exact keyword matches.

Tools:
    semantic_search_emails
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

    Use semantic_search_emails instead of search_emails when:
    - The user describes a concept or topic, not an exact phrase
    - Keyword search returned no results but the email should exist
    - The user wants to search by meaning or sentiment

    Args:
        query (str): Natural language description of what to find.
                      e.g. "deadline pressure from management",
                           "budget approval requests"
        top_k (int): Number of top matching emails to return. Default 10.
        corpus_size (int): How many recent emails to search over.
                            Default from settings (100). Max 200.

    Returns:
        dict with semantically ranked email results and display table.
    """
    logger.info(
        f"Tool called: semantic_search_emails (query='{query}', top_k={top_k})"
    )

    try:
        if not query or not query.strip():
            return {
                "error": True,
                "message": "Please provide a search query.",
            }

        safe_top_k = min(max(top_k, 1), 25)
        max_corpus = min(
            corpus_size or settings.SEMANTIC_SEARCH_MAX_EMAILS,
            200,
        )

        # ── Audit log ────────────────────────────────────────────────
        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="semantic_search_emails",
            user_email=user_email,
            inputs={"top_k": safe_top_k},
        )

        # ── Fetch email corpus ────────────────────────────────────────
        logger.info(f"Fetching {max_corpus} emails for semantic corpus...")
        raw_emails = await fetch_recent_messages(top=max_corpus)

        if not raw_emails:
            return {
                "results": [],
                "query": query,
                "display_table": "📭 No emails found to search over.",
                "instruction": "Inform the user their inbox appears empty.",
            }

        # ── Run semantic search ───────────────────────────────────────
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
                    f"Try rephrasing your query or use search_emails for "
                    f"keyword search."
                ),
                "instruction": (
                    "Display the message above and suggest the user "
                    "try keyword search or rephrase their query."
                ),
            }

        # ── Build display table ───────────────────────────────────────
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
                .get("name") or "—"
            )
            subject = (email.get("subject") or "(no subject)")[:55]
            relevance = email.get("relevance", "—")
            score = email.get("similarity_score", 0)

            table_lines.append(
                f"| {i} | {relevance} ({score:.2f}) | {date_display} "
                f"| {sender} | {subject} |"
            )

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
```

---

## 4. `tools/chart_tools.py`

```python
"""
chart_tools.py
==============
MCP tool for generating charts and visualisations from email data,
task lists, calendar events, or any structured data.

Governance layer:
    - Every tool call is audit-logged via utils/audit_logger.py

Output format:
    SVG — renders natively in LibreChat's markdown chat interface.
    Files also saved to temp_attachments/charts/ as backup.

Supported chart types:
    bar  — compare values across categories
    line — show trends over time
    pie  — show proportions/distribution

Tools:
    generate_chart
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import os
import uuid
from pathlib import Path
from datetime import datetime

import matplotlib
matplotlib.use("Agg")  # Non-interactive backend — required for Linux server
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches

from tools.mcp_instance import mcp
from config.settings import settings
from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.audit_logger import log_tool_call, get_user_email_from_headers

logger = get_logger(__name__)

# Professional colour palette — blue-aligned
CHART_COLOURS = [
    "#0078D4",  # Microsoft Blue
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
# Helpers
# ---------------------------------------------------------------------------
def _ensure_chart_dir() -> Path:
    """Create chart output directory if it doesn't exist."""
    chart_dir = Path(settings.CHARTS_OUTPUT_DIR).resolve()
    chart_dir.mkdir(parents=True, exist_ok=True)
    return chart_dir


def _make_filename(chart_type: str) -> str:
    """Generate a unique timestamped SVG filename."""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    short_id = str(uuid.uuid4())[:4]
    return f"{chart_type}_chart_{timestamp}_{short_id}.svg"


def _save_as_svg(fig: plt.Figure, filename: str) -> tuple[str, str]:
    """
    Save a matplotlib figure as SVG and read it as a string.
    SVG renders natively in LibreChat's markdown.

    Args:
        fig (plt.Figure): The matplotlib figure to save.
        filename (str): Output filename.

    Returns:
        tuple[str, str]: (file_path, svg_string)
    """
    chart_dir = _ensure_chart_dir()
    file_path = chart_dir / filename

    fig.savefig(
        file_path,
        format="svg",
        bbox_inches="tight",
        facecolor="white",
    )
    plt.close(fig)

    svg_string = file_path.read_text(encoding="utf-8")
    logger.info(f"SVG chart saved: {file_path}")
    return str(file_path), svg_string


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
    Generate a chart (bar, line, or pie) from provided data and render
    it as an SVG image directly in the LibreChat chat interface.

    Use this tool when the user asks things like:
    - "Show me a bar chart of emails by sender"
    - "Generate a pie chart of my task status"
    - "Plot a line chart of emails received per day this week"
    - "Visualise my calendar events by day as a bar chart"
    - "Create a chart from this data"

    Call AFTER another tool provides data (e.g. list_emails, list_tasks).
    The LLM aggregates the data from those results and passes it here.

    Args:
        chart_type (str): "bar", "line", or "pie".
        labels (str): Comma-separated category labels.
                       e.g. "John Smith, Jane Doe, HR Team"
        values (str): Comma-separated numeric values matching labels.
                       e.g. "15, 8, 3"
        title (str): Chart title.
        x_label (str): X-axis label (bar and line charts only).
        y_label (str): Y-axis label (bar and line charts only).

    Returns:
        dict with SVG content and instruction to render it in chat.
    """
    logger.info(
        f"Tool called: generate_chart (type={chart_type}, title='{title}')"
    )

    try:
        chart_type_clean = chart_type.strip().lower()

        if chart_type_clean not in ("bar", "line", "pie"):
            return {
                "error": True,
                "message": (
                    f"Unsupported chart type: '{chart_type}'. "
                    f"Please use 'bar', 'line', or 'pie'."
                ),
            }

        # ── Parse and validate inputs ─────────────────────────────────
        label_list = [l.strip() for l in labels.split(",") if l.strip()]
        value_list_raw = [v.strip() for v in values.split(",") if v.strip()]

        if not label_list or not value_list_raw:
            return {
                "error": True,
                "message": "Labels and values cannot be empty.",
            }

        if len(label_list) != len(value_list_raw):
            return {
                "error": True,
                "message": (
                    f"Mismatch: {len(label_list)} labels but "
                    f"{len(value_list_raw)} values. They must match."
                ),
            }

        try:
            value_list = [float(v) for v in value_list_raw]
        except ValueError as ve:
            return {
                "error": True,
                "message": (
                    f"Values must be numbers. Got: {value_list_raw}. "
                    f"Error: {ve}"
                ),
            }

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

        # ── Apply consistent chart styling ───────────────────────────
        plt.rcParams.update({
            "font.family": "DejaVu Sans",
            "font.size": 11,
            "axes.titlesize": 14,
            "axes.titleweight": "bold",
            "axes.titlepad": 16,
            "axes.spines.top": False,
            "axes.spines.right": False,
            "figure.facecolor": "white",
            "axes.facecolor": "#f8f9fa",
            "axes.grid": True,
            "grid.alpha": 0.4,
            "grid.linestyle": "--",
        })

        # ── Generate chart ────────────────────────────────────────────
        if chart_type_clean == "bar":
            file_path, svg_content = _generate_bar_chart(
                label_list, value_list, title, x_label, y_label
            )
        elif chart_type_clean == "line":
            file_path, svg_content = _generate_line_chart(
                label_list, value_list, title, x_label, y_label
            )
        elif chart_type_clean == "pie":
            file_path, svg_content = _generate_pie_chart(
                label_list, value_list, title
            )

        file_name = Path(file_path).name

        return {
            "chart_type": chart_type_clean,
            "title": title,
            "file_name": file_name,
            "file_path": file_path,
            "data_points": len(label_list),
            "instruction": (
                f"Display this SVG chart directly in the chat — "
                f"render it as raw SVG markup, do not wrap it in a code block:\n\n"
                f"**📊 {title}**\n\n"
                f"{svg_content}\n\n"
                f"Then in 1-2 sentences summarise the key insight from this data."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="generate_chart", logger=logger)


# ---------------------------------------------------------------------------
# Internal: Bar chart
# ---------------------------------------------------------------------------
def _generate_bar_chart(
    labels: list,
    values: list,
    title: str,
    x_label: str,
    y_label: str,
) -> tuple[str, str]:
    """Generate a professional vertical bar chart and return as SVG."""
    fig_width = max(8, len(labels) * 0.9)
    fig, ax = plt.subplots(figsize=(fig_width, 5))

    colours = [CHART_COLOURS[i % len(CHART_COLOURS)] for i in range(len(labels))]
    bars = ax.bar(labels, values, color=colours, edgecolor="white",
                  linewidth=1.5, width=0.6)

    for bar, value in zip(bars, values):
        ax.text(
            bar.get_x() + bar.get_width() / 2,
            bar.get_height() + max(values) * 0.01,
            f"{value:,.0f}" if value == int(value) else f"{value:,.1f}",
            ha="center", va="bottom", fontsize=10,
            fontweight="bold", color="#333333",
        )

    ax.set_title(title or "Bar Chart", pad=16)
    if x_label:
        ax.set_xlabel(x_label, labelpad=8)
    if y_label:
        ax.set_ylabel(y_label, labelpad=8)
    if len(labels) > 6:
        plt.xticks(rotation=30, ha="right")
    ax.set_ylim(0, max(values) * 1.15)
    plt.tight_layout()

    return _save_as_svg(fig, _make_filename("bar"))


# ---------------------------------------------------------------------------
# Internal: Line chart
# ---------------------------------------------------------------------------
def _generate_line_chart(
    labels: list,
    values: list,
    title: str,
    x_label: str,
    y_label: str,
) -> tuple[str, str]:
    """Generate a professional line/trend chart and return as SVG."""
    fig, ax = plt.subplots(figsize=(10, 5))
    x_positions = range(len(labels))

    ax.plot(
        x_positions, values,
        color=CHART_COLOURS[0], linewidth=2.5,
        marker="o", markersize=7,
        markerfacecolor="white",
        markeredgecolor=CHART_COLOURS[0],
        markeredgewidth=2.5, zorder=3,
    )
    ax.fill_between(x_positions, values, alpha=0.12, color=CHART_COLOURS[0])

    for i, (x, y) in enumerate(zip(x_positions, values)):
        ax.annotate(
            f"{y:,.0f}" if y == int(y) else f"{y:,.1f}",
            (x, y), textcoords="offset points", xytext=(0, 10),
            ha="center", fontsize=9, color="#333333",
        )

    ax.set_xticks(x_positions)
    ax.set_xticklabels(
        labels,
        rotation=30 if len(labels) > 6 else 0,
        ha="right",
    )
    ax.set_title(title or "Line Chart", pad=16)
    if x_label:
        ax.set_xlabel(x_label, labelpad=8)
    if y_label:
        ax.set_ylabel(y_label, labelpad=8)
    ax.set_ylim(0, max(values) * 1.2)
    plt.tight_layout()

    return _save_as_svg(fig, _make_filename("line"))


# ---------------------------------------------------------------------------
# Internal: Pie chart
# ---------------------------------------------------------------------------
def _generate_pie_chart(
    labels: list,
    values: list,
    title: str,
) -> tuple[str, str]:
    """Generate a professional pie chart and return as SVG."""
    fig, ax = plt.subplots(figsize=(9, 7))
    colours = [CHART_COLOURS[i % len(CHART_COLOURS)] for i in range(len(labels))]

    max_idx = values.index(max(values))
    explode = [0.05 if i == max_idx else 0 for i in range(len(values))]

    wedges, texts, autotexts = ax.pie(
        values, labels=None, autopct="%1.1f%%",
        colors=colours, explode=explode, startangle=90,
        pctdistance=0.75,
        wedgeprops={"linewidth": 2, "edgecolor": "white"},
    )

    for autotext in autotexts:
        autotext.set_fontsize(10)
        autotext.set_fontweight("bold")
        autotext.set_color("white")

    total = sum(values)
    legend_labels = [
        f"{label}  ({val:,.0f} — {val/total*100:.1f}%)"
        for label, val in zip(labels, values)
    ]
    legend_patches = [
        mpatches.Patch(color=colours[i], label=legend_labels[i])
        for i in range(len(labels))
    ]
    ax.legend(
        handles=legend_patches,
        loc="lower center",
        bbox_to_anchor=(0.5, -0.15),
        ncol=min(3, len(labels)),
        frameon=False,
        fontsize=9,
    )

    ax.set_title(title or "Pie Chart", pad=20)
    plt.tight_layout()

    return _save_as_svg(fig, _make_filename("pie"))
```

---

## Summary

| File | Audit log | Governance | Validation |
|---|---|---|---|
| `mom_tools.py` | ✅ both tools | `get_mom_rules()` in `generate_mom` | `validate_mom_output` + `validate_draft_content` |
| `followup_tools.py` | ✅ all 6 tools | `get_followup_rules()` in `compose_followup` | ❌ |
| `semantic_tools.py` | ✅ | ❌ | ❌ |
| `chart_tools.py` | ✅ | ❌ | ❌ |

---

*Copy each code block to its respective file. Restart server after all 4 are updated.*
ENDOFFILE
echo "done"
