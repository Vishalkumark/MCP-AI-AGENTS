# Phase 1 — Output & Feature Enhancements

**Project:** outlook-ai-agent-mcp
**Covers:** Points 1–6 from review
**Status:** Apply in order, restart server after all changes

-----

## Point 1 & 2 — Tabular default output + neat templates

### Change: `tools/email_tools.py`

The `list_emails` tool currently returns a plain list of dicts. We add
a formatted markdown table as an additional field so LibreChat’s LLM
displays it neatly without any extra prompting.

**Find the result building block in `list_emails`:**

```python
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
```

**Replace with:**

```python
        result = []
        for msg in raw_messages:
            received_raw = msg.get("receivedDateTime", "")
            # Format date: 2026-06-15T09:30:00Z → 15 Jun 2026 09:30
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

        # Build markdown table for clean display in LibreChat
        table_lines = [
            "| # | Date | From | Subject | Preview | 📎 |",
            "|---|------|------|---------|---------|-----|",
        ]
        for i, m in enumerate(result, 1):
            att = "✅" if m["has_attachments"] else "—"
            sender_col = f"{m['sender']}"
            subject_col = m["subject"][:50] + ("…" if len(m["subject"]) > 50 else "")
            preview_col = m["preview"][:80] + ("…" if len(m["preview"]) > 80 else "")
            table_lines.append(
                f"| {i} | {m['date_display']} | {sender_col} "
                f"| {subject_col} | {preview_col} | {att} |"
            )

        return {
            "emails": result,
            "count": len(result),
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display the 'display_table' field as a markdown table. "
                "Do not reformat or summarise it — show it exactly as given. "
                "Then offer to read, summarise, or search any specific email."
            ),
        }
```

-----

## Point 3 — Inbox view for subject-based search

When the user asks for a specific email by subject, return a rich
inbox-style view instead of a plain dict.

**In `tools/email_tools.py`, find the result block in `read_email`:**

```python
        return {
            "subject": msg.get("subject"),
            "sender": msg.get("from", {}).get("emailAddress", {}).get("name"),
            "sender_email": msg.get("from", {}).get("emailAddress", {}).get("address"),
            "received_date": msg.get("receivedDateTime"),
            "body": msg.get("body", {}).get("content", ""),
            "has_attachments": msg.get("hasAttachments", False),
        }
```

**Replace with:**

```python
        subject = msg.get("subject") or "(no subject)"
        sender_name = msg.get("from", {}).get("emailAddress", {}).get("name") or "—"
        sender_addr = msg.get("from", {}).get("emailAddress", {}).get("address") or "—"
        received_raw = msg.get("receivedDateTime", "")
        body = msg.get("body", {}).get("content", "")
        has_att = msg.get("hasAttachments", False)

        # Format date
        try:
            from dateutil import parser as dp
            dt = dp.parse(received_raw)
            date_display = dt.strftime("%A, %d %B %Y at %H:%M")
        except Exception:
            date_display = received_raw

        # Build Outlook-style inbox view as markdown
        inbox_view = f"""
---
### 📧 {subject}

| Field       | Details                          |
|-------------|----------------------------------|
| **From**    | {sender_name} <{sender_addr}>   |
| **Date**    | {date_display}                   |
| **Subject** | {subject}                        |
| **Attach.** | {"📎 Yes" if has_att else "None"} |

---

{body}

---
"""

        return {
            "subject": subject,
            "sender": sender_name,
            "sender_email": sender_addr,
            "received_date": received_raw,
            "body": body,
            "has_attachments": has_att,
            "inbox_view": inbox_view,
            "instruction": (
                "Display the 'inbox_view' field exactly as markdown — "
                "this is the formatted email view. Do not reformat the body."
            ),
        }
```

-----

## Point 4 — Date range + sender filtering

Add a new MCP tool `search_emails_advanced` that supports sender,
date range, and keyword filtering together.

**In `tools/email_tools.py`, add this new tool after `search_emails`:**

```python
@mcp.tool
async def search_emails_advanced(
    sender_email: str = "",
    keyword: str = "",
    date_from: str = "",
    date_to: str = "",
    count: int = 20,
) -> dict:
    """
    Advanced email search with sender, keyword, and date range filters.

    Use this tool when the user asks things like:
    - "Get emails from jane@company.com between June 10 and June 18"
    - "Show emails from John about budget last week"
    - "Find all emails received between 1 July and 7 July"

    Args:
        sender_email (str): Filter by sender email address.
        keyword (str): Filter by keyword in subject or body.
        date_from (str): Start date in any format e.g. "June 10",
                          "2026-06-10", "10 Jun 2026".
        date_to (str): End date in any format.
        count (int): Maximum number of results. Default 20, max 50.

    Returns:
        dict with emails list and a display_table for clean rendering.
    """
    logger.info(
        f"Tool called: search_emails_advanced "
        f"(sender={sender_email}, keyword={keyword}, "
        f"date_from={date_from}, date_to={date_to})"
    )

    try:
        from graph.mail_client import search_messages_advanced
        safe_count = min(max(count, 1), 50)

        raw_messages = await search_messages_advanced(
            sender_email=sender_email,
            keyword=keyword,
            date_from=date_from,
            date_to=date_to,
            top=safe_count,
        )

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

        # Build table
        table_lines = [
            f"**Search Results** — {len(result)} email(s) found\n",
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

        return {
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

    except Exception as exc:
        return format_tool_error(exc, tool_name="search_emails_advanced", logger=logger)
```

### Add `search_messages_advanced` to `graph/mail_client.py`

Add this function at the bottom of `graph/mail_client.py`:

```python
async def search_messages_advanced(
    sender_email: str = "",
    keyword: str = "",
    date_from: str = "",
    date_to: str = "",
    top: int = 20,
) -> list[dict]:
    """
    Search messages with combined sender, keyword, and date range filters
    using Graph API's $filter and $search query parameters.
    """
    logger.info(
        f"Advanced search: sender={sender_email}, keyword={keyword}, "
        f"date_from={date_from}, date_to={date_to}"
    )

    client = get_graph_client()

    from msgraph.generated.users.item.messages.messages_request_builder import (
        MessagesRequestBuilder,
    )
    from utils.date_utils import parse_date_string

    # Build $filter clauses
    filter_parts = []

    if sender_email:
        filter_parts.append(
            f"from/emailAddress/address eq '{sender_email}'"
        )

    if date_from:
        dt_from = parse_date_string(date_from)
        if dt_from:
            filter_parts.append(
                f"receivedDateTime ge {dt_from.strftime('%Y-%m-%dT00:00:00Z')}"
            )

    if date_to:
        dt_to = parse_date_string(date_to)
        if dt_to:
            filter_parts.append(
                f"receivedDateTime le {dt_to.strftime('%Y-%m-%dT23:59:59Z')}"
            )

    filter_str = " and ".join(filter_parts) if filter_parts else None

    # $search and $filter cannot be combined in Graph API —
    # if both are needed, apply $search and filter dates/sender client-side
    if keyword and filter_str:
        # Fetch via search, then filter client-side by date/sender
        query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
            search=f'"{keyword}"',
            top=min(top * 3, 100),  # fetch more to allow for client-side filtering
        )
        request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
            query_parameters=query_params,
        )
        request_config.headers.add("ConsistencyLevel", "eventual")
        response = await client.me.messages.get(request_configuration=request_config)
        messages = response.value if response and response.value else []

        # Apply sender/date filters client-side
        from dateutil import parser as dp
        filtered = []
        for msg in messages:
            if sender_email:
                msg_sender = ""
                if msg.from_ and msg.from_.email_address:
                    msg_sender = msg.from_.email_address.address or ""
                if sender_email.lower() not in msg_sender.lower():
                    continue
            if date_from:
                dt_from = parse_date_string(date_from)
                if dt_from and msg.received_date_time:
                    msg_dt = msg.received_date_time.replace(tzinfo=None) if msg.received_date_time.tzinfo else msg.received_date_time
                    if msg.received_date_time < dt_from:
                        continue
            if date_to:
                dt_to = parse_date_string(date_to)
                if dt_to and msg.received_date_time:
                    from datetime import timedelta
                    if msg.received_date_time > dt_to + timedelta(days=1):
                        continue
            filtered.append(msg)
            if len(filtered) >= top:
                break
        return [_message_to_dict(m) for m in filtered]

    elif keyword:
        # Keyword only — use $search
        query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
            search=f'"{keyword}"',
            top=top,
        )
        request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
            query_parameters=query_params,
        )
        request_config.headers.add("ConsistencyLevel", "eventual")

    elif filter_str:
        # Sender/date only — use $filter
        query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
            filter=filter_str,
            top=top,
            orderby=["receivedDateTime DESC"],
        )
        request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
            query_parameters=query_params,
        )

    else:
        # No filters — fall back to recent messages
        query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
            top=top,
            orderby=["receivedDateTime DESC"],
        )
        request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
            query_parameters=query_params,
        )

    response = await client.me.messages.get(request_configuration=request_config)
    messages = response.value if response and response.value else []
    return [_message_to_dict(m) for m in messages]
```

-----

## Point 5 — Pagination for 200+ emails

Add a `list_emails_paged` tool with `skip` offset support, plus a
`export_emails_markdown` tool that dumps all results to a markdown file.

**Add both tools to `tools/email_tools.py`:**

```python
@mcp.tool
async def list_emails_paged(
    count: int = 50,
    page: int = 1,
) -> dict:
    """
    List emails in pages of up to 50. Use this when the user wants
    more than 50 emails or asks for the "next page" of results.

    Use this tool when the user asks things like:
    - "Show me the next 50 emails"
    - "Get page 2 of my inbox"
    - "List emails 51 to 100"

    Args:
        count (int): Emails per page, max 50. Default 50.
        page (int): Page number, starting at 1. Default 1.

    Returns:
        dict with emails, table, page info, and next page prompt.
    """
    logger.info(f"Tool called: list_emails_paged (count={count}, page={page})")

    try:
        from graph.mail_client import fetch_messages_paged
        safe_count = min(max(count, 1), 50)
        skip = (page - 1) * safe_count

        raw_messages = await fetch_messages_paged(top=safe_count, skip=skip)

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

        has_more = len(result) == safe_count

        return {
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

    except Exception as exc:
        return format_tool_error(exc, tool_name="list_emails_paged", logger=logger)


@mcp.tool
async def export_emails_markdown(
    count: int = 200,
    keyword: str = "",
) -> dict:
    """
    Export a large list of emails to a downloadable markdown table.
    Use this when the user wants more than 50 emails exported or
    asks to save email results to a file.

    Use this tool when the user asks things like:
    - "Export all my emails to a file"
    - "Give me all 200 emails in a markdown file"
    - "Download my inbox as a table"

    Args:
        count (int): Total emails to export, max 200.
        keyword (str): Optional keyword filter.

    Returns:
        dict with the full markdown content and instruction to display it.
    """
    logger.info(f"Tool called: export_emails_markdown (count={count})")

    try:
        from graph.mail_client import fetch_recent_messages, search_messages_by_keyword
        safe_count = min(max(count, 1), 200)

        # Fetch in batches of 50 (Graph API limit per request)
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
                break  # No more messages available

        # Build full markdown table
        lines = [
            f"# Outlook Inbox Export",
            f"**Total emails:** {len(all_messages)}",
            f"**Filter:** {keyword if keyword else 'None'}",
            f"",
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
            sender_email = msg.get("from", {}).get("emailAddress", {}).get("address") or "—"
            subject = (msg.get("subject") or "(no subject)")[:60]
            att = "Yes" if msg.get("hasAttachments") else "No"

            lines.append(
                f"| {i} | {date_display} | {sender} | {sender_email} "
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

### Add `fetch_messages_paged` to `graph/mail_client.py`

```python
async def fetch_messages_paged(top: int = 50, skip: int = 0) -> list[dict]:
    """
    Fetch messages with $skip offset for pagination support.

    Args:
        top (int): Number of messages per page.
        skip (int): Number of messages to skip (page offset).

    Returns:
        list[dict]: A list of message dictionaries.
    """
    logger.info(f"Fetching messages page: top={top}, skip={skip}")

    client = get_graph_client()

    from msgraph.generated.users.item.messages.messages_request_builder import (
        MessagesRequestBuilder,
    )

    query_params = MessagesRequestBuilder.MessagesRequestBuilderGetQueryParameters(
        top=top,
        skip=skip,
        orderby=["receivedDateTime DESC"],
    )
    request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
        query_parameters=query_params
    )

    response = await client.me.messages.get(request_configuration=request_config)
    messages = response.value if response and response.value else []
    return [_message_to_dict(m) for m in messages]
```

-----

## Point 6 — Reading Sent, Drafts, Deleted, Junk, Archive

**Short answer:** Yes we can add it, and it’s not complex — Graph API
supports reading any mail folder via `/me/mailFolders/{folder}/messages`.
The folder names are:

|Folder       |Graph API folder name|
|-------------|---------------------|
|Inbox        |`inbox`              |
|Sent Items   |`sentitems`          |
|Drafts       |`drafts`             |
|Deleted Items|`deleteditems`       |
|Junk Email   |`junkemail`          |
|Archive      |`archive`            |

**Decision: Defer to Phase 2** for these reasons:

1. Phase 1 scope is inbox only — it’s clean and tested
1. Each folder needs its own tool or a `folder` parameter — adds
   complexity to the tool descriptions the LLM reads
1. Sent items have a different data shape (no `from`, only `toRecipients`)
1. Drafts have no `receivedDateTime` — date handling changes
1. Notes are not email — they’re a separate Graph API endpoint entirely

**What we’ll add in Phase 2:**

- `list_sent_emails` — sent items with `to` field instead of `from`
- `list_folder_emails(folder_name)` — generic folder reader
- `list_draft_emails` — drafts with composition status
- `search_all_folders` — cross-folder search

-----

## Summary of all changes

|File                      |What changed                                                                                                                                        |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
|`tools/email_tools.py`    |`list_emails` returns markdown table; `read_email` returns inbox view; added `search_emails_advanced`, `list_emails_paged`, `export_emails_markdown`|
|`graph/mail_client.py`    |Added `search_messages_advanced`, `fetch_messages_paged`                                                                                            |
|`parsers/parser_router.py`|Added TXT, MD, CSV, image support (Point 2 from previous session)                                                                                   |

**Total tools after these changes: 12**
(was 9 — added `search_emails_advanced`, `list_emails_paged`, `export_emails_markdown`)

-----

## Restart server after all changes

```bash
python server.py
```

Verify health check shows 12 tools:

```bash
curl http://localhost:8000/health
```

Then test in LibreChat:

```
List my last 10 emails
Get emails from jane@company.com between June 10 and June 18
Show me emails 51 to 100
Export my last 100 emails to a file
```