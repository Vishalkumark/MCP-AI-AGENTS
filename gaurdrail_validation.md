# Output Validation & Hallucination Prevention

**Project:** outlook-ai-agent-mcp
**Scope:** Guardrails, source grounding, output validation, human-in-loop
**Status:** Implement in order — each section builds on the previous

-----

## PART A — Understanding the Problem

### What is hallucination in this project?

Hallucination means the LLM generates content that looks correct
but was never in the source data. In our context this means:

|Scenario           |Example hallucination risk                         |
|-------------------|---------------------------------------------------|
|MOM generation     |LLM invents a decision nobody made                 |
|Task extraction    |LLM adds a deadline not mentioned in the email     |
|Email summarisation|LLM merges two unrelated emails into one summary   |
|Follow-up drafts   |LLM adds promises the user never intended          |
|Availability check |LLM guesses a time slot instead of reading calendar|

### Why our architecture already reduces this

Our design already has a strong natural guardrail:

- Tools return **raw data** from Graph API
- LLM only receives **what the tool gives it**
- LLM cannot query the internet or external sources
- The `instruction` field tells the LLM exactly what to do with the data

The gaps are in **what the LLM does with that data** — it can still
infer, extrapolate, or confabulate within the context it receives.

### Three layers of defence we will add

```
Layer 1 — INPUT VALIDATION
    Validate everything before it reaches Graph API.
    Reject bad inputs early with clear messages.

Layer 2 — SOURCE GROUNDING
    Every tool response includes source references.
    LLM is instructed to cite sources and flag uncertainty.

Layer 3 — OUTPUT VALIDATION
    Check tool outputs before returning to LibreChat.
    Detect and flag empty, suspiciously short, or mismatched results.
```

-----

## PART B — New Files to Create

```
utils/
├── governance.py        ← Centralised rules injected into every tool
├── validator.py         ← Output validation functions
└── audit_logger.py      ← Structured audit trail of every tool call
```

-----

## PART C — `utils/governance.py`

```python
"""
governance.py
=============
Centralised AI governance rules injected into every MCP tool's
instruction field.

Why this file exists:
    Without a single source of truth for AI behaviour rules,
    each tool would need its own copy of "don't invent facts" or
    "always cite sources" — leading to inconsistency and drift.
    This file defines all rules once. Every tool imports from here.

Design notes:
    - Rules are plain strings appended to every tool's instruction.
    - Specialised rule sets exist per tool category (email, MOM,
      draft, task) — each inherits the base rules plus its own.
    - Rules are deliberately short and directive — LLMs follow
      short, clear constraints better than long essays.
"""

# ---------------------------------------------------------------------------
# Base rules — applied to EVERY tool output without exception
# ---------------------------------------------------------------------------
BASE_RULES = """
STRICT OUTPUT RULES — FOLLOW EXACTLY:
1. Only state facts that appear in the data provided to you.
2. Never invent names, email addresses, dates, or decisions.
3. If information is missing or unclear, say so explicitly.
4. Never combine information from different emails unless instructed.
5. If you are uncertain about any fact, prefix it with [UNCERTAIN].
6. Do not add promises, commitments, or deadlines not in the source data.
"""

# ---------------------------------------------------------------------------
# Email summarisation rules
# ---------------------------------------------------------------------------
EMAIL_SUMMARY_RULES = BASE_RULES + """
EMAIL SUMMARY SPECIFIC RULES:
- Summarise only what is in the email body provided.
- Do not reference other emails or prior conversation context.
- If the email body is empty or unclear, say "Email body unavailable".
- Never guess the sender's intent — only describe what they wrote.
"""

# ---------------------------------------------------------------------------
# MOM generation rules
# ---------------------------------------------------------------------------
MOM_RULES = BASE_RULES + """
MOM SPECIFIC RULES:
- List only decisions explicitly stated in the emails.
- List only action items explicitly assigned in the emails.
- If no decisions are found, write exactly: "No explicit decisions recorded."
- If no action items are found, write exactly: "No action items recorded."
- Never add agenda items that do not appear in the email content.
- Every attendee listed must appear as a sender or recipient in the emails.
- Dates must be copied verbatim from the email — never calculated or assumed.
"""

# ---------------------------------------------------------------------------
# Draft composition rules
# ---------------------------------------------------------------------------
DRAFT_RULES = BASE_RULES + """
DRAFT EMAIL SPECIFIC RULES:
- Write only what the context instructs — nothing more.
- Never add promises, timelines, or commitments not stated in the context.
- Do not reference information from other emails in this conversation.
- If the context is ambiguous, draft a neutral, open response.
- Always use professional language — never casual, never aggressive.
- Closing must always be: Best regards,
"""

# ---------------------------------------------------------------------------
# Task extraction rules
# ---------------------------------------------------------------------------
TASK_RULES = BASE_RULES + """
TASK EXTRACTION SPECIFIC RULES:
- Only extract tasks that contain a clear action verb (e.g. send, review, prepare).
- Only include a deadline if one is explicitly stated — never infer one.
- Only assign an owner if a specific person is named in relation to the task.
- If owner is unclear, write "Unassigned" — never guess.
- If deadline is unclear, write "Not specified" — never guess.
"""

# ---------------------------------------------------------------------------
# Follow-up rules
# ---------------------------------------------------------------------------
FOLLOWUP_RULES = BASE_RULES + """
FOLLOW-UP SPECIFIC RULES:
- Base the follow-up on the original email content provided.
- Never reference conversations or context not in the provided email.
- Keep the follow-up to 2-3 sentences maximum.
- Never demand or pressure — always polite and professional.
- Do not add new topics not raised in the original email.
"""

# ---------------------------------------------------------------------------
# Source citation instruction — appended to all data-heavy tools
# ---------------------------------------------------------------------------
SOURCE_CITATION = """
SOURCE CITATION:
After your response, add a brief "Sources:" section listing:
- Email subject(s) used
- Sender(s) referenced
- Date(s) of emails used
This helps the user verify your output against the original emails.
"""

# ---------------------------------------------------------------------------
# Uncertainty flag instruction
# ---------------------------------------------------------------------------
UNCERTAINTY_INSTRUCTION = """
UNCERTAINTY HANDLING:
If any part of your response is inferred rather than explicitly stated
in the source data, prefix that sentence with [INFERRED].
If you cannot determine a fact from the data, write [UNAVAILABLE].
Never omit these flags to make the response look cleaner.
"""


# ---------------------------------------------------------------------------
# Helper functions
# ---------------------------------------------------------------------------
def get_email_rules(include_sources: bool = True) -> str:
    """
    Get governance rules for email summarisation tools.

    Args:
        include_sources (bool): Whether to append source citation rules.

    Returns:
        str: Combined rules string to append to tool instruction.
    """
    rules = EMAIL_SUMMARY_RULES + UNCERTAINTY_INSTRUCTION
    if include_sources:
        rules += SOURCE_CITATION
    return rules


def get_mom_rules(include_sources: bool = True) -> str:
    """
    Get governance rules for MOM generation tools.

    Args:
        include_sources (bool): Whether to append source citation rules.

    Returns:
        str: Combined rules string to append to tool instruction.
    """
    rules = MOM_RULES + UNCERTAINTY_INSTRUCTION
    if include_sources:
        rules += SOURCE_CITATION
    return rules


def get_draft_rules() -> str:
    """
    Get governance rules for all draft composition tools.

    Returns:
        str: Combined rules string to append to tool instruction.
    """
    return DRAFT_RULES + UNCERTAINTY_INSTRUCTION


def get_task_rules(include_sources: bool = True) -> str:
    """
    Get governance rules for task extraction tools.

    Args:
        include_sources (bool): Whether to append source citation rules.

    Returns:
        str: Combined rules string to append to tool instruction.
    """
    rules = TASK_RULES + UNCERTAINTY_INSTRUCTION
    if include_sources:
        rules += SOURCE_CITATION
    return rules


def get_followup_rules() -> str:
    """
    Get governance rules for follow-up composition tools.

    Returns:
        str: Combined rules string.
    """
    return FOLLOWUP_RULES + UNCERTAINTY_INSTRUCTION
```

-----

## PART D — `utils/validator.py`

```python
"""
validator.py
============
Output validation functions for MCP tool responses.

Why this file exists:
    Tools return data from Microsoft Graph API — but the data can be:
        - Empty (no emails found)
        - Suspiciously short (possible API error or empty response)
        - Structurally wrong (missing required fields)
        - Containing known hallucination patterns

    This file validates tool outputs BEFORE they are returned to
    LibreChat, catching problems at the tool layer rather than
    letting bad data reach the LLM.

Design notes:
    - Validators are lightweight — no LLM calls, no external requests.
    - Each validator returns a ValidationResult with:
        - is_valid (bool)
        - warnings (list of strings shown to the user)
        - errors (list of strings — blocks response if critical)
    - Warnings are surfaced to the user as [VALIDATION WARNING] notes.
    - Errors replace the tool response with a clear failure message.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from dataclasses import dataclass, field
from typing import Any

from utils.logger import get_logger

logger = get_logger(__name__)


# ---------------------------------------------------------------------------
# ValidationResult dataclass
# ---------------------------------------------------------------------------
@dataclass
class ValidationResult:
    """
    Result of a tool output validation check.

    Attributes:
        is_valid (bool): True if output passed all checks.
        warnings (list): Non-critical issues to surface to user.
        errors (list): Critical issues that block the response.
    """
    is_valid: bool = True
    warnings: list = field(default_factory=list)
    errors: list = field(default_factory=list)

    def add_warning(self, message: str) -> None:
        self.warnings.append(message)
        logger.warning(f"Validation warning: {message}")

    def add_error(self, message: str) -> None:
        self.errors.append(message)
        self.is_valid = False
        logger.error(f"Validation error: {message}")

    def format_warnings(self) -> str:
        """Format warnings as a markdown notice block."""
        if not self.warnings:
            return ""
        lines = ["⚠️ **Validation Notices:**"]
        for w in self.warnings:
            lines.append(f"- {w}")
        return "\n".join(lines)

    def format_errors(self) -> str:
        """Format errors as a markdown error block."""
        if not self.errors:
            return ""
        lines = ["❌ **Validation Errors:**"]
        for e in self.errors:
            lines.append(f"- {e}")
        return "\n".join(lines)


# ---------------------------------------------------------------------------
# Function: validate_email_list
# ---------------------------------------------------------------------------
def validate_email_list(emails: list[dict]) -> ValidationResult:
    """
    Validate a list of email dictionaries returned by Graph API.

    Checks:
        - List is not empty
        - Each email has required fields (id, subject, sender)
        - No duplicate email IDs (would indicate API pagination bug)
        - Dates are not in the future (would indicate data corruption)

    Args:
        emails (list[dict]): List of email dicts from mail_client.

    Returns:
        ValidationResult: Validation outcome with warnings/errors.
    """
    result = ValidationResult()

    if not emails:
        result.add_warning(
            "No emails were returned. Your inbox may be empty or "
            "the filter returned no matches."
        )
        return result

    seen_ids = set()
    for i, email in enumerate(emails):
        # Check required fields
        if not email.get("id"):
            result.add_warning(
                f"Email at position {i+1} is missing an ID — "
                f"it may not be openable."
            )

        if not email.get("subject"):
            result.add_warning(
                f"Email {i+1} has no subject line."
            )

        # Check for duplicate IDs
        email_id = email.get("id", "")
        if email_id in seen_ids:
            result.add_warning(
                f"Duplicate email ID detected at position {i+1}. "
                f"This may indicate a Graph API pagination issue."
            )
        seen_ids.add(email_id)

    return result


# ---------------------------------------------------------------------------
# Function: validate_email_body
# ---------------------------------------------------------------------------
def validate_email_body(body_text: str, email_subject: str = "") -> ValidationResult:
    """
    Validate that an email body is readable and substantial enough
    to summarise meaningfully.

    Checks:
        - Body is not empty
        - Body is not just whitespace or HTML tags
        - Body is long enough for meaningful summarisation (>50 chars)

    Args:
        body_text (str): The email body text to validate.
        email_subject (str): Subject line for error context.

    Returns:
        ValidationResult: Validation outcome.
    """
    result = ValidationResult()

    if not body_text or not body_text.strip():
        result.add_warning(
            f"Email body is empty"
            + (f" for '{email_subject}'" if email_subject else "")
            + ". Cannot summarise."
        )
        return result

    # Strip HTML and check readable length
    import re
    clean_text = re.sub(r"<[^>]+>", "", body_text).strip()

    if len(clean_text) < 50:
        result.add_warning(
            f"Email body is very short ({len(clean_text)} characters). "
            f"Summary may not be meaningful."
        )

    return result


# ---------------------------------------------------------------------------
# Function: validate_mom_output
# ---------------------------------------------------------------------------
def validate_mom_output(
    mom_text: str,
    expected_sections: list[str] = None,
) -> ValidationResult:
    """
    Validate that a generated MOM contains all required sections
    and doesn't show obvious signs of hallucination.

    Checks:
        - All required sections are present
        - Action items section is not empty
        - No [INFERRED] flags exceed a threshold (too much guessing)
        - MOM is not suspiciously short

    Args:
        mom_text (str): The generated MOM text to validate.
        expected_sections (list[str]): Section headers to check for.
                                        Defaults to standard formal MOM.

    Returns:
        ValidationResult: Validation outcome.
    """
    result = ValidationResult()

    if not mom_text or len(mom_text.strip()) < 100:
        result.add_error(
            "Generated MOM is too short to be valid. "
            "Please try again or check the source email content."
        )
        return result

    # Check all required sections are present
    if expected_sections is None:
        expected_sections = [
            "MEETING DETAILS",
            "ATTENDEES",
            "DISCUSSION SUMMARY",
            "DECISIONS MADE",
            "ACTION ITEMS",
            "NEXT STEPS",
        ]

    for section in expected_sections:
        if section not in mom_text.upper():
            result.add_warning(
                f"MOM is missing the '{section}' section. "
                f"The output may be incomplete."
            )

    # Check for excessive uncertainty flags
    inferred_count = mom_text.upper().count("[INFERRED]")
    unavailable_count = mom_text.upper().count("[UNAVAILABLE]")

    if inferred_count > 5:
        result.add_warning(
            f"MOM contains {inferred_count} [INFERRED] flags — "
            f"many facts were not explicitly stated in the source emails. "
            f"Please review carefully before distributing."
        )

    if unavailable_count > 3:
        result.add_warning(
            f"MOM contains {unavailable_count} [UNAVAILABLE] fields — "
            f"the source emails may not contain enough meeting information."
        )

    return result


# ---------------------------------------------------------------------------
# Function: validate_task_list
# ---------------------------------------------------------------------------
def validate_task_list(tasks: list[dict]) -> ValidationResult:
    """
    Validate a list of extracted tasks for completeness and quality.

    Checks:
        - At least one task was found
        - Each task has a title/action
        - Suspicious patterns: tasks with no action verb
        - Tasks with identical titles (duplicate extraction)

    Args:
        tasks (list[dict]): List of task dictionaries.

    Returns:
        ValidationResult: Validation outcome.
    """
    result = ValidationResult()

    if not tasks:
        result.add_warning(
            "No tasks were extracted from the emails. "
            "The emails may not contain any clear action items."
        )
        return result

    # Check for action verbs in task titles
    action_verbs = [
        "send", "review", "prepare", "update", "complete", "submit",
        "check", "confirm", "schedule", "arrange", "follow", "create",
        "write", "draft", "share", "provide", "organise", "inform",
        "contact", "reply", "respond", "approve", "coordinate",
    ]

    seen_titles = set()
    for task in tasks:
        title = (task.get("title") or task.get("action") or "").lower()

        # Check for duplicates
        if title in seen_titles:
            result.add_warning(
                f"Duplicate task detected: '{title}'. "
                f"This may be an extraction error."
            )
        seen_titles.add(title)

        # Check for action verb presence
        has_action_verb = any(verb in title for verb in action_verbs)
        if not has_action_verb and len(title) > 5:
            result.add_warning(
                f"Task '{title[:50]}' may not be an actionable item — "
                f"no clear action verb found."
            )

    return result


# ---------------------------------------------------------------------------
# Function: validate_draft_content
# ---------------------------------------------------------------------------
def validate_draft_content(
    body_text: str,
    to_emails: list[str],
    subject: str,
) -> ValidationResult:
    """
    Validate email draft content before saving to Outlook Drafts.

    Checks:
        - Body is not empty
        - Recipients are valid email format
        - Subject is not empty
        - Body doesn't contain obvious placeholder text
        - Body is not identical to the context (raw context pasted in)

    Args:
        body_text (str): Draft email body text.
        to_emails (list[str]): List of recipient email addresses.
        subject (str): Email subject line.

    Returns:
        ValidationResult: Validation outcome.
    """
    result = ValidationResult()
    import re

    # Check subject
    if not subject or not subject.strip():
        result.add_error("Email subject is empty.")

    # Check recipients format
    email_pattern = re.compile(r"^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$")
    for addr in to_emails:
        if not email_pattern.match(addr.strip()):
            result.add_error(
                f"'{addr}' does not appear to be a valid email address."
            )

    # Check body
    if not body_text or not body_text.strip():
        result.add_error("Email body is empty — cannot save an empty draft.")
        return result

    # Check for obvious placeholder text
    placeholder_patterns = [
        "[your name]", "[name]", "[date]", "[insert", "lorem ipsum",
        "placeholder", "todo:", "fixme:", "[context provided]",
        "context to base", "compose ", "professional email body",
    ]
    body_lower = body_text.lower()
    for pattern in placeholder_patterns:
        if pattern in body_lower:
            result.add_warning(
                f"Draft body may contain placeholder text: '{pattern}'. "
                f"Please review before sending from Outlook."
            )

    # Check minimum length
    clean_body = re.sub(r"<[^>]+>", "", body_text).strip()
    if len(clean_body) < 30:
        result.add_warning(
            "Draft body is very short. Please ensure the email is complete."
        )

    return result


# ---------------------------------------------------------------------------
# Function: validate_calendar_events
# ---------------------------------------------------------------------------
def validate_calendar_events(events: list[dict]) -> ValidationResult:
    """
    Validate calendar events returned from Graph API.

    Checks:
        - Events list is not empty
        - Each event has subject and start time
        - Events are in chronological order

    Args:
        events (list[dict]): List of calendar event dicts.

    Returns:
        ValidationResult: Validation outcome.
    """
    result = ValidationResult()

    if not events:
        result.add_warning(
            "No calendar events found for the requested time range."
        )
        return result

    for i, event in enumerate(events):
        if not event.get("subject"):
            result.add_warning(
                f"Calendar event {i+1} has no subject/title."
            )
        if not event.get("start_time"):
            result.add_warning(
                f"Calendar event {i+1} is missing a start time."
            )

    return result


# ---------------------------------------------------------------------------
# Function: append_validation_to_result
# ---------------------------------------------------------------------------
def append_validation_to_result(
    tool_result: dict,
    validation: ValidationResult,
) -> dict:
    """
    Append validation warnings/errors to a tool result dictionary.
    Warnings are added as a notice field.
    Errors replace the result with a failure message.

    This is the single integration point — call this at the end of
    every tool function before returning.

    Args:
        tool_result (dict): The tool's result dictionary.
        validation (ValidationResult): The validation outcome.

    Returns:
        dict: Updated tool result with validation notices appended.
    """
    if not validation.is_valid:
        # Critical error — override the result entirely
        return {
            "error": True,
            "message": validation.format_errors(),
            "warnings": validation.warnings,
        }

    if validation.warnings:
        # Non-critical warnings — append to existing result
        tool_result["validation_notices"] = validation.format_warnings()
        tool_result["instruction"] = (
            tool_result.get("instruction", "")
            + f"\n\n{validation.format_warnings()}"
        )

    return tool_result
```

-----

## PART E — `utils/audit_logger.py`

```python
"""
audit_logger.py
===============
Structured audit trail for every MCP tool call.

Why this file exists:
    In an enterprise environment, knowing WHO called WHICH tool
    WHEN and with WHAT inputs is critical for:
        - Security review (was sensitive data accessed?)
        - Debugging (what exactly happened before an error?)
        - Compliance (audit trail for email access)
        - Performance monitoring (which tools are used most?)

    This file writes a separate audit log alongside the standard
    application log. Each entry is one JSON line — easy to parse,
    easy to grep, easy to feed into a SIEM tool later.

Design notes:
    - Audit log is separate from the main application log.
    - Each entry is a single JSON line (JSON-Lines format).
    - Sensitive values (email bodies, tokens) are NEVER logged.
    - Only metadata is logged: who, what, when, which email IDs.
    - Log file rotates daily to prevent unbounded growth.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import json
import logging
import os
from datetime import datetime, timezone
from pathlib import Path
from logging.handlers import TimedRotatingFileHandler

from utils.logger import get_logger

logger = get_logger(__name__)

# ---------------------------------------------------------------------------
# Audit log setup
# ---------------------------------------------------------------------------
AUDIT_LOG_DIR = Path("./logs")
AUDIT_LOG_FILE = AUDIT_LOG_DIR / "audit.log"


def _get_audit_logger() -> logging.Logger:
    """
    Get or create the dedicated audit logger.
    Writes to logs/audit.log with daily rotation.
    """
    audit_logger = logging.getLogger("audit")

    if audit_logger.handlers:
        return audit_logger

    # Create logs directory
    AUDIT_LOG_DIR.mkdir(parents=True, exist_ok=True)

    # Rotating file handler — keeps 30 days of audit logs
    handler = TimedRotatingFileHandler(
        AUDIT_LOG_FILE,
        when="midnight",
        interval=1,
        backupCount=30,
        encoding="utf-8",
    )
    handler.setFormatter(logging.Formatter("%(message)s"))
    audit_logger.addHandler(handler)
    audit_logger.setLevel(logging.INFO)
    audit_logger.propagate = False  # Don't duplicate to main log

    return audit_logger


# ---------------------------------------------------------------------------
# Function: log_tool_call
# ---------------------------------------------------------------------------
def log_tool_call(
    tool_name: str,
    user_email: str,
    inputs: dict,
    outcome: str = "success",
    error_message: str = "",
    email_ids_accessed: list = None,
) -> None:
    """
    Write a single audit log entry for an MCP tool call.

    Call this at the start of every tool function, after inputs
    are validated, before any Graph API calls are made.

    Args:
        tool_name (str): Name of the MCP tool being called.
        user_email (str): Email of the authenticated user.
                           Read from X-Fichtner-Email header.
        inputs (dict): SAFE input parameters — do NOT include
                        email body content, tokens, or secrets.
                        Only IDs, counts, keywords, types.
        outcome (str): "success", "error", or "validation_failed".
        error_message (str): Error detail if outcome != "success".
        email_ids_accessed (list): List of email IDs that were
                                    read during this tool call.
    """
    audit_entry = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "tool": tool_name,
        "user": user_email or "unknown",
        "inputs": _sanitise_inputs(inputs),
        "outcome": outcome,
        "email_ids_accessed": email_ids_accessed or [],
        "error": error_message or None,
    }

    try:
        audit_logger = _get_audit_logger()
        audit_logger.info(json.dumps(audit_entry))
    except Exception as log_err:
        # Audit logging must never crash the tool
        logger.warning(f"Audit log write failed: {log_err}")


# ---------------------------------------------------------------------------
# Function: get_user_email_from_headers
# ---------------------------------------------------------------------------
def get_user_email_from_headers() -> str:
    """
    Extract the authenticated user's email from the LibreChat
    request headers for audit logging purposes.

    Returns:
        str: User email address or "unknown" if not present.
    """
    try:
        from fastmcp.server.dependencies import get_http_headers
        headers = get_http_headers()
        return (
            headers.get("X-Fichtner-Email")
            or headers.get("x-fichtner-email")
            or "unknown"
        )
    except Exception:
        return "unknown"


# ---------------------------------------------------------------------------
# Helper: sanitise inputs for safe logging
# ---------------------------------------------------------------------------
def _sanitise_inputs(inputs: dict) -> dict:
    """
    Remove sensitive values from inputs before logging.
    Keeps only safe metadata fields.

    Args:
        inputs (dict): Raw input parameters from tool call.

    Returns:
        dict: Sanitised copy safe to write to audit log.
    """
    # Fields that are safe to log
    SAFE_FIELDS = {
        "count", "top", "keyword", "date_range", "date_from", "date_to",
        "chart_type", "title", "use_thread", "is_read", "flag_status",
        "source", "list_name", "days_threshold", "folder_name",
        "duration_minutes", "search_days", "lang_code", "chart_type",
    }

    # Fields that must be truncated (IDs are OK, content is not)
    ID_FIELDS = {
        "email_id", "attachment_id", "message_id", "folder_id",
        "event_id", "task_id",
    }

    safe = {}
    for key, value in inputs.items():
        if key in SAFE_FIELDS:
            safe[key] = value
        elif key in ID_FIELDS:
            # Log only a short prefix of IDs — enough to correlate
            # without exposing the full Graph API object reference
            safe[key] = str(value)[:12] + "..." if value else None
        # Intentionally exclude: body_text, context, mom_content,
        # to_emails (contains real addresses), subject (may be sensitive)

    return safe
```

-----

## PART F — How to Integrate Into Existing Tools

### Step 1 — Integrate governance rules into tools

**In `tools/mom_tools.py` — `generate_mom` function**

Find the `mom_instruction` variable and add governance rules at the end:

```python
        # Import governance rules
        from utils.governance import get_mom_rules

        mom_instruction = f"""
Generate a formal Minutes of Meeting (MOM) document.
{lang_instruction}
... (existing instruction content) ...
{get_mom_rules(include_sources=True)}
"""
```

**In `tools/task_tools.py` — `extract_tasks` function**

Find the `extraction_instruction` variable:

```python
        from utils.governance import get_task_rules

        extraction_instruction = (
            f"Extract all actionable tasks...\n"
            f"... (existing instruction) ...\n"
            f"{get_task_rules(include_sources=True)}"
        )
```

**In `tools/email_tools.py` — `summarise_email` function**

Find the `instruction` field in the return dict:

```python
        from utils.governance import get_email_rules

        return {
            ...
            "instruction": (
                "Please summarise this email in 3-4 concise sentences...\n"
                f"{get_email_rules(include_sources=True)}"
            ),
        }
```

**In `tools/draft_tools.py` — `compose_email` and `compose_reply`**

Find the `instruction` field:

```python
        from utils.governance import get_draft_rules

        return {
            ...
            "instruction": (
                f"Compose and display a professional email...\n"
                f"{get_draft_rules()}"
            ),
        }
```

**In `tools/followup_tools.py` — `compose_followup`**

```python
        from utils.governance import get_followup_rules

        return {
            ...
            "instruction": (
                f"Compose and display a professional follow-up...\n"
                f"{get_followup_rules()}"
            ),
        }
```

-----

### Step 2 — Integrate output validation into tools

**In `tools/email_tools.py` — `list_emails` function**

Add at the bottom before `return result`:

```python
        from utils.validator import validate_email_list, append_validation_to_result

        validation = validate_email_list(result["emails"])
        return append_validation_to_result(result, validation)
```

**In `tools/email_tools.py` — `summarise_email` function**

Add after building the `msg` dict:

```python
        from utils.validator import validate_email_body, append_validation_to_result

        body_text = msg.get("body", {}).get("content", "")
        validation = validate_email_body(body_text, msg.get("subject", ""))
        if not validation.is_valid:
            return append_validation_to_result({}, validation)
```

**In `tools/draft_tools.py` — `save_draft_to_outlook` function**

Add before calling `create_draft`:

```python
        from utils.validator import validate_draft_content, append_validation_to_result

        validation = validate_draft_content(
            body_text=body_text,
            to_emails=to_list,
            subject=subject,
        )
        if not validation.is_valid:
            return append_validation_to_result({}, validation)
```

**In `tools/mom_tools.py` — `save_mom_as_draft` function**

Add before saving:

```python
        from utils.validator import validate_draft_content, append_validation_to_result

        validation = validate_draft_content(
            body_text=mom_content,
            to_emails=to_list,
            subject=subject,
        )
        if not validation.is_valid:
            return append_validation_to_result({}, validation)
```

-----

### Step 3 — Integrate audit logging into tools

Add this pattern to the start of EVERY tool function, after input parsing:

```python
        from utils.audit_logger import log_tool_call, get_user_email_from_headers

        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="list_emails",        # ← change per tool
            user_email=user_email,
            inputs={"count": safe_count},   # ← safe fields only
        )
```

**Example — full integration in `list_emails`:**

```python
@mcp.tool
async def list_emails(count: int = 10) -> dict:
    logger.info(f"Tool called: list_emails (count={count})")

    try:
        safe_count = min(max(count, 1), 50)

        # Audit log — before any Graph API call
        from utils.audit_logger import log_tool_call, get_user_email_from_headers
        from utils.validator import validate_email_list, append_validation_to_result

        user_email = get_user_email_from_headers()
        log_tool_call(
            tool_name="list_emails",
            user_email=user_email,
            inputs={"count": safe_count},
        )

        raw_messages = await fetch_recent_messages(top=safe_count)

        result = []
        for msg in raw_messages:
            result.append({ ... })  # existing code unchanged

        # Build display table (existing code unchanged)
        ...

        final_result = {
            "emails": result,
            "count": len(result),
            "display_table": ...,
            "instruction": ...,
        }

        # Validate output
        validation = validate_email_list(result)
        return append_validation_to_result(final_result, validation)

    except Exception as exc:
        return format_tool_error(exc, tool_name="list_emails", logger=logger)
```

-----

## PART G — Add `logs/` to `.gitignore`

```gitignore
# Audit and application logs
logs/
*.log
```

Create the logs folder placeholder:

```bash
mkdir -p /opt/FiGPT_OutlookMCP/logs
touch /opt/FiGPT_OutlookMCP/logs/.gitkeep
```

Add `.gitkeep` exception to `.gitignore`:

```gitignore
logs/*
!logs/.gitkeep
```

-----

## PART H — Test File: `tests/test_guardrails.py`

```python
"""
test_guardrails.py
==================
Tests for utils/governance.py, utils/validator.py,
and utils/audit_logger.py.

How to run:
    python -m pytest tests/test_guardrails.py -v
"""

import pytest
import json
from pathlib import Path


# ===========================================================================
# Governance Rules Tests
# ===========================================================================

class TestGovernanceRules:
    """Tests for utils/governance.py"""

    def test_base_rules_in_email_rules(self):
        """
        get_email_rules() should contain the base rules
        prohibiting invented facts.
        """
        from utils.governance import get_email_rules
        rules = get_email_rules()
        assert "Only state facts" in rules
        assert "Never invent" in rules

    def test_email_rules_contain_source_citation(self):
        """
        get_email_rules(include_sources=True) should include
        the SOURCE CITATION block.
        """
        from utils.governance import get_email_rules
        rules = get_email_rules(include_sources=True)
        assert "Sources:" in rules

    def test_email_rules_no_citation_when_disabled(self):
        """
        get_email_rules(include_sources=False) should NOT include
        SOURCE CITATION block.
        """
        from utils.governance import get_email_rules
        rules = get_email_rules(include_sources=False)
        assert "Sources:" not in rules

    def test_mom_rules_contain_decisions_instruction(self):
        """
        get_mom_rules() should include instruction about only listing
        explicitly stated decisions.
        """
        from utils.governance import get_mom_rules
        rules = get_mom_rules()
        assert "explicitly stated" in rules

    def test_draft_rules_contain_no_promises(self):
        """
        get_draft_rules() should prohibit adding promises not
        in the context.
        """
        from utils.governance import get_draft_rules
        rules = get_draft_rules()
        assert "promises" in rules.lower()

    def test_task_rules_contain_owner_instruction(self):
        """
        get_task_rules() should include instruction about not
        guessing task owners.
        """
        from utils.governance import get_task_rules
        rules = get_task_rules()
        assert "Unassigned" in rules


# ===========================================================================
# Validator Tests
# ===========================================================================

class TestValidateEmailList:
    """Tests for validator.validate_email_list"""

    def test_valid_email_list_passes(self):
        """
        A list of properly formed emails should pass validation.
        """
        from utils.validator import validate_email_list

        emails = [
            {"id": "msg-001", "subject": "Test", "sender": "John"},
            {"id": "msg-002", "subject": "Another", "sender": "Jane"},
        ]
        result = validate_email_list(emails)
        assert result.is_valid is True
        assert len(result.errors) == 0

    def test_empty_list_generates_warning(self):
        """
        An empty email list should generate a warning but not an error.
        """
        from utils.validator import validate_email_list

        result = validate_email_list([])
        assert result.is_valid is True
        assert len(result.warnings) == 1
        assert "empty" in result.warnings[0].lower() or "No emails" in result.warnings[0]

    def test_duplicate_ids_generate_warning(self):
        """
        Duplicate email IDs should generate a validation warning.
        """
        from utils.validator import validate_email_list

        emails = [
            {"id": "msg-001", "subject": "Test 1"},
            {"id": "msg-001", "subject": "Test 2"},  # duplicate
        ]
        result = validate_email_list(emails)
        assert any("Duplicate" in w for w in result.warnings)


class TestValidateEmailBody:
    """Tests for validator.validate_email_body"""

    def test_normal_body_passes(self):
        """
        A normal email body should pass validation without warnings.
        """
        from utils.validator import validate_email_body

        result = validate_email_body(
            "Please find attached the quarterly report for your review. "
            "Let me know if you have any questions.",
            "Q3 Report"
        )
        assert result.is_valid is True
        assert len(result.warnings) == 0

    def test_empty_body_generates_warning(self):
        """
        An empty email body should generate a validation warning.
        """
        from utils.validator import validate_email_body

        result = validate_email_body("", "Test Subject")
        assert len(result.warnings) == 1

    def test_very_short_body_generates_warning(self):
        """
        A very short email body should generate a warning about
        summary quality.
        """
        from utils.validator import validate_email_body

        result = validate_email_body("Hi", "Short email")
        assert any("short" in w.lower() for w in result.warnings)


class TestValidateMomOutput:
    """Tests for validator.validate_mom_output"""

    def test_complete_mom_passes(self):
        """
        A MOM with all required sections should pass validation.
        """
        from utils.validator import validate_mom_output

        complete_mom = """
        MINUTES OF MEETING
        1. MEETING DETAILS
        2. ATTENDEES | Name | Email |
        3. DISCUSSION SUMMARY Some discussion happened here.
        4. DECISIONS MADE Decision was made.
        5. ACTION ITEMS | Task | Owner |
        6. NEXT STEPS Next meeting planned.
        7. NOTES No additional notes.
        """
        result = validate_mom_output(complete_mom)
        assert result.is_valid is True

    def test_too_short_mom_fails(self):
        """
        A MOM that is too short should fail validation.
        """
        from utils.validator import validate_mom_output

        result = validate_mom_output("Short MOM")
        assert result.is_valid is False
        assert len(result.errors) > 0

    def test_missing_section_generates_warning(self):
        """
        A MOM missing a required section should generate a warning.
        """
        from utils.validator import validate_mom_output

        mom_missing_actions = """
        MEETING DETAILS date today
        ATTENDEES John Jane
        DISCUSSION SUMMARY We talked about things.
        DECISIONS MADE We decided something.
        NEXT STEPS Meet again.
        """ * 5  # Make it long enough to pass length check

        result = validate_mom_output(mom_missing_actions)
        assert any("ACTION ITEMS" in w for w in result.warnings)


class TestValidateDraftContent:
    """Tests for validator.validate_draft_content"""

    def test_valid_draft_passes(self):
        """
        A properly formed draft should pass all validation checks.
        """
        from utils.validator import validate_draft_content

        result = validate_draft_content(
            body_text="Dear John,\n\nThank you for your email.\n\nBest regards,",
            to_emails=["john@company.com"],
            subject="Re: Project Update",
        )
        assert result.is_valid is True

    def test_invalid_email_address_fails(self):
        """
        An invalid recipient email address should fail validation.
        """
        from utils.validator import validate_draft_content

        result = validate_draft_content(
            body_text="Dear John, please review. Best regards,",
            to_emails=["not-an-email"],
            subject="Test",
        )
        assert result.is_valid is False
        assert any("valid email" in e for e in result.errors)

    def test_placeholder_text_generates_warning(self):
        """
        Placeholder text in draft body should generate a warning.
        """
        from utils.validator import validate_draft_content

        result = validate_draft_content(
            body_text="Dear [Your Name], please review the attached. Best regards,",
            to_emails=["john@company.com"],
            subject="Test",
        )
        assert any("placeholder" in w.lower() for w in result.warnings)

    def test_empty_body_fails(self):
        """
        An empty draft body should fail validation.
        """
        from utils.validator import validate_draft_content

        result = validate_draft_content(
            body_text="",
            to_emails=["john@company.com"],
            subject="Test",
        )
        assert result.is_valid is False


class TestValidateTaskList:
    """Tests for validator.validate_task_list"""

    def test_valid_tasks_pass(self):
        """
        A list of properly formed tasks should pass validation.
        """
        from utils.validator import validate_task_list

        tasks = [
            {"title": "Review the Q3 report by Friday", "owner": "John"},
            {"title": "Send updated timeline to client", "owner": "Jane"},
        ]
        result = validate_task_list(tasks)
        assert result.is_valid is True

    def test_empty_task_list_generates_warning(self):
        """
        An empty task list should generate a warning.
        """
        from utils.validator import validate_task_list

        result = validate_task_list([])
        assert len(result.warnings) == 1

    def test_duplicate_tasks_generate_warning(self):
        """
        Duplicate task titles should generate a validation warning.
        """
        from utils.validator import validate_task_list

        tasks = [
            {"title": "review the report"},
            {"title": "review the report"},  # duplicate
        ]
        result = validate_task_list(tasks)
        assert any("Duplicate" in w for w in result.warnings)


# ===========================================================================
# Audit Logger Tests
# ===========================================================================

class TestAuditLogger:
    """Tests for utils/audit_logger.py"""

    def test_log_tool_call_does_not_raise(self):
        """
        log_tool_call() should complete without raising exceptions
        even if the logs directory doesn't exist yet.
        """
        from utils.audit_logger import log_tool_call

        # Should not raise
        log_tool_call(
            tool_name="list_emails",
            user_email="test@company.com",
            inputs={"count": 10},
            outcome="success",
        )

    def test_sanitise_inputs_removes_body_content(self):
        """
        _sanitise_inputs() should not log email body text or
        context strings — only safe metadata fields.
        """
        from utils.audit_logger import _sanitise_inputs

        raw_inputs = {
            "count": 10,
            "keyword": "budget",
            "body_text": "SENSITIVE EMAIL CONTENT HERE",
            "context": "Private business context",
            "to_emails": "someone@company.com",
        }
        safe = _sanitise_inputs(raw_inputs)

        assert "count" in safe
        assert "keyword" in safe
        assert "body_text" not in safe
        assert "context" not in safe
        assert "to_emails" not in safe

    def test_sanitise_inputs_truncates_ids(self):
        """
        _sanitise_inputs() should truncate email IDs to a short
        prefix rather than logging the full ID.
        """
        from utils.audit_logger import _sanitise_inputs

        long_id = "AAMKADZmNDgyYjViLWQ40DctNDgwZi04ZmQwLWU3YjNjNjMyZT12"
        safe = _sanitise_inputs({"email_id": long_id})

        assert "email_id" in safe
        assert len(safe["email_id"]) < len(long_id)
        assert safe["email_id"].endswith("...")
```

-----

## PART I — Definition of Done

|Check                                                                               |Status|
|------------------------------------------------------------------------------------|------|
|`utils/governance.py` created                                                       |⬜     |
|`utils/validator.py` created                                                        |⬜     |
|`utils/audit_logger.py` created                                                     |⬜     |
|`logs/` folder created with `.gitkeep`                                              |⬜     |
|`.gitignore` updated to exclude `logs/`                                             |⬜     |
|`get_email_rules()` added to `summarise_email` instruction                          |⬜     |
|`get_mom_rules()` added to `generate_mom` instruction                               |⬜     |
|`get_draft_rules()` added to `compose_email` instruction                            |⬜     |
|`get_task_rules()` added to `extract_tasks` instruction                             |⬜     |
|`get_followup_rules()` added to `compose_followup` instruction                      |⬜     |
|`validate_email_list()` called in `list_emails`                                     |⬜     |
|`validate_email_body()` called in `summarise_email`                                 |⬜     |
|`validate_draft_content()` called in `save_draft_to_outlook`                        |⬜     |
|`validate_draft_content()` called in `save_mom_as_draft`                            |⬜     |
|`log_tool_call()` added to at minimum `list_emails`, `generate_mom`, `extract_tasks`|⬜     |
|All 30 tests in `test_guardrails.py` pass                                           |⬜     |
|Audit log file appears in `logs/audit.log` after a tool call                        |⬜     |
|`[INFERRED]` flags visible in MOM output when LLM is uncertain                      |⬜     |
|Placeholder text in draft triggers validation warning                               |⬜     |
|Server restarts cleanly — no import errors                                          |⬜     |

-----

*Apply in order: Part B → C → D → E → F → G → H → verify with test suite.*


---
---

Clear and structured. Let me map every tool to exactly what needs to be added — no ambiguity.

Three things to add per tool

1. log_tool_call()      → top of every tool, after input parsing
2. governance rules     → appended to the instruction field
3. validate_*()         → before returning the result


Not every tool needs all three. Here’s the exact map:

Complete Integration Map
tools/email_tools.py



|Tool                    |log_tool_call                                              |governance                                 |validate                               |
|------------------------|-----------------------------------------------------------|-------------------------------------------|---------------------------------------|
|`list_emails`           |✅ `{"count": safe_count}`                                  |❌ not needed                               |`validate_email_list(result["emails"])`|
|`read_email`            |✅ `{"email_id": email_id[:12]}`                            |❌ not needed                               |`validate_email_body(body, subject)`   |
|`search_emails`         |✅ `{"keyword": keyword, "count": safe_count}`              |❌ not needed                               |`validate_email_list(result["emails"])`|
|`summarise_email`       |✅ `{"email_id": email_id[:12]}`                            |`get_email_rules()` → append to instruction|`validate_email_body(body, subject)`   |
|`list_emails_paged`     |✅ `{"count": safe_count, "page": page}`                    |❌ not needed                               |`validate_email_list(result["emails"])`|
|`search_emails_advanced`|✅ `{"keyword": keyword, "sender": sender_email}`           |❌ not needed                               |`validate_email_list(result["emails"])`|
|`export_emails_markdown`|✅ `{"count": safe_count}`                                  |❌ not needed                               |❌ not needed                           |
|`mark_email_read`       |✅ `{"email_id": email_id[:12], "is_read": is_read}`        |❌ not needed                               |❌ not needed                           |
|`flag_email`            |✅ `{"email_id": email_id[:12], "flag_status": flag_status}`|❌ not needed                               |❌ not needed                           |

tools/attachment_tools.py



|Tool                  |log_tool_call                                                       |governance                                 |validate    |
|----------------------|--------------------------------------------------------------------|-------------------------------------------|------------|
|`list_attachments`    |✅ `{"email_id": email_id[:12]}`                                     |❌ not needed                               |❌ not needed|
|`read_attachment`     |✅ `{"email_id": email_id[:12], "attachment_id": attachment_id[:12]}`|❌ not needed                               |❌ not needed|
|`summarise_attachment`|✅ `{"email_id": email_id[:12]}`                                     |`get_email_rules()` → append to instruction|❌ not needed|

tools/calendar_tools.py



|Tool                  |log_tool_call                 |governance  |validate                          |
|----------------------|------------------------------|------------|----------------------------------|
|`list_calendar_events`|✅ `{"date_range": date_range}`|❌ not needed|`validate_calendar_events(result)`|

tools/draft_tools.py



|Tool                   |log_tool_call                             |governance                                 |validate                                             |
|-----------------------|------------------------------------------|-------------------------------------------|-----------------------------------------------------|
|`list_draft_emails`    |✅ `{"count": safe_count}`                 |❌ not needed                               |❌ not needed                                         |
|`compose_email`        |✅ `{"subject": subject}`                  |`get_draft_rules()` → append to instruction|❌ not needed                                         |
|`compose_reply`        |✅ `{"email_id": email_id[:12]}`           |`get_draft_rules()` → append to instruction|❌ not needed                                         |
|`save_draft_to_outlook`|✅ `{"subject": subject}`                  |❌ not needed                               |`validate_draft_content(body_text, to_list, subject)`|
|`find_availability`    |✅ `{"duration_minutes": duration_minutes}`|❌ not needed                               |❌ not needed                                         |
|`create_draft_invite`  |✅ `{"subject": subject}`                  |`get_draft_rules()` → append to instruction|❌ not needed                                         |

tools/folder_tools.py



|Tool           |log_tool_call                                                     |governance  |validate    |
|---------------|------------------------------------------------------------------|------------|------------|
|`list_folders` |✅ `{}`                                                            |❌ not needed|❌ not needed|
|`create_folder`|✅ `{"folder_name": folder_name}`                                  |❌ not needed|❌ not needed|
|`move_email`   |✅ `{"email_id": email_id[:12], "destination": destination_folder}`|❌ not needed|❌ not needed|

tools/task_tools.py



|Tool           |log_tool_call                                |governance                                |validate                                        |
|---------------|---------------------------------------------|------------------------------------------|------------------------------------------------|
|`extract_tasks`|✅ `{"count": safe_count, "keyword": keyword}`|`get_task_rules()` → append to instruction|`validate_task_list(tasks)` if tasks list exists|

tools/mom_tools.py



|Tool               |log_tool_call                                            |governance                               |validate                                               |
|-------------------|---------------------------------------------------------|-----------------------------------------|-------------------------------------------------------|
|`generate_mom`     |✅ `{"email_id": email_id[:12], "use_thread": use_thread}`|`get_mom_rules()` → append to instruction|`validate_mom_output(result["instruction"])`           |
|`save_mom_as_draft`|✅ `{"subject": subject}`                                 |❌ not needed                             |`validate_draft_content(mom_content, to_list, subject)`|

tools/followup_tools.py



|Tool                 |log_tool_call                                         |governance                                    |validate    |
|---------------------|------------------------------------------------------|----------------------------------------------|------------|
|`track_followups`    |✅ `{"days_threshold": threshold, "count": scan_count}`|❌ not needed                                  |❌ not needed|
|`check_email_replied`|✅ `{"email_id": email_id[:12]}`                       |❌ not needed                                  |❌ not needed|
|`compose_followup`   |✅ `{"email_id": email_id[:12]}`                       |`get_followup_rules()` → append to instruction|❌ not needed|
|`add_task_todo`      |✅ `{"title": title, "due_date": due_date}`            |❌ not needed                                  |❌ not needed|
|`add_task_planner`   |✅ `{"title": title, "due_date": due_date}`            |❌ not needed                                  |❌ not needed|
|`list_tasks`         |✅ `{"source": source}`                                |❌ not needed                                  |❌ not needed|

tools/semantic_tools.py



|Tool                    |log_tool_call                            |governance  |validate    |
|------------------------|-----------------------------------------|------------|------------|
|`semantic_search_emails`|✅ `{"query": query, "top_k": safe_top_k}`|❌ not needed|❌ not needed|

tools/chart_tools.py



|Tool            |log_tool_call                                 |governance  |validate    |
|----------------|----------------------------------------------|------------|------------|
|`generate_chart`|✅ `{"chart_type": chart_type, "title": title}`|❌ not needed|❌ not needed|

Exact code pattern for each addition
Pattern 1 — log_tool_call (add at start of every try block)

from utils.audit_logger import log_tool_call, get_user_email_from_headers

user_email = get_user_email_from_headers()
log_tool_call(
    tool_name="list_emails",          # change per tool
    user_email=user_email,
    inputs={"count": safe_count},     # safe fields only per table above
)


Pattern 2 — governance rules (append to instruction field)

from utils.governance import get_email_rules   # change per tool category

return {
    ...
    "instruction": (
        "your existing instruction text here...\n\n"
        f"{get_email_rules()}"         # appended at the end
    ),
}


Pattern 3 — validate and append (before return)

from utils.validator import validate_email_list, append_validation_to_result

validation = validate_email_list(result["emails"])
return append_validation_to_result(final_result, validation)


Priority order — do these first
If you want to start with the highest-value tools:



|Priority|Tool                             |Why                          |
|--------|---------------------------------|-----------------------------|
|1       |`generate_mom`                   |Highest hallucination risk   |
|2       |`extract_tasks`                  |Deadline/owner invention risk|
|3       |`summarise_email`                |Context mixing risk          |
|4       |`save_draft_to_outlook`          |Placeholder text risk        |
|5       |`compose_email` / `compose_reply`|Tone/promise risk            |
|6       |All others                       |Lower risk, operational tools|

Start with Priority 1–3, test, then roll out to the rest.​​​​​​​​​​​​​​​​





