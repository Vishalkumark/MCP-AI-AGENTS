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