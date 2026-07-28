# FIX-03 — Complete Remediation

**Project:** FiGPT_OutlookMCP
**Target:** `/opt/FiGPT_OutlookMCP` on FGSV2187
**Date:** 26 July 2026
**Companion doc:** `AUDIT-2026-07-26_full-codebase-review.md` (diagnosis — this file is the fixes)

Every outstanding finding, as paste-ready patches, in dependency order.

---

## How to use this document

Work **top to bottom**. Fixes are ordered so nothing depends on something later.

Each fix is self-contained: one **Find**, one **Replace with**, in one file. Where a fix adds a whole new file, the full content is given.

**Three rules learned the hard way this session:**

1. **Select exactly the Find text — no more.** On 26 July a paste over-selected and commented out four dict fields, producing a `KeyError` that took two round trips to find. If your editor auto-selects whole lines, check the boundaries before pasting.
2. **Restart the server after every file change.** Python caches imported modules; editing a file changes nothing until the process restarts.
3. **Run the syntax check before restarting** — it catches a bad paste in one second instead of after a failed startup:
   ```bash
   python -c "import ast; ast.parse(open('tools/email_tools.py').read()); print('OK')"
   ```

### Checklist

| # | Fix | File(s) | Risk |
|---|---|---|---|
| **BLOCK A — the invite path** | | | |
| A1 | Delete the crash block | `graph/availability_client.py` | Low |
| A2 | Stop the client claiming nothing was sent | `graph/availability_client.py` | Low |
| A3 | Split into `compose_invite` / `save_invite` | `tools/draft_tools.py` | **Med** |
| A4 | Wrong confirmation text on email drafts | `tools/draft_tools.py` | Low |
| **BLOCK B — prompt corruption** | | | |
| B1 | `generate_mom` — leaked source, doubled digest | `tools/mom_tools.py` | Low |
| B2 | `extract_tasks` — doubled governance | `tools/task_tools.py` | Low |
| **BLOCK C — the L-01 pattern, everywhere else** | | | |
| C1 | New shared `utils/table_utils.py` | new file | Low |
| C2 | Point `email_tools` at it, drop both local copies | `tools/email_tools.py` | Med |
| C3 | `search_emails` — cells + `is_read` | `tools/email_tools.py` | Low |
| C4 | `search_emails_advanced` — cells + `is_read` | `tools/email_tools.py` | Low |
| C5 | `semantic_search_emails` — cells + `is_read` | `tools/semantic_tools.py` | Low |
| C6 | `track_followups` + `list_tasks` — cells | `tools/followup_tools.py` | Low |
| C7 | `list_draft_emails` — cells + safe key access | `tools/draft_tools.py` | Low |
| C8 | Timezone crash on keyword+date search | `graph/mail_client.py` | Low |
| C9 | `IndexError` on an empty thread | `tools/followup_tools.py` | Low |
| **BLOCK D — follow-ups that work** | | | |
| D1 | `track_followups` actually checks replies | `tools/followup_tools.py` | **Med** |
| D2 | `compose_followup` keeps the thread | `tools/followup_tools.py` | Low |
| **BLOCK E — deployment readiness** | | | |
| E1 | Persistent log file | `utils/logger.py` | Low |
| E2 | Verify secrets are gitignored | commands only | — |
| E3 | Pin dependencies | `requirements.txt` | Low |
| E4 | Fix the env template | `.env.example` | Low |
| E5 | Repair the test runner | `run_all_test.sh` | Low |
| **BLOCK F — hygiene** | | | |
| F1 | `server.py` dead scaffolding | `server.py` | Low |
| F2 | `IndexError` on a bare "1." line | `tools/mom_tools.py` | Low |
| F3 | Audit log leaks task titles | `utils/audit_logger.py` | Low |
| F4 | Dead code removal ×4 | various | Low |
| F5 | `generate_chart` unused parameter | `tools/chart_tools.py` | Low |
| F6 | Deprecated `datetime.utcnow()` | `graph/availability_client.py` | Low |
| F7 | `list_tasks` docstring overpromises | `tools/followup_tools.py` | Low |

---
---

# BLOCK A — The invite path

> **Do A1–A3 as one unit.** A1 alone produces something worse than a crash: a tool that emails external attendees while telling the user it only saved a draft.

## A1 — Delete the crash block

**File:** `graph/availability_client.py` → `create_calendar_draft_invite`, Step 3

**Find:**
```python
    # Step 3: Create the event
    from msgraph.generated.users.item.events.events_request_builder import (
        EventsRequestBuilder,
    )

    query_params = EventsRequestBuilder.EventsRequestBuilderPostQueryParameters(
        send_invitations_or_cancel="sendToAllAndSaveCopy"
    )
    request_config = EventsRequestBuilder.EventsRequestBuilderPostRequestConfiguration(
        query_parameters=query_params
    )

    created = await client.me.events.post(event)
```

**Replace with:**
```python
    # Step 3: Create the event.
    # POST /me/events emails an invitation to every attendee in
    # event.attendees automatically. There is no query parameter to
    # control this, and the SDK exposes no Post query-parameters class —
    # referencing EventsRequestBuilderPostQueryParameters raised
    # AttributeError before any request reached Microsoft.
    #
    # IMPORTANT: event.is_draft must remain unset. A draft event is saved
    # to the calendar but notifies nobody, which is the original reason
    # attendees were not receiving invites.
    created = await client.me.events.post(event)
```

`request_config` was never passed to `.post()`, so removing it changes no behaviour beyond deleting the crash.

## A2 — Stop the Graph client claiming nothing was sent

**File:** `graph/availability_client.py`, the return of `create_calendar_draft_invite`

**Find:**
```python
        "web_link": created.web_link or "",
        "status": "Meeting invite drafted. Review in your Calendar before sending.",
    }
```

**Replace with:**
```python
        "web_link": created.web_link or "",
        # Graph has already emailed every attendee at this point. Saying
        # anything else here misleads the user about an action that has
        # already reached third parties.
        "status": "Meeting created and invitations emailed to all attendees.",
    }
```

## A3 — Split into `compose_invite` and `save_invite`

This implements **D-15**. Every other write path in the system displays first and asks (D-02); the invite path was the only one that didn't — and it's the one that contacts people outside the company.

**File:** `tools/draft_tools.py`

Replace the **entire** `create_draft_invite` tool — from its `# ---` header comment down to its `except` block — with the two tools below.

**Find** (the header line, decorator and signature — this anchors the start):
```python
# ---------------------------------------------------------------------------
# Tool: create_draft_invite
# ---------------------------------------------------------------------------
@mcp.tool
async def create_draft_invite(
```

**Replace with:**
```python
# ---------------------------------------------------------------------------
# Tool: compose_invite
# ---------------------------------------------------------------------------
# Step 1 of 2. Renders the invite in chat and asks for approval.
# Touches no Graph API. Under D-02 every write is composed, approved,
# then saved — and POST /me/events emails attendees the instant it runs,
# so this gate matters more here than anywhere else in the system.
@mcp.tool
async def compose_invite(
    subject: str,
    attendee_emails: str,
    start_datetime: str,
    agenda: str,
    duration_minutes: int = 30,
    location: str = "Microsoft Teams",
    is_online: bool = True,
) -> dict:
    """
    Compose a meeting invite and display it in chat for review.
    Does NOT create the event and does NOT contact anyone yet.

    ALWAYS use this before save_invite. Sending a meeting invitation
    emails every attendee immediately and cannot be undone, so the user
    must see and approve the details first.

    Use this tool when the user asks things like:
    - "Draft a meeting invite for the project kickoff with John and Jane"
    - "Create a calendar invite for our review meeting on Monday at 10am"
    - "Schedule a 30-minute call with the team about the quarterly report"

    Args:
        subject (str): Meeting title — clear and descriptive.
        attendee_emails (str): Comma-separated attendee email addresses.
        start_datetime (str): Meeting start — ISO format or natural
                               language e.g. "Monday 10am".
        agenda (str): Agenda points, used to build the invite body.
        duration_minutes (int): Duration in minutes. Default 30.
        location (str): Location string. Default "Microsoft Teams".
        is_online (bool): Create as an online meeting. Default True.

    Returns:
        dict with a rendered preview and an approval instruction.
    """
    logger.info(f"Tool called: compose_invite (subject={subject})")

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
            tool_name="compose_invite",
            user_email=user_email,
            inputs={"subject": subject, "duration_minutes": duration_minutes},
        )

        # ── Governance rules ─────────────────────────────────────────
        governance = get_draft_rules()

        invite_preview = (
            f"### 📅 Meeting Invite — review before sending\n\n"
            f"| Field | Details |\n"
            f"|---|---|\n"
            f"| **Subject** | {subject} |\n"
            f"| **Start** | {start_datetime} |\n"
            f"| **Duration** | {duration_minutes} minutes |\n"
            f"| **Attendees** | {', '.join(email_list)} |\n"
            f"| **Location** | {location} |\n"
            f"| **Online** | {'Yes (Teams)' if is_online else 'No'} |\n\n"
            f"**Agenda:**\n\n{agenda}\n"
        )

        return {
            "subject": subject,
            "attendees": email_list,
            "start_datetime": start_datetime,
            "duration_minutes": duration_minutes,
            "location": location,
            "is_online": is_online,
            "agenda": agenda,
            "invite_preview": invite_preview,
            "instruction": (
                f"Display 'invite_preview' as markdown, exactly as given.\n\n"
                f"Then warn the user clearly:\n"
                f"'⚠️ Saving this will create the event AND immediately email "
                f"an invitation to every attendee. This cannot be undone.'\n\n"
                f"Then ask: 'Shall I send this invite?'\n\n"
                f"Only if the user explicitly confirms, call save_invite with:\n"
                f"- subject: '{subject}'\n"
                f"- attendee_emails: '{', '.join(email_list)}'\n"
                f"- start_datetime: '{start_datetime}'\n"
                f"- agenda: the agenda exactly as shown above\n"
                f"- duration_minutes: {duration_minutes}\n"
                f"- location: '{location}'\n"
                f"- is_online: {is_online}\n\n"
                f"Never call save_invite without an explicit yes from the user.\n\n"
                f"{governance}"
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="compose_invite", logger=logger)


# ---------------------------------------------------------------------------
# Tool: save_invite
# ---------------------------------------------------------------------------
# Step 2 of 2. Creates the event, which sends the invitations.
# Only reachable after the user has approved a compose_invite preview.
@mcp.tool
async def save_invite(
```

Now the **second half**. The original function body still follows — you only need to change its docstring and its return block.

**Find:**
```python
    """
    Draft a professional meeting invite with agenda and save it to Calendar.
    The invite is NOT sent until the user confirms in Outlook Calendar.

    Use this tool when the user asks things like:
    - "Draft a meeting invite for the project kickoff with John and Jane"
    - "Create a calendar invite for our review meeting on Monday at 10am"
    - "Schedule a 30-minute call with the team about the quarterly report"
```

**Replace with:**
```python
    """
    Create the meeting event and SEND invitations to all attendees.

    ⚠️ This immediately emails every attendee. It cannot be undone.
    Only call this after compose_invite has displayed the details and
    the user has explicitly confirmed. Never call it directly from a
    user's first request.

    Use this tool ONLY when the user has just approved a composed
    invite, saying things like:
    - "Yes, send it"
    - "Go ahead and send the invite"
    - "Confirmed, send it to them"
```

**Find:**
```python
        logger.info(f"Tool called: create_draft_invite (subject={subject})")
```

> If your file has this line without the leading spaces shown, match whatever indentation is actually there.

**Replace with:**
```python
        logger.info(f"Tool called: save_invite (subject={subject})")
```

**Find:**
```python
        log_tool_call(
            tool_name="create_draft_invite",
```

**Replace with:**
```python
        log_tool_call(
            tool_name="save_invite",
```

Now the preview block and return. **Find:**
```python
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
                f"Display 'invite_preview' as markdown. "
                f"Confirm it has been saved to Calendar for review.\n\n"
                f"{governance}"
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="create_draft_invite", logger=logger)
```

**Replace with:**
```python
        # Confirmation of an action that has ALREADY reached the attendees.
        # Worded in the past tense on purpose — the previous version said
        # "Draft saved… review and send from Outlook", which was untrue:
        # POST /me/events emails everyone the moment it runs.
        sent_confirmation = (
            f"### ✅ Meeting Invite Sent\n\n"
            f"| Field | Details |\n"
            f"|---|---|\n"
            f"| **Subject** | {subject} |\n"
            f"| **Time** | {result['start']} – {result['end']} |\n"
            f"| **Duration** | {duration_minutes} minutes |\n"
            f"| **Invitations emailed to** | {', '.join(email_list)} |\n"
            f"| **Location** | {location} |\n"
            f"| **Online** | {'Yes (Teams)' if is_online else 'No'} |\n\n"
            f"**Agenda:**\n\n{agenda}\n\n"
            f"*The event is in your calendar and all attendees have been "
            f"notified. To cancel, delete the event in Outlook — attendees "
            f"will receive a cancellation.*\n"
        )

        return {
            **result,
            "sent_confirmation": sent_confirmation,
            "instruction": (
                f"Display 'sent_confirmation' as markdown, exactly as given. "
                f"Make clear the invitations have already been emailed — do "
                f"not describe this as a draft.\n\n"
                f"{governance}"
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="save_invite", logger=logger)
```

**Tool count goes 35 → 36.** Check `curl http://localhost:8000/health` after restarting.

> **If you get a clean `403` from `save_invite`** rather than a crash, the code is correct and `Calendars.ReadWrite` is not granted (B-02). Run the curl permission test in `FIX-01` §2.3 to confirm, then request the grant from your Azure admin.

## A4 — Wrong confirmation text on email drafts

**File:** `tools/draft_tools.py` → `save_draft_to_outlook`

**Find:**
```python
                f"Display this confirmation:\n\n"
                f"✅ *Meeting invite created and notifications sent to all attendees.*"
                f"| Field | Details |\n"
```

**Replace with:**
```python
                f"Display this confirmation:\n\n"
                f"**✅ Draft Saved to Outlook**\n\n"
                f"| Field | Details |\n"
```

Two bugs in one line: it told the user attendees were notified when saving an ordinary email, and it had no trailing `\n` so it broke the table below it.

---
---

# BLOCK B — Prompt corruption

## B1 — `generate_mom`

Three problems in nine lines: leaked f-string source, the **entire email digest sent twice**, and governance injected twice.

**File:** `tools/mom_tools.py` → `generate_mom`

**Find:**
```python
═══════════════════════════════════════════════════
f"{governance}\n\n"
f"Email content to base the MOM on:\n\n{email_digest}"

Email content to base the MOM on:

{email_digest}

Rules:
- Be factual — only include what is in the emails
- Do not invent names, dates, or decisions
- {lang_instruction}
- Keep Action Items table populated even if only 1 item found
- Format output as clean markdown

{get_mom_rules(include_sources=True)}
"""
```

**Replace with:**
```python
═══════════════════════════════════════════════════

Rules:
- Be factual — only include what is in the emails
- Do not invent names, dates, or decisions
- {lang_instruction}
- Keep Action Items table populated even if only 1 item found
- Format output as clean markdown

{governance}

Email content to base the MOM on:

{email_digest}
"""
```

What changed:

- **The two `f"..."` lines are gone.** They were meant to be Python string concatenation but ended up *inside* the triple-quoted literal. Because the block is an f-string, `{governance}` and `{email_digest}` interpolated — but the surrounding `f"`, `\n\n` and `"` were emitted as literal characters into the prompt.
- **The digest now appears once.** It was being sent twice, doubling the token cost of every MOM call on a long thread.
- **`{get_mom_rules(include_sources=True)}` replaced with `{governance}`** — which is already the result of that exact call, made at the top of the function. It was running twice and injecting the rules twice.
- **Digest moved last.** Long context at the end of a prompt is more reliably attended to than buried mid-instruction.

## B2 — `extract_tasks`

**File:** `tools/task_tools.py` → `extract_tasks`

**Find:**
```python
            f"{governance}\n\n"
            f"Email digest to analyse:\n\n{full_digest}"
            f"{get_task_rules(include_sources=True)}"
        )
```

**Replace with:**
```python
            f"{governance}\n\n"
            f"Email digest to analyse:\n\n{full_digest}"
        )
```

`governance` on the first line is already `get_task_rules(include_sources=True)`, assigned a few lines above. The trailing call repeated the whole rule block — and with no separator, glued straight onto the end of the digest.

---
---

# BLOCK C — The L-01 pattern, everywhere else

## C1 — New shared helper

**Create a new file:** `utils/table_utils.py`

```python
"""
table_utils.py
==============
Helpers for building markdown tables that survive real-world email data.

Why this file exists:
    Every list-style tool in this project renders a markdown table for
    LibreChat. Markdown tables are line-based — one row per line — so a
    single \\r\\n inside an email preview splits the row and mangles the
    whole table. A literal '|' in a subject opens a phantom column.

    This was found in list_emails on 26 July 2026: previews containing
    "Dear X,\\r\\n\\r\\nPlease hold on..." were breaking every table the
    tool produced. The same pattern existed in six other tools, so the
    fix lives here rather than being copy-pasted seven times.

Design notes:
    - Collapse whitespace BEFORE truncating, so the character budget is
      spent on visible text rather than invisible line breaks.
    - Truncate last, so the ellipsis reflects the length the user
      actually sees.
"""


def table_cell(text: str, limit: int) -> str:
    """
    Make a string safe to drop into a markdown table cell.

    Args:
        text (str): Raw text — may contain newlines, tabs, or pipes.
                     None is tolerated and becomes an empty string.
        limit (int): Maximum visible characters before truncation.

    Returns:
        str: Single-line, pipe-escaped, truncated text.
    """
    flat = " ".join((text or "").split()).replace("|", "\\|")
    return flat[:limit] + ("…" if len(flat) > limit else "")
```

## C2 — Point `email_tools` at it and remove both local copies

`list_emails` has a local `_cell()` and the module has a `_table_cell()`. Both are replaced by the shared one.

**File:** `tools/email_tools.py`

### C2.1 — Add the import

**Find:**
```python
from utils.error_handler import format_tool_error
```

**Replace with:**
```python
from utils.error_handler import format_tool_error
from utils.table_utils import table_cell
```

> If `email_tools.py` imports `format_tool_error` differently, add the `table_cell` line next to any other `from utils.` import at the top of the file. Placement doesn't matter as long as it's at module level.

### C2.2 — Remove the local `_cell()` inside `list_emails`

**Find:**
```python
        # Build markdown table for clean display in LibreChat
        def _cell(text: str, limit: int) -> str:
            """
            Make a string safe to drop into a markdown table cell.

            Markdown tables are line-based — one row per line — so a \\r\\n
            inside an email preview splits the row and breaks the whole
            table. A literal '|' opens a phantom column. Collapse every
            run of whitespace to a single space, escape pipes, then
            truncate. Truncating last means the ellipsis reflects the
            length the user actually sees.
            """
            flat = " ".join((text or "").split()).replace("|", "\\|")
            return flat[:limit] + ("…" if len(flat) > limit else "")

        table_lines = [
```

**Replace with:**
```python
        # Build markdown table for clean display in LibreChat
        table_lines = [
```

### C2.3 — Update its three call sites

**Find:**
```python
            sender_col = _cell(m["sender"], 40)
            subject_col = _cell(m["subject"], 50)
            preview_col = _cell(m["preview"], 80)
```

**Replace with:**
```python
            sender_col = table_cell(m["sender"], 40)
            subject_col = table_cell(m["subject"], 50)
            preview_col = table_cell(m["preview"], 80)
```

### C2.4 — Remove the module-level `_table_cell()`

**Find:**
```python
# ---------------------------------------------------------------------------
# Helper: _table_cell
# ---------------------------------------------------------------------------
def _table_cell(text: str, limit: int) -> str:
    """
    Make a string safe to drop into a markdown table cell.

    Markdown tables are line-based — one row per line — so a \\r\\n inside
    an email preview or subject splits the row and breaks the whole table.
    A literal '|' opens a phantom column. Collapse every run of whitespace
    to a single space, escape pipes, then truncate. Truncating last means
    the ellipsis reflects the length the user actually sees.
    """
    flat = " ".join((text or "").split()).replace("|", "\\|")
    return flat[:limit] + ("…" if len(flat) > limit else "")


# ---------------------------------------------------------------------------
# Tool: list_emails_paged
# ---------------------------------------------------------------------------
```

**Replace with:**
```python
# ---------------------------------------------------------------------------
# Tool: list_emails_paged
# ---------------------------------------------------------------------------
```

### C2.5 — Update its call sites in `list_emails_paged`

**Find:**
```python
            sender_col = _table_cell(m["sender"], 40)
            subject_col = _table_cell(m["subject"], 55)
```

**Replace with:**
```python
            sender_col = table_cell(m["sender"], 40)
            subject_col = table_cell(m["subject"], 55)
```

> **Verify before restarting:** `grep -n "_cell\|_table_cell" tools/email_tools.py` should return nothing.

## C3 — `search_emails`: cells + `is_read`

**File:** `tools/email_tools.py` → `search_emails`

### C3.1 — Add the field

**Find:**
```python
                "has_attachments": msg.get("hasAttachments", False),
                "preview": msg.get("bodyPreview", "")[:150],
            })

        # Build markdown table for clean display in LibreChat
        table_lines = [
            "| # | Date | From | Subject | Preview | 📎 |",
```

**Replace with:**
```python
                "has_attachments": msg.get("hasAttachments", False),
                # Mirrors list_emails so "which of these are unread?"
                # works on search results too. Defaults to True (read) so
                # a missing field never reports a read mail as unread.
                "is_read": msg.get("is_read", True),
                "preview": msg.get("bodyPreview", "")[:150],
            })

        # Build markdown table for clean display in LibreChat
        table_lines = [
            "| # | 🔵 | Date | From | Subject | Preview | 📎 |",
```

### C3.2 — Header separator and row builder

**Find:**
```python
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
```

**Replace with:**
```python
            "|---|---|------|------|---------|---------|-----|",
        ]
        for i, m in enumerate(result, 1):
            att = "✅" if m["has_attachments"] else "—"
            unread = "🔵" if not m["is_read"] else "⚪"
            sender_col = table_cell(m["sender"], 40)
            subject_col = table_cell(m["subject"], 50)
            preview_col = table_cell(m["preview"], 80)
            table_lines.append(
                f"| {i} | {unread} | {m['date_display']} | {sender_col} "
                f"| {subject_col} | {preview_col} | {att} |"
            )
```

### C3.3 — Instruction

**Find:**
```python
            "instruction": (
                "Display 'display_table' as markdown. "
                "Then offer to open or summarise any specific result."
            ),
        }

        # ── Validate output ───────────────────────────────────────────
        validation = validate_email_list(result)
        return append_validation_to_result(final_result, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="search_emails", logger=logger)
```

**Replace with:**
```python
            "instruction": (
                "Display 'display_table' as markdown. "
                "🔵 = Unread, ⚪ = Read. "
                "Every object in the 'emails' list carries a boolean "
                "'is_read' field: false = unread, true = read. Never reply "
                "that read status is unavailable — it is present on every "
                "email object. "
                "Then offer to open or summarise any specific result."
            ),
        }

        # ── Validate output ───────────────────────────────────────────
        validation = validate_email_list(result)
        return append_validation_to_result(final_result, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="search_emails", logger=logger)
```

## C4 — `search_emails_advanced`: cells + `is_read`

**File:** `tools/email_tools.py` → `search_emails_advanced`

**Find:**
```python
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
```

**Replace with:**
```python
                "has_attachments": msg.get("hasAttachments", False),
                "is_read": msg.get("is_read", True),
                "preview": msg.get("bodyPreview", "")[:150],
            })

        # Build table
        table_lines = [
            f"**Search Results** — {len(result)} email(s) found\n",
            "| # | 🔵 | Date | From | Subject | 📎 |",
            "|---|---|------|------|---------|-----|",
        ]
        for i, m in enumerate(result, 1):
            att = "✅" if m["has_attachments"] else "—"
            unread = "🔵" if not m["is_read"] else "⚪"
            sender_col = table_cell(m["sender"], 40)
            subject_col = table_cell(m["subject"], 55)
            table_lines.append(
                f"| {i} | {unread} | {m['date_display']} | {sender_col} "
                f"| {subject_col} | {att} |"
            )
```

And its instruction. **Find:**
```python
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "Then offer to open or summarise any specific result."
            ),
        }

        # ── Validate output ───────────────────────────────────────────
        from utils.validator import validate_email_list, append_validation_to_result
```

**Replace with:**
```python
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "🔵 = Unread, ⚪ = Read. Every object in the 'emails' list "
                "carries a boolean 'is_read' field. "
                "Then offer to open or summarise any specific result."
            ),
        }

        # ── Validate output ───────────────────────────────────────────
        from utils.validator import validate_email_list, append_validation_to_result
```

## C5 — `semantic_search_emails`

**File:** `tools/semantic_tools.py`

### C5.1 — Import

**Find:**
```python
from utils.error_handler import format_tool_error
from utils.logger import get_logger
```

**Replace with:**
```python
from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.table_utils import table_cell
```

### C5.2 — Row builder

**Find:**
```python
            subject = (email.get("subject") or "(no subject)")[:55]
            relevance = email.get("relevance", "—")
            score = email.get("similarity_score", 0)

            table_lines.append(
                f"| {i} | {relevance} ({score:.2f}) | {date_display} "
                f"| {sender} | {subject} |"
            )
```

**Replace with:**
```python
            subject = email.get("subject") or "(no subject)"
            relevance = email.get("relevance", "—")
            score = email.get("similarity_score", 0)

            table_lines.append(
                f"| {i} | {relevance} ({score:.2f}) | {date_display} "
                f"| {table_cell(sender, 40)} | {table_cell(subject, 55)} |"
            )
```

### C5.3 — `is_read` on the results

**Find:**
```python
                "has_attachments": email.get("hasAttachments", False),
                "preview": email.get("bodyPreview") or email.get("preview") or "",
```

**Replace with:**
```python
                "has_attachments": email.get("hasAttachments", False),
                "is_read": email.get("is_read", True),
                "preview": email.get("bodyPreview") or email.get("preview") or "",
```

## C6 — `track_followups` and `list_tasks`

**File:** `tools/followup_tools.py`

### C6.1 — Import

**Find:**
```python
from utils.error_handler import format_tool_error
from utils.logger import get_logger
```

**Replace with:**
```python
from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.table_utils import table_cell
```

### C6.2 — `track_followups` row

**Find:**
```python
            sender = (
                msg.get("from", {}).get("emailAddress", {}).get("name") or "—"
            )
            subject = (msg.get("subject") or "(no subject)")[:50]

            urgency = "🔴" if isinstance(age_days, int) and age_days >= 7 else "🟡"

            table_lines.append(
                f"| {i} | {date_display} | {sender} "
                f"| {subject} | {urgency} {age_days} days |"
            )
```

**Replace with:**
```python
            sender = (
                msg.get("from", {}).get("emailAddress", {}).get("name") or "—"
            )
            subject = msg.get("subject") or "(no subject)"

            urgency = "🔴" if isinstance(age_days, int) and age_days >= 7 else "🟡"

            table_lines.append(
                f"| {i} | {date_display} | {table_cell(sender, 40)} "
                f"| {table_cell(subject, 50)} | {urgency} {age_days} days |"
            )
```

### C6.3 — `list_tasks` row

**Find:**
```python
            title = (task.get("title") or "")[:60]
            table_lines.append(
                f"| {i} | {title} | {due} | {status_icon} {status} |"
            )
```

**Replace with:**
```python
            title = task.get("title") or ""
            table_lines.append(
                f"| {i} | {table_cell(title, 60)} | {due} "
                f"| {status_icon} {status} |"
            )
```

## C7 — `list_draft_emails`: cells + safe key access

Bracket indexing on Graph-derived dicts is exactly the failure mode that cost a round trip on 26 July.

**File:** `tools/draft_tools.py`

### C7.1 — Import

**Find:**
```python
from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.governance import get_draft_rules
```

**Replace with:**
```python
from utils.error_handler import format_tool_error
from utils.logger import get_logger
from utils.governance import get_draft_rules
from utils.table_utils import table_cell
```

### C7.2 — The row builder

**Find:**
```python
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
```

**Replace with:**
```python
        for i, d in enumerate(drafts, 1):
            # .get() throughout — a draft with no recipients or no subject
            # is perfectly normal in Outlook, and a missing key from the
            # Graph layer should not take the whole tool down.
            last_modified = d.get("last_modified") or ""

            try:
                from dateutil import parser as dp
                dt = dp.parse(last_modified)
                date_display = dt.strftime("%d %b %Y %H:%M")
            except Exception:
                date_display = last_modified[:10] if last_modified else "—"

            recipients = d.get("to") or []
            to_display = ", ".join(recipients) if recipients else "(no recipients)"

            table_lines.append(
                f"| {i} | {date_display} | {table_cell(to_display, 40)} "
                f"| {table_cell(d.get('subject'), 50)} "
                f"| {table_cell(d.get('preview'), 60)} |"
            )
```

## C8 — Timezone crash on keyword + date search

`msg_dt` was computed and never used; the comparison below it used the raw tz-aware value against a naive one.

**File:** `graph/mail_client.py` → `search_messages_advanced`

**Find:**
```python
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
```

**Replace with:**
```python
            # Graph returns tz-aware datetimes; parse_date_string returns
            # naive ones. Comparing the two raises
            #   TypeError: can't compare offset-naive and offset-aware
            # Strip tzinfo from BOTH sides — .replace(tzinfo=None) is a
            # no-op on an already-naive value, so this is safe either way.
            if date_from:
                dt_from = parse_date_string(date_from)
                if dt_from and msg.received_date_time:
                    msg_dt = msg.received_date_time.replace(tzinfo=None)
                    if msg_dt < dt_from.replace(tzinfo=None):
                        continue
            if date_to:
                dt_to = parse_date_string(date_to)
                if dt_to and msg.received_date_time:
                    from datetime import timedelta
                    msg_dt = msg.received_date_time.replace(tzinfo=None)
                    if msg_dt > dt_to.replace(tzinfo=None) + timedelta(days=1):
                        continue
```

**Test after applying:** `Find emails from anyone about "report" between 1 July and 26 July` — this path threw a `TypeError` before.

## C9 — `IndexError` on an empty thread

**File:** `tools/followup_tools.py` → `check_email_replied`

**Find:**
```python
        # Step 3: Check if any message in the thread was sent by the user
        target_msg = next(
            (m for m in thread if m.get("id") == email_id), thread[0]
        )
```

**Replace with:**
```python
        # Step 3: Check if any message in the thread was sent by the user.
        # Guard the empty case first: Python evaluates next()'s default
        # argument EAGERLY, so `thread[0]` as a default raises IndexError
        # on an empty thread before the generator ever runs.
        if not thread:
            return {
                "has_replied": False,
                "reply_count": 0,
                "status_display": "⚠️ No thread found for this email.",
                "instruction": (
                    "Tell the user the conversation thread could not be "
                    "retrieved — the email may have been deleted or moved. "
                    "Suggest they list emails again to get a fresh ID."
                ),
            }

        target_msg = next(
            (m for m in thread if m.get("id") == email_id), thread[0]
        )
```

---
---

# BLOCK D — Follow-ups that actually work

## D1 — `track_followups` checks replies

Today `_needs_followup()` filters on age and "not sent by me" only. It never checks whether you replied — while the tool's own output claims *"Emails older than N days with no reply sent"*. Emails you already answered keep reappearing.

`check_email_replied` already does this properly. It just was never wired in.

**File:** `tools/followup_tools.py`

### D1.1 — Add the reply-check helper

**Find:**
```python
# ---------------------------------------------------------------------------
# Tool: track_followups
# ---------------------------------------------------------------------------
@mcp.tool
async def track_followups(
```

**Replace with:**
```python
# ---------------------------------------------------------------------------
# Helper: _user_has_replied
# ---------------------------------------------------------------------------
async def _user_has_replied(email_id: str, user_email: str) -> bool:
    """
    Check whether the logged-in user has sent any message in this
    email's conversation thread.

    This is the check the docstring of track_followups always promised
    and never performed. It costs one extra Graph call per candidate,
    which is why track_followups caps the candidate list before calling
    it — see MAX_THREAD_CHECKS.

    Args:
        email_id (str): Graph API ID of the email to check.
        user_email (str): The logged-in user's email address.

    Returns:
        bool: True if the user has replied in this thread. Returns False
              on any error — a failed check should surface the email as
              needing follow-up rather than silently hiding it.
    """
    try:
        thread = await fetch_email_thread(email_id)
        if not thread:
            return False

        user_lower = (user_email or "").lower()
        return any(
            m.get("sender_email", "").lower() == user_lower
            and m.get("id") != email_id
            for m in thread
        )
    except Exception as exc:
        logger.warning(
            f"Reply check failed for {email_id[:12]}...: {exc}. "
            f"Treating as not replied."
        )
        return False


# ---------------------------------------------------------------------------
# Tool: track_followups
# ---------------------------------------------------------------------------
# Candidates are filtered cheaply by age first, then only this many get
# a thread lookup. Each lookup is a Graph round trip, so an uncapped
# scan of 50 inbox emails would mean 50 extra API calls per invocation.
MAX_THREAD_CHECKS = 15


@mcp.tool
async def track_followups(
```

### D1.2 — Wire it into the filter

**Find:**
```python
        # Step 3: Filter emails needing follow-up
        needs_reply = [
            m for m in raw_messages
            if _needs_followup(m, user_email, threshold)
        ]
```

**Replace with:**
```python
        # Step 3a: Cheap filter first — age, and not sent by the user.
        candidates = [
            m for m in raw_messages
            if _needs_followup(m, user_email, threshold)
        ]

        # Step 3b: Now the expensive part. Only emails that survived the
        # age filter get a thread lookup, capped at MAX_THREAD_CHECKS and
        # run concurrently so the wall-clock cost is one round trip, not N.
        import asyncio

        to_check = candidates[:MAX_THREAD_CHECKS]
        overflow = candidates[MAX_THREAD_CHECKS:]

        if overflow:
            logger.info(
                f"{len(candidates)} follow-up candidates; checking threads "
                f"for the first {MAX_THREAD_CHECKS}. The remaining "
                f"{len(overflow)} are reported on age alone."
            )

        replied_flags = await asyncio.gather(*[
            _user_has_replied(m.get("id", ""), user_email)
            for m in to_check
        ])

        needs_reply = [
            m for m, replied in zip(to_check, replied_flags) if not replied
        ] + overflow
```

### D1.3 — Make the output honest about the cap

**Find:**
```python
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
```

**Replace with:**
```python
        return {
            "followup_count": len(needs_reply),
            "emails_scanned": len(raw_messages),
            "threshold_days": threshold,
            "reply_checked": len(to_check),
            "reply_check_capped": bool(overflow),
            "followups": followup_list,
            "display_table": "\n".join(table_lines),
            "instruction": (
                "Display 'display_table' as markdown. "
                "🔴 = 7+ days overdue, 🟡 = 3–6 days. "
                + (
                    f"Note: reply status was verified for the "
                    f"{len(to_check)} oldest candidates only; the rest are "
                    f"listed on age alone and may already be answered. "
                    if overflow else
                    "Every email listed has been verified as having no "
                    "reply from the user in its thread. "
                )
                + "Offer to: compose a follow-up, add to tasks, or both."
            ),
        }
```

### D1.4 — Correct the helper's docstring

**Find:**
```python
def _needs_followup(msg: dict, user_email: str, threshold_days: int) -> bool:
    """
    Determine if an email needs a follow-up based on age and
    whether the current user has already replied.
```

**Replace with:**
```python
def _needs_followup(msg: dict, user_email: str, threshold_days: int) -> bool:
    """
    Cheap first-pass filter: is this email old enough to be a follow-up
    candidate, and was it sent by someone other than the user?

    This does NOT check whether a reply was sent — that requires a thread
    lookup per email. track_followups applies this filter first, then
    calls _user_has_replied() on the survivors.
```

## D2 — `compose_followup` keeps the thread

This is L-03 in a second tool. `compose_reply` was hardened to insist on `save_reply_draft`; `compose_followup` still routes to `save_draft_to_outlook`, creating a standalone email with a cosmetic `Re:` prefix.

**File:** `tools/followup_tools.py` → `compose_followup`

**Find:**
```python
                f"After composing, ask:\n"
                f"'Would you like me to save this follow-up to Outlook Drafts?\n"
                f"If yes, call save_draft_to_outlook with:\n"
                f"- to_emails: '{original_sender_email}'\n"
                f"- subject: 'Re: {original_subject}'\n"
                f"- body_text: the exact composed follow-up"
```

**Replace with:**
```python
                f"After composing, ask:\n"
                f"'Would you like me to save this follow-up to Outlook Drafts?'\n"
                f"IMPORTANT: If yes, you MUST call save_reply_draft "
                f"(NOT save_draft_to_outlook) with:\n"
                f"- email_id: '{email_id}'\n"
                f"- body_text: the exact composed follow-up\n"
                f"A follow-up belongs to an existing conversation. "
                f"save_draft_to_outlook creates a standalone email with a "
                f"cosmetic 'Re:' prefix and breaks the thread chain."
```

The original also had an unclosed single quote on the "Would you like" line — fixed above.

---
---

# BLOCK E — Deployment readiness

## E1 — Persistent log file

Right now every traceback exists only in the terminal scrollback of a manually-started server. We hit this directly on 26 July.

**File:** `utils/logger.py`

**Find:**
```python
import logging
import sys
from config.settings import settings
```

**Replace with:**
```python
import logging
import sys
from pathlib import Path
from logging.handlers import TimedRotatingFileHandler

from config.settings import settings

# Application log — separate from logs/audit.log, which is written by
# utils/audit_logger.py with propagate=False and holds only structured
# audit entries, never tracebacks.
APP_LOG_DIR = Path("./logs")
APP_LOG_FILE = APP_LOG_DIR / "app.log"
```

**Find:**
```python
    # Step 2c: Apply the standard format defined above.
    formatter = logging.Formatter(LOG_FORMAT, datefmt=DATE_FORMAT)
    handler.setFormatter(formatter)

    root_logger.addHandler(handler)
```

**Replace with:**
```python
    # Step 2c: Apply the standard format defined above.
    formatter = logging.Formatter(LOG_FORMAT, datefmt=DATE_FORMAT)
    handler.setFormatter(formatter)

    root_logger.addHandler(handler)

    # Step 2d: Also write to a rotating file.
    #
    # Without this, the ONLY copy of every traceback lives in the
    # scrollback of whichever terminal started the server — and the
    # server is started by hand over SSH (B-04), so it dies with the
    # session. Diagnosing the 26 July KeyError depended on that terminal
    # still being open. Keeps 14 days.
    #
    # Wrapped in try/except: if ./logs is not writable the server must
    # still start, just without file logging.
    try:
        APP_LOG_DIR.mkdir(parents=True, exist_ok=True)
        file_handler = TimedRotatingFileHandler(
            APP_LOG_FILE,
            when="midnight",
            interval=1,
            backupCount=14,
            encoding="utf-8",
        )
        file_handler.setLevel(log_level)
        file_handler.setFormatter(formatter)
        root_logger.addHandler(file_handler)
    except Exception as file_log_err:
        root_logger.warning(
            f"Could not create the application log file at {APP_LOG_FILE}: "
            f"{file_log_err}. Continuing with console logging only."
        )
```

**Verify:**
```bash
python server.py &
sleep 5
tail -5 logs/app.log
```

> **Add `logs/` to `.gitignore` before doing this** — see E2. You are about to start writing a second file of potentially sensitive data into the repo tree.

## E2 — Verify secrets are gitignored

No patch — these are checks. **Run them before E1.**

```bash
cd /opt/FiGPT_OutlookMCP

# Are they ignored right now?
git check-ignore -v logs/audit.log bin/del.txt bin/token.json

# Were they EVER committed? .gitignore does not remove history.
git log --oneline -- logs/ bin/token.json bin/del.txt
```

| Result | Meaning |
|---|---|
| First command prints a `.gitignore` line for each | ✅ Ignored |
| First command prints nothing for a path | ❌ Not ignored — add it now |
| Second command prints commits | ⚠️ Already in Azure DevOps history — needs a separate conversation |

If anything is unignored, append to `.gitignore`:

```gitignore
# Logs — audit.log contains real user email addresses,
# app.log can contain email subjects inside tracebacks
logs/
*.log

# Tokens and scratch files
bin/token.json
bin/*.txt
bin/agent_output.txt

# Downloaded attachments and generated charts
temp_attachments/
```

Then remove them from tracking without deleting them from disk:

```bash
git rm -r --cached logs/ temp_attachments/ bin/token.json 2>/dev/null
git status
```

`bin/del.txt` holds a full Graph JWT. It is expired, but delete it regardless — `rm bin/del.txt`.

## E3 — Pin dependencies

`fastmcp>=2.0.0` spans the 2→3 major that removed constructor `host`/`port` — the exact change behind the HTTP 421 saga. A clean install today could resolve to 4.x.

**File:** `requirements.txt` — replace the whole file:

```
# =============================================================================
# FiGPT_OutlookMCP — dependencies
#
# Pinned with compatible-release bounds (~=), NOT open-ended >=.
#
# Why: fastmcp>=2.0.0 spanned the 2 -> 3 major that removed host/port from
# the FastMCP constructor and started rejecting cors_origins /
# transport_security. That change cost days of HTTP 421 debugging. An
# open-ended pin means the next clean install can reproduce it.
#
# ~=3.4.3 means ">=3.4.3, <3.5.0" — patches yes, minors no.
# ~=3.4   means ">=3.4, <4.0"     — minors yes, majors no.
#
# Regenerate the exact set with:  pip freeze > requirements.lock.txt
# =============================================================================

# Core MCP framework — see note above before widening this
fastmcp~=3.4.3

# Microsoft authentication
msal~=1.28

# Microsoft Graph SDK
msgraph-sdk~=1.5

# Environment / config loading
python-dotenv~=1.0

# HTTP client
httpx~=0.27

# Data validation
pydantic~=2.7

# Date parsing utility
python-dateutil~=2.9

# Attachment parsing — PDF
# NOTE: pymupdf is AGPL-licensed. Flag for OSS compliance review before
# production rollout (D-08).
pymupdf~=1.24

# Attachment parsing — Word
python-docx~=1.1

# Attachment parsing — PowerPoint
python-pptx~=0.6.23

# Attachment parsing — Excel
openpyxl~=3.1

# OCR fallback (optional, scanned documents)
pytesseract~=0.3.10
pillow~=10.3

# Test runner — pinned to the versions the suite is green on.
# pytest-asyncio runs in STRICT mode; see D-03 before upgrading.
pytest~=9.1.1
pytest-asyncio~=1.4.0

# Semantic search
sentence-transformers~=2.7
numpy~=1.26
scikit-learn~=1.4

# Multi-language support
langdetect~=1.0.9

# NOTE: matplotlib was removed on 26 Jul 2026. Charts are generated as
# Mermaid text by tools/chart_tools.py — no image rendering, no
# matplotlib. Do not re-add it without checking chart_tools first.
```

Then:

```bash
source .venv/bin/activate
pip install -r requirements.txt      # confirm nothing is downgraded
pip freeze > requirements.lock.txt
python -m pytest tests/ -v           # still 90 passed
```

> If `pip install` wants to **downgrade** something, your installed version is newer than the `~=` ceiling. Widen that one line to match reality rather than downgrading a working server.

## E4 — Fix the env template

Currently ships `FASTMCP_HOST=127.0.0.1` with the required `0.0.0.0` commented out. Provisioning a second environment from this file reproduces the HTTP 421 bug.

**File:** `.env.example`

**Find:**
```
MCP_HOST=0.0.0.0
MCP_PORT=8000
MCP_SERVER_NAME=outlook-ai-agent-mcp
MCP_TRANSPORT=streamable-http
MCP_TRANSPORT=http

# FASTMCP_HOST=0.0.0.0
FASTMCP_HOST=127.0.0.1
FASTMCP_PORT=8000
```

**Replace with:**
```
MCP_HOST=0.0.0.0
MCP_PORT=8000
MCP_SERVER_NAME=outlook-ai-agent-mcp

# Plain "http" — NOT "streamable-http".
# "streamable-http" is what LibreChat's client config uses; the SERVER
# side must be "http" or the nginx proxy will not connect.
MCP_TRANSPORT=http

# REQUIRED — do not change to 127.0.0.1.
# FastMCP 3.x has DNS-rebinding protection that returns HTTP 421 unless
# it binds 0.0.0.0. FastMCP 3.x also removed host/port from the
# constructor, so these env vars are the only way to set them.
FASTMCP_HOST=0.0.0.0
FASTMCP_PORT=8000
```

**Find:**
```
GRAPH_API_BASE_URL=https://graph.microsoft.com/v1.0
GRAPH_API_SCOPES=Mail.Read,Calendars.Read,User.Read
```

**Replace with:**
```
GRAPH_API_BASE_URL=https://graph.microsoft.com/v1.0
# Calendars.ReadWrite is required by save_invite. Tasks.ReadWrite and
# Group.Read.All are required by the Planner tools and are NOT yet
# granted (B-01).
GRAPH_API_SCOPES=Mail.Read,Mail.ReadWrite,Mail.Send,Calendars.Read,Calendars.ReadWrite,Calendars.Read.Shared,User.Read
```

Then **delete** the now-duplicated block further down the file:

**Find:**
```
# =================================================================
# GRAPH API SCOPES (update existing line)
# =================================================================
GRAPH_API_SCOPES=Mail.Read,Mail.ReadWrite,Calendars.Read,Calendars.ReadWrite,Calendars.Read.Shared,User.Read
```

**Replace with:** *(nothing — delete these four lines)*

## E5 — Repair the test runner

Two independent breakages: smart quotes inside the embedded Python, and a pytest command split across lines with no continuations, so each flag ran as its own command with an en-dash instead of a hyphen.

**File:** `run_all_test.sh` — replace the whole file:

```bash
#!/bin/bash
# =============================================================================
# run_all_test.sh
# Full test suite runner — lists all registered tools, then runs all tests.
# Usage: bash run_all_test.sh
#
# NOTE: this file previously contained smart quotes (‘ ’) inside the embedded
# python3 snippets and en-dashes (–) instead of hyphens on the pytest flags,
# which meant it could not execute at all. Keep it ASCII-only. If you edit it
# in a word processor, check the quotes and dashes afterwards.
# =============================================================================

set -o pipefail

echo ""
echo "============================================================"
echo "  FiGPT_OutlookMCP — Full Test Suite"
echo "============================================================"

# ---------------------------------------------------------------------------
# Step 1: Check the server is up and list registered tools
# ---------------------------------------------------------------------------
echo ""
echo "[ STEP 1 ] Checking server health and listing tools..."
echo "------------------------------------------------------------"

HEALTH=$(curl -s --max-time 5 http://localhost:8000/health)

if [ -z "$HEALTH" ]; then
    echo "  WARNING: server not responding on localhost:8000"
    echo "  Start it with: python server.py"
    echo "  Skipping tool listing - running tests only."
else
    echo "$HEALTH" | python3 -c '
import sys, json
try:
    d = json.load(sys.stdin)
except json.JSONDecodeError:
    print("  ERROR: /health did not return valid JSON")
    sys.exit(0)

print("  Status      : %s" % d.get("status", "unknown"))
print("  Server      : %s" % d.get("server", "unknown"))
print("  Transport   : %s" % d.get("transport", "unknown"))
print("  Tools count : %s" % d.get("tools_registered", 0))
print("")
print("  Tools registered:")
for i, t in enumerate(d.get("tools", []), 1):
    print("    %2d. %s" % (i, t))
'
fi

# ---------------------------------------------------------------------------
# Step 2: Run the test suite
# ---------------------------------------------------------------------------
echo ""
echo "[ STEP 2 ] Running all tests..."
echo "------------------------------------------------------------"

python -m pytest tests/ -v --tb=short --no-header
TEST_EXIT=$?

echo ""
echo "============================================================"
if [ $TEST_EXIT -eq 0 ]; then
    echo "  All tests passed."
else
    echo "  TESTS FAILED (exit code $TEST_EXIT)"
fi
echo "============================================================"
echo ""

exit $TEST_EXIT
```

The runner now also **exits non-zero when tests fail** — the old version piped through `tail`, which swallowed the exit code and always reported success. That matters the moment this runs in CI.

---
---

# BLOCK F — Hygiene

## F1 — `server.py` dead scaffolding

**Find:**
```python
# FastMCP — the MCP server framework
from fastmcp import FastMCP

# from server import mcp
from tools.mcp_instance import mcp


# CORS middleware — required for LibreChat to connect from a different
# origin (different domain/port). Without this, LibreChat's browser-based
# connection will be blocked by the browser's cross-origin policy.
from starlette.responses import JSONResponse
from starlette.middleware.cors import CORSMiddleware
cors_app_wrapper = None  # placeholder — applied after mcp is initialised
```

**Replace with:**
```python
# The shared FastMCP instance. It is created in tools/mcp_instance.py,
# NOT here — every tools/ module imports it from there, so importing it
# from server.py would create a circular import.
from tools.mcp_instance import mcp

# JSONResponse is used by the /health route below.
#
# NOTE: CORS is handled by nginx, not here. A CORSMiddleware import and a
# `cors_app_wrapper = None` placeholder used to sit at this spot; neither
# was ever applied. FastMCP 3.x also rejects cors_origins on the
# constructor, so do not try to reintroduce it in Python.
from starlette.responses import JSONResponse
```

Also correct the module docstring:

**Find:**
```
        INFO | server | 9 tools registered and ready
```

**Replace with:**
```
        INFO | server | All tool modules imported and registered successfully
```

### F1.1 — Confirm which host/port actually binds

Not a patch — a check worth doing once.

`server.py` calls `mcp.run(transport=..., host=..., port=...)` while `.env` also sets `FASTMCP_HOST`/`FASTMCP_PORT`. If FastMCP 3.x ignores `host`/`port` on `run()` the way it rejects them on the constructor, then `MCP_HOST`/`MCP_PORT` are decorative and only the `FASTMCP_*` pair matters. Two variables that appear to control the same thing where only one does is a future outage.

Test: set `MCP_PORT=8001` in `.env`, leave `FASTMCP_PORT=8000`, restart, and see which port answers.

```bash
curl -s -o /dev/null -w "8000: %{http_code}\n" http://localhost:8000/health
curl -s -o /dev/null -w "8001: %{http_code}\n" http://localhost:8001/health
```

Whichever responds is the one that counts. Delete or comment the other pair and note it in `PROJECT_STATE.md`. **Set `MCP_PORT` back to 8000 afterwards.**

## F2 — `IndexError` on a bare "1." line

MOM text is LLM-generated, so a numbered line with nothing after it is reachable.

**File:** `tools/mom_tools.py` → `_convert_mom_to_html`

**Find:**
```python
        elif stripped[:2] in ("1.", "2.", "3.", "4.", "5.", "6.", "7.") and stripped[2] == " ":
```

**Replace with:**
```python
        # len > 2 guard first: a line that is exactly "1." matches the
        # prefix check and then raises IndexError on stripped[2].
        elif (
            len(stripped) > 2
            and stripped[:2] in ("1.", "2.", "3.", "4.", "5.", "6.", "7.")
            and stripped[2] == " "
        ):
```

## F3 — Audit log leaks task titles

`_sanitise_inputs` allowlists `title` while its own comment excludes `subject` as *"may be sensitive"* — but `add_task_todo(title=...)` passes user task text, frequently derived from that very subject.

**File:** `utils/audit_logger.py`

**Find:**
```python
    SAFE_FIELDS = {
        "count", "top", "keyword", "date_range", "date_from", "date_to",
        "chart_type", "title", "use_thread", "is_read", "flag_status",
        "source", "list_name", "days_threshold", "folder_name",
        "duration_minutes", "search_days", "lang_code", "chart_type",
    }
```

**Replace with:**
```python
    SAFE_FIELDS = {
        "count", "top", "keyword", "date_range", "date_from", "date_to",
        "chart_type", "use_thread", "is_read", "flag_status",
        "source", "list_name", "days_threshold", "folder_name",
        "duration_minutes", "search_days", "lang_code", "page",
        "reply_all",
    }

    # "title" was in SAFE_FIELDS, which contradicted the exclusion of
    # "subject" a few lines below: add_task_todo and add_task_planner
    # pass user-authored task text as `title`, usually derived from the
    # email subject that was deliberately excluded. Titles are now
    # length-only — enough to correlate an entry, no content.
    LENGTH_ONLY_FIELDS = {"title", "subject"}
```

**Find:**
```python
    safe = {}
    for key, value in inputs.items():
        if key in SAFE_FIELDS:
            safe[key] = value
        elif key in ID_FIELDS:
```

**Replace with:**
```python
    safe = {}
    for key, value in inputs.items():
        if key in SAFE_FIELDS:
            safe[key] = value
        elif key in LENGTH_ONLY_FIELDS:
            safe[f"{key}_length"] = len(str(value)) if value else 0
        elif key in ID_FIELDS:
```

> `tests/test_guardrails.py::TestAuditLogger::test_sanitise_inputs_removes_body_content` may assert on `title`. If it fails, update the assertion to expect `title_length`.

## F4 — Dead code removal

### F4.1 — Old Requesty block in `attachment_tools.py`

~45 lines of the pre-D-01 implementation parked in a module-level string after `summarise_attachment`. It references `generate_summary`, which is commented out of the imports.

**Find:**
```python
"""
    try:
        # Step 1: Reuse the same download + parse chain as read_attachment.
        file_bytes, file_name, content_type = await download_attachment(
```

Delete from that opening `"""` down to and including the closing `"""` that follows `return format_tool_error(exc, tool_name="summarise_attachment", logger=logger)` — the second, indented copy near the end of the file.

The file should end with the real function's `except` block:

```python
    except Exception as exc:
        return format_tool_error(
            exc, tool_name="summarise_attachment", logger=logger
        )
```

Also remove the stale import comment:

**Find:**
```python
# from llm.requesty_client import generate_summary
```

**Replace with:**
```python
# NOTE: no LLM import here by design (D-01). This tool returns extracted
# text plus an instruction; LibreChat's own model does the summarising.
```

### F4.2 — Duplicate import in `mom_tools.py`

**Find:**
```python
from utils.validator import validate_draft_content, append_validation_to_result
from utils.validator import (validate_mom_output, validate_draft_content, append_validation_to_result,)
```

**Replace with:**
```python
from utils.validator import (
    validate_mom_output,
    validate_draft_content,
    append_validation_to_result,
)
```

### F4.3 — matplotlib remnants in `chart_tools.py`

**Find:**
```python
Why this file exists:
    LibreChat's LLM can produce SVG code for charts, but cannot
    generate actual downloadable image files. This tool fills that
    gap — it takes structured data, builds a real chart using
    matplotlib, saves it as a PNG file, and returns both the file
    path and a base64-encoded version for inline display.
```

**Replace with:**
```python
Why this file exists:
    LibreChat renders Mermaid diagrams natively in chat. This tool takes
    structured data and emits Mermaid chart syntax, which LibreChat
    displays as a real chart with no image rendering involved.

    It used to build PNGs with matplotlib. That was replaced on
    24 July 2026 — no image files, no base64, no matplotlib dependency.
```

**Find:**
```python
Design notes:
    - No external API calls — matplotlib runs entirely locally.
    - Charts saved to temp_attachments/charts/ as PNG files.
    - Base64 encoding allows LibreChat to display inline.
    - matplotlib uses 'Agg' backend (no display/GUI required)
      which is correct for server-side rendering on Linux.
    - File names include timestamp to avoid collisions.
```

**Replace with:**
```python
Design notes:
    - No external API calls and no rendering library — the output is
      plain text in Mermaid syntax.
    - Charts saved to temp_attachments/charts/ as .mmd files.
    - Bar and line use 'xychart-beta'; pie uses Mermaid's 'pie' block.
    - File names include a timestamp and a short uuid to avoid collisions.
```

**Find:**
```python
import os
import uuid
import base64
from pathlib import Path
from datetime import datetime
```

**Replace with:**
```python
import uuid
from pathlib import Path
from datetime import datetime
```

**Find:**
```python
# comp. brand-aligned colour palette
# Professional blues, greys and accent colours
CHART_COLOURS = [
    "#0078D4",  # Microsoft Blue — primary
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
```

**Replace with:** *(nothing — delete. Mermaid uses LibreChat's own theme; this palette has been unused since the rewrite)*

### F4.4 — Dead `is_draft` check in `followup_tools.py`

`_message_to_dict` never sets an `is_draft` key, so this always evaluates false. Harmless — `fetch_recent_messages` filters `isDraft eq false` server-side — but misleading.

**Find:**
```python
    # Skip drafts and emails sent by the user themselves
    if msg.get("is_draft"):
        return False

    sender_email = (
```

**Replace with:**
```python
    # Drafts are already excluded server-side by fetch_recent_messages
    # (filter="isDraft eq false"), and _message_to_dict does not carry an
    # is_draft key — so there is nothing to check here. Skip emails the
    # user sent themselves.
    sender_email = (
```

## F5 — `generate_chart` unused parameter

`x_label` is accepted and documented but never emitted; `xychart-beta` only gets `y-axis`.

**File:** `tools/chart_tools.py`

**Find:**
```python
        x_label (str): X-axis label (bar and line only).
        y_label (str): Y-axis label (bar and line only).
```

**Replace with:**
```python
        x_label (str): Accepted for compatibility but NOT rendered —
                        Mermaid's xychart-beta has no x-axis title.
                        Fold the label into `title` instead.
        y_label (str): Y-axis label (bar and line only).
```

Leaving the parameter in place keeps the signature stable for LibreChat; the docstring now stops the model expecting it to appear.

## F6 — Deprecated `datetime.utcnow()`

**File:** `graph/availability_client.py`

**Find:**
```python
from datetime import datetime, timedelta
```

**Replace with:**
```python
from datetime import datetime, timedelta, timezone
```

**Find:**
```python
    if preferred_start:
        start_dt = parse_date_string(preferred_start)
        if not start_dt:
            start_dt = datetime.utcnow()
    else:
        start_dt = datetime.utcnow()
```

**Replace with:**
```python
    # datetime.utcnow() is deprecated in Python 3.12. .replace(tzinfo=None)
    # keeps the naive value the rest of this function expects — the times
    # are later stamped with "UTC" explicitly via DateTimeTimeZone.
    if preferred_start:
        start_dt = parse_date_string(preferred_start)
        if not start_dt:
            start_dt = datetime.now(timezone.utc).replace(tzinfo=None)
    else:
        start_dt = datetime.now(timezone.utc).replace(tzinfo=None)
```

**Find:**
```python
        parsed = parse_date_string(start_datetime)
        start_dt = parsed if parsed else datetime.utcnow() + timedelta(days=1)
```

**Replace with:**
```python
        parsed = parse_date_string(start_datetime)
        start_dt = parsed if parsed else (
            datetime.now(timezone.utc).replace(tzinfo=None) + timedelta(days=1)
        )
```

## F7 — `list_tasks` docstring overpromises

Planner listing returns an error (blocked on B-01), but the docstring advertises it, so the model keeps trying.

**File:** `tools/followup_tools.py` → `list_tasks`

**Find:**
```python
    Use this tool when the user asks things like:
    - "Show my To-Do tasks"
    - "What tasks do I have?"
    - "List my Planner tasks"
    - "What's on my task list?"

    Args:
        source (str): "todo" for Microsoft To-Do,
                       "planner" for Microsoft Planner.
                       Default "todo".
```

**Replace with:**
```python
    Use this tool when the user asks things like:
    - "Show my To-Do tasks"
    - "What tasks do I have?"
    - "What's on my task list?"

    NOTE: Only Microsoft To-Do is supported. Planner listing is not
    implemented — it needs Tasks.ReadWrite and Group.Read.All admin
    consent, which has not been granted. Passing source="planner"
    returns an explanatory error rather than results.

    Args:
        source (str): Currently only "todo" is supported.
                       Default "todo".
```

---
---

# Verification

## After each block

```bash
cd /opt/FiGPT_OutlookMCP
source .venv/bin/activate

# Syntax check every file you touched
python -c "import ast; ast.parse(open('tools/draft_tools.py').read()); print('OK')"

# Full suite
python -m pytest tests/ -v

# Restart, capturing logs
python server.py 2>&1 | tee /tmp/mcp.log
```

Expect `All tool modules imported and registered successfully`. An `ImportError` names the offending file.

## Health check

```bash
curl -s http://localhost:8000/health | python3 -m json.tool
```

**Expect `"tools_registered": 36`** — 35 before, minus `create_draft_invite`, plus `compose_invite` and `save_invite`.

## Behaviour tests in LibreChat

Reconnect the MCP server first.

| # | Prompt | Pass condition | Covers |
|---|---|---|---|
| 1 | `Search my emails for "report"` | Table has a 🔵 column; no broken rows | C3 |
| 2 | `Which of my search results are unread?` | A filtered count, no refusal | C3 |
| 3 | `Find emails about "report" between 1 July and 26 July` | Results, not an error | C8 |
| 4 | `Show me my drafts` | Clean table, no crash on an empty draft | C7 |
| 5 | `Which emails need a follow-up?` | Emails you have already answered are **absent** | D1 |
| 6 | `Compose a follow-up for <an unanswered email>` → yes | Draft is **threaded** under the original | D2 |
| 7 | `Generate MOM from this email thread` | No stray `f"` or `\n\n` in the output | B1 |
| 8 | `Extract tasks from my recent emails` | Governance rules appear once, not twice | B2 |
| 9 | `Draft a meeting invite with <internal colleague> tomorrow 10am` | Preview **and a warning that saving sends immediately**. Nothing sent yet | A3 |
| 10 | *(then)* `Yes, send it` | Event created, colleague receives the invitation, confirmation says **sent** | A3 |
| 11 | `Save an email draft to <yourself>` | Confirmation says "Draft Saved", **not** "Meeting invite created" | A4 |

> **Test 9 is the important one.** If the model calls `save_invite` without showing a preview first, the approval gate isn't working — tell me and I'll strengthen the docstring wording. Use an internal colleague; external addresses have no free/busy data (B-03).

## Report back

1. `curl http://localhost:8000/health` — the `tools_registered` count.
2. The pytest summary line.
3. Results of behaviour tests 5, 9 and 10.
4. Output of the two `git` commands in E2.

---

# Deliberately not included

| Item | Why |
|---|---|
| **Rate limiting** (§5b) | ~20 lines of FastMCP middleware, but it needs a decision on limits and on whether the cap is per-token or per-server. Own conversation |
| **Scope validation** (§5b) | Requires decoding the JWT to read its `scp` claim. Straightforward, but it changes the auth path — the one part of the system that is currently working reliably. Not while we're changing eleven other files |
| **`temp_attachments/` cleanup** (§5b) | Needs a decision: delete immediately after parsing, or age out on a timer? Affects whether a user can re-read an attachment in the same conversation |
| **Semantic search caching** (F-10) | Re-embeds up to 200 emails per call. A real fix needs an id→embedding cache with an eviction policy, not a patch |
| **HITL gates on `move_email` / `add_task_planner`** (§5b) | Instruction-only changes, easy — but `move_email` to `deleteditems` is destructive and deserves the same care as the invite gate rather than being tacked on here |
| **Planner tools** (B-01) | Externally blocked on Azure admin consent |
| **systemd unit** (B-04) | Server team's task. Pairs naturally with E1 |



---


State file updated. Here's the verdict.

34 of 36 passed — and the two Highs are dead

Round 7 held at every step. Preview rendered, irreversibility warned, refusal honoured with nothing created, then a real invite sent and received by a real colleague. That closes L-02 and F-17 together — which is exactly why they had to ship together. The F-17 trap (a tool that emails external parties while claiming to save a draft) never got a chance to fire, because the D-15 split landed with the L-02 fix rather than after it.

B-02 resolved itself. The plan said a clean 403 would confirm the missing grant. Instead POST /me/events just worked — so Calendars.ReadWrite is already granted. No Azure escalation. That blocker was hypothetical all along.

Thirteen findings close on their tests: F-04, F-05, F-08, F-11, F-14, F-15, F-16, F-18, F-23, F-25, F-17, plus F-03 and half of F-10. The markdown-table, read-status, safe-key and governance-duplication families are now behaviour-verified, not just syntax-verified — which was the entire point of running this.

Two of your results aren't failures:

- T7.3, no self-invite — expected Graph behaviour. The organiser is never sent an invitation; the event is written straight to your calendar. Worth confirming it appeared there, but nothing to fix.
- T5.4 — the "Planner isn't available" message is correct and honest. That's B-01 behaving properly.

What actually needs work

L-03 (T3.4) — don't patch this yet. Your own observation undermines the diagnosis: if sending the draft manually nests it correctly under the original conversation, the conversationId is right, which is not what a standalone save_draft_to_outlook email looks like. Outlook lists reply drafts flat in the Drafts folder regardless of threading. And T4.4 — the same threading path via compose_followup — passed cleanly. So the code is probably correct and the Drafts view is lying to you. One grep settles it; patching first risks "fixing" working code.

F-27 (new, Med) — you asked for Planner, got To-Do. The heading said "Microsoft To-Do" so it wasn't a lie, but it answered a Planner question with a different data source without saying it was substituting. Instruction-level fix.

F-28 (new, Med) — list_emails_paged returns Sent Items mixed into Inbox; list_emails doesn't. The two tools disagree about what "my emails" means. Almost certainly /me/messages instead of /me/mailFolders/inbox/messages.

F-13 / T8.2 — you reported it with T8.1's wording ("a Mermaid pie chart") for a bar chart request, so I can't tell what happened. Leaving it open rather than guessing.

⚠️ The one urgent thing — F-07, escalated to High

Your log listing exposed something worse than the ledger recorded. .gitignore line 41 is logs/audit.log — a single exact path. It does not match:

- logs/app.log (63 KB) — brand new, created by F-11's own fix, full of tracebacks
- logs/app.log.2026-07-27
- eight rotated logs/audit.log.2026-07-2X files, up to 44 KB, carrying real user email addresses

F-11's fix quietly widened F-11's blast radius. This is the only open finding that gets worse with time — every push bakes it further into Azure DevOps history, and it's a GDPR item for a German entity.

Step 1 — run these two read-only checks first

Order matters here: if the files are already tracked, widening .gitignore accomplishes nothing on its own.

cd /opt/FiGPT_OutlookMCP
git ls-files logs/ bin/
git status --short

Step 2 — the .gitignore patch (safe in every case)

Find (line 41):
logs/
(your grep showed the rule resolving to logs/audit.log — paste the actual line back if it differs)

Replace with:
# Runtime logs — contain real user email addresses and tracebacks.
# Must match rotated files too: app.log.2026-07-27, audit.log.2026-07-26, ...
logs/*.log
logs/*.log.*
!logs/.gitkeep

The !logs/.gitkeep line keeps the directory itself in the repo so the app doesn't fail on a fresh clone. If git ls-files returned anything, tell me and I'll give you the git rm --cached commands — that has to be done carefully, since a wrong flag deletes the live logs off the server.

Three questions, then I write FIX-04

1. L-03, the decisive one. What does the audit log say fired for T3.4?
grep -E "save_reply_draft|save_draft_to_outlook" logs/audit.log | tail -5
2. T8.2 — was it actually a bar chart, and did the x-axis label appear on it?
3. Check #36 — was that grep against .env.example or the live .env? It matters: in .env.example it's F-20, a landmine for the next environment. In the live .env it means the server is bound to loopback and something else is holding the proxy together.

✻ Cogitated for 4m 41s
