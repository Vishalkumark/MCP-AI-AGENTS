# Phase 2 — Code Section 2

**Project:** outlook-ai-agent-mcp
**Covers:** `tools/email_tools.py` additions, `tools/task_tools.py`,
`tools/semantic_tools.py`, `semantic/embedder.py`,
`semantic/search_engine.py`, `server.py` updates,
`tests/` Phase 2 test files
**Status:** Code only — do not execute until reviewed

-----

## 1. `tools/email_tools.py` — Addition Only

Add this single new tool to the existing `email_tools.py` file.
Place it after the existing `search_emails` function.
Do NOT replace anything — this is an addition only.

```python
# ---------------------------------------------------------------------------
# Tool: mark_email_read
# ---------------------------------------------------------------------------
@mcp.tool
async def mark_email_read(
    email_id: str,
    is_read: bool = True,
) -> dict:
    """
    Mark a specific email as read or unread.

    Use this tool when the user asks things like:
    - "Mark this email as read"
    - "Mark John's email as unread"
    - "Set this message as unread so I remember to come back to it"

    Args:
        email_id (str): The unique Graph API ID of the email.
                         Obtained from list_emails or search_emails.
        is_read (bool): True to mark as read, False to mark as unread.
                         Defaults to True (mark as read).

    Returns:
        dict with confirmation of the read status change.
    """
    logger.info(
        f"Tool called: mark_email_read "
        f"(email_id={email_id[:20]}..., is_read={is_read})"
    )

    try:
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

-----

## 2. `tools/task_tools.py`

```python
"""
task_tools.py
=============
MCP tool for extracting actionable tasks and to-do items from emails.

Why this file exists:
    One of the core business problems this platform solves is the time
    employees spend manually reading emails to extract tasks, deadlines,
    and action owners. This tool automates that by reading recent emails
    and returning a clean, structured to-do list.

Design notes:
    - This tool does NOT call Requesty.AI internally — it returns the
      raw email content with a structured instruction for LibreChat's
      LLM to extract tasks. This follows the Phase 1 decision to let
      LibreChat's LLM handle all reasoning/summarisation.
    - The tool fetches emails, formats them clearly, and passes an
      extraction prompt as the 'instruction' field so LibreChat's LLM
      knows exactly what to do with the content.
    - Task extraction works on inbox by default but can be filtered
      by keyword or sender for targeted extraction.
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
        count (int): Number of recent emails to analyse. Default 10,
                      max 25 to keep response manageable.
        keyword (str): Optional keyword to filter emails before
                        extracting tasks e.g. "project", "deadline".
        sender_email (str): Optional filter by sender email address
                             to extract tasks only from that person's emails.

    Returns:
        dict with raw email data and a structured extraction instruction
        for LibreChat's LLM to produce the final to-do list.
    """
    logger.info(
        f"Tool called: extract_tasks "
        f"(count={count}, keyword='{keyword}', sender='{sender_email}')"
    )

    try:
        safe_count = min(max(count, 1), 25)

        # Step 1: Fetch emails — filtered or recent
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

        # Step 2: Filter by sender if specified
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
                    f"Inform the user and suggest they check the email address."
                ),
            }

        # Step 3: Build a structured email digest for task extraction.
        # Each email is formatted clearly so the LLM can parse
        # task-relevant content (action verbs, deadlines, names) easily.
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

        # Step 4: Build the task extraction instruction for LibreChat's LLM.
        # This instruction is returned alongside the raw data so
        # LibreChat processes it rather than Requesty.
        extraction_instruction = (
            "Extract all actionable tasks, to-do items, deadlines, and "
            "action owners from the email digest below. "
            "Format the output as a numbered to-do list using this structure:\n\n"
            "## 📋 Action Items\n\n"
            "| # | Task | Owner | Deadline | Source Email |\n"
            "|---|------|-------|----------|--------------|\n"
            "| 1 | [action] | [person responsible] | [date or 'Not specified'] | [email subject] |\n\n"
            "After the table, add a section:\n"
            "## ⚠️ Deadlines to Watch\n"
            "List only items with explicit deadlines mentioned.\n\n"
            "## ℹ️ Notes\n"
            "Any important context that doesn't fit as a task.\n\n"
            "If no actionable tasks are found in an email, skip it. "
            "Only include real, actionable items — not general discussion.\n\n"
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

-----

## 3. `semantic/` Folder

Create this folder at the project root:

```
outlook-ai-agent-mcp/
└── semantic/
    ├── __init__.py
    ├── embedder.py
    └── search_engine.py
```

Create `semantic/__init__.py` as an empty file.

-----

### 3.1 `semantic/embedder.py`

```python
"""
embedder.py
===========
Converts email text into numerical vector embeddings using
sentence-transformers, enabling semantic similarity search.

Why this file exists:
    Keyword search (Graph API's $search) only finds emails containing
    exact words. Semantic search understands MEANING — so a query like
    "budget concerns" can find emails about "financial worries" or
    "cost overruns" even if those exact words weren't in the query.

    This file handles the conversion:
        email text → numerical vector (list of floats)

    The vector mathematically represents the meaning of the text in a
    high-dimensional space. Two vectors close together = similar meaning.

How it works (first principles):
    1. A pre-trained model (all-MiniLM-L6-v2) has learned to convert
       sentences into 384-dimensional vectors from millions of examples.
    2. We feed it an email's subject + body preview.
    3. It returns a 384-float vector representing that email's meaning.
    4. We do the same for the user's search query.
    5. Emails whose vectors are "closest" to the query vector are
       the most semantically relevant results.

Design notes:
    - The model is loaded ONCE at module import time and reused for
      every call — loading takes ~2 seconds the first time but is
      instant after that.
    - Model: all-MiniLM-L6-v2 — small (80MB), fast, and good quality
      for English business email content.
    - First run downloads the model from HuggingFace (~80MB).
      Subsequent runs use the cached version.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from sentence_transformers import SentenceTransformer
import numpy as np

from config.settings import settings
from utils.logger import get_logger

logger = get_logger(__name__)

# ---------------------------------------------------------------------------
# Step 1: Load the sentence transformer model once at module load time.
# ---------------------------------------------------------------------------
# This is intentionally a module-level variable — loading the model is
# expensive (~2 seconds first time). Loading it once and reusing it
# across all calls is the correct pattern.
_model: SentenceTransformer = None


def get_model() -> SentenceTransformer:
    """
    Get the loaded sentence transformer model, loading it on first call.
    Uses lazy initialisation so the model only loads when first needed,
    not at server startup.

    Returns:
        SentenceTransformer: The loaded embedding model.
    """
    global _model
    if _model is None:
        model_name = settings.SEMANTIC_SEARCH_MODEL
        logger.info(f"Loading sentence transformer model: {model_name}")
        logger.info("(First load may take ~30 seconds to download the model)")
        _model = SentenceTransformer(model_name)
        logger.info(f"Model '{model_name}' loaded successfully")
    return _model


# ---------------------------------------------------------------------------
# Function: embed_text
# ---------------------------------------------------------------------------
def embed_text(text: str) -> np.ndarray:
    """
    Convert a single text string into a numerical vector embedding.

    Args:
        text (str): Any text string to embed — email subject,
                     body, or search query.

    Returns:
        np.ndarray: A 1D numpy array of floats representing the text's
                     semantic meaning. Shape: (384,) for all-MiniLM-L6-v2.
    """
    model = get_model()
    # encode() returns a numpy array automatically
    embedding = model.encode(text, convert_to_numpy=True)
    return embedding


# ---------------------------------------------------------------------------
# Function: embed_email
# ---------------------------------------------------------------------------
def embed_email(email: dict) -> np.ndarray:
    """
    Convert an email dictionary into a vector embedding by combining
    its subject and body preview into one text representation.

    Combining subject + body gives richer semantic coverage than
    either alone — subjects are often brief titles while bodies
    contain the actual context.

    Args:
        email (dict): An email dict from mail_client.py with at least
                       'subject' and 'bodyPreview' or 'preview' fields.

    Returns:
        np.ndarray: A 1D numpy array representing this email's meaning.
    """
    subject = email.get("subject") or ""
    # Support both raw Graph API format and our cleaned format
    body = (
        email.get("bodyPreview")
        or email.get("preview")
        or email.get("body", {}).get("content", "")
        or ""
    )

    # Combine subject and body — weight subject slightly more by
    # repeating it, since subject is usually the most informative.
    combined_text = f"{subject}. {subject}. {body}"

    return embed_text(combined_text)


# ---------------------------------------------------------------------------
# Function: embed_emails_batch
# ---------------------------------------------------------------------------
def embed_emails_batch(emails: list[dict]) -> np.ndarray:
    """
    Convert a list of email dicts into a matrix of embeddings.
    Batch processing is significantly faster than embedding one by one.

    Args:
        emails (list[dict]): List of email dicts.

    Returns:
        np.ndarray: A 2D numpy array of shape (len(emails), 384).
                     Each row is the embedding for one email.
    """
    if not emails:
        return np.array([])

    model = get_model()

    # Build text representations for the whole batch
    texts = []
    for email in emails:
        subject = email.get("subject") or ""
        body = (
            email.get("bodyPreview")
            or email.get("preview")
            or email.get("body", {}).get("content", "")
            or ""
        )
        texts.append(f"{subject}. {subject}. {body}")

    logger.info(f"Embedding batch of {len(texts)} emails...")

    # encode() with a list returns a 2D array automatically
    embeddings = model.encode(texts, convert_to_numpy=True, show_progress_bar=False)

    logger.info(f"Batch embedding complete. Shape: {embeddings.shape}")
    return embeddings
```

-----

### 3.2 `semantic/search_engine.py`

```python
"""
search_engine.py
================
Performs semantic similarity search over a corpus of email embeddings.

Why this file exists:
    embedder.py handles converting text → vectors.
    This file handles the actual search:
        given a query vector + a matrix of email vectors,
        find the N emails most similar to the query.

How cosine similarity works (first principles):
    Two vectors can be compared by the angle between them.
    - Angle = 0°  → identical meaning → similarity = 1.0
    - Angle = 90° → unrelated meaning → similarity = 0.0
    - Cosine similarity = cos(angle) = dot product / (length × length)

    We use scikit-learn's cosine_similarity which handles this
    computation efficiently even for large batches.

Design notes:
    - No database needed — we fetch emails fresh from Graph API,
      embed them in memory, search, and return results.
    - This is an in-memory search — for Phase 2 with up to 100 emails
      this is fast (< 1 second). A vector database (Pinecone, Chroma)
      would be added in a later phase if the corpus grows to thousands.
    - Threshold filtering removes low-relevance results — only emails
      with similarity >= MIN_SIMILARITY_THRESHOLD are returned.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

from semantic.embedder import embed_text, embed_emails_batch
from utils.logger import get_logger

logger = get_logger(__name__)

# Minimum similarity score to include in results.
# 0.0 = include everything, 1.0 = only exact matches.
# 0.25 is a good threshold for business email semantic search —
# low enough to catch related topics, high enough to filter noise.
MIN_SIMILARITY_THRESHOLD = 0.25


# ---------------------------------------------------------------------------
# Function: semantic_search
# ---------------------------------------------------------------------------
def semantic_search(
    query: str,
    emails: list[dict],
    top_k: int = 10,
) -> list[dict]:
    """
    Find the most semantically relevant emails for a given query.

    Args:
        query (str): The user's natural language search query.
                      e.g. "budget concerns from last month",
                           "project deadline reminders",
                           "John asking for report updates"
        emails (list[dict]): List of email dicts to search over.
                              Should include subject, preview/bodyPreview.
        top_k (int): Maximum number of top results to return.

    Returns:
        list[dict]: The top_k most relevant emails, each augmented with:
                     - 'similarity_score' (float 0.0–1.0)
                     - 'relevance' (str: High/Medium/Low label)
                    Sorted by similarity score descending.
    """
    if not emails:
        logger.info("Semantic search called with empty email list")
        return []

    if not query or not query.strip():
        logger.warning("Semantic search called with empty query")
        return []

    logger.info(
        f"Running semantic search: query='{query}', "
        f"corpus_size={len(emails)}, top_k={top_k}"
    )

    # Step 1: Embed the search query into a vector.
    query_vector = embed_text(query)

    # Step 2: Embed all emails into a matrix of vectors.
    email_matrix = embed_emails_batch(emails)

    if email_matrix.size == 0:
        return []

    # Step 3: Compute cosine similarity between the query vector
    #         and every email vector in the matrix.
    # query_vector shape: (384,) → reshape to (1, 384) for sklearn
    # email_matrix shape: (N, 384)
    # Result shape: (1, N) → squeeze to (N,)
    similarities = cosine_similarity(
        query_vector.reshape(1, -1),
        email_matrix
    ).squeeze()

    # Step 4: Filter by minimum threshold and sort by score.
    scored_emails = []
    for i, score in enumerate(similarities):
        if float(score) >= MIN_SIMILARITY_THRESHOLD:
            email_copy = dict(emails[i])  # don't mutate original
            email_copy["similarity_score"] = round(float(score), 4)
            email_copy["relevance"] = (
                "🟢 High" if score >= 0.65
                else "🟡 Medium" if score >= 0.40
                else "🔵 Low"
            )
            scored_emails.append(email_copy)

    # Step 5: Sort by similarity descending and return top_k.
    scored_emails.sort(key=lambda x: x["similarity_score"], reverse=True)
    top_results = scored_emails[:top_k]

    logger.info(
        f"Semantic search complete: {len(top_results)} results above threshold "
        f"(threshold={MIN_SIMILARITY_THRESHOLD})"
    )

    return top_results
```

-----

## 4. `tools/semantic_tools.py`

```python
"""
semantic_tools.py
=================
MCP tool for semantic search over email history.

Why this file exists:
    Keyword search only works when the user knows the exact words.
    Semantic search understands intent — "find emails about delays"
    will also find emails using words like "postponed", "behind schedule",
    "pushed back", even if the word "delay" never appears.

    This tool is the most AI-native capability in Phase 2 — it uses
    a pre-trained language model to understand meaning and surface
    contextually relevant emails the user might never find with
    regular search.

Design notes:
    - Fetches up to SEMANTIC_SEARCH_MAX_EMAILS emails fresh from
      Graph API on each call, embeds them in memory, runs cosine
      similarity against the query, and returns ranked results.
    - No vector database is used in Phase 2 — in-memory search is
      sufficient for up to ~200 emails (< 2 seconds).
    - The model (all-MiniLM-L6-v2) is loaded lazily on first use
      and cached for the server's lifetime.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from tools.mcp_instance import mcp

from graph.mail_client import fetch_recent_messages
from semantic.search_engine import semantic_search

from utils.error_handler import format_tool_error
from utils.logger import get_logger
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

    This is different from search_emails (keyword search).
    Use semantic_search_emails when:
    - The user describes a concept or topic, not an exact phrase
    - Keyword search returned no results but the email should exist
    - The user wants to search by meaning or sentiment

    Args:
        query (str): Natural language description of what to find.
                      e.g. "deadline pressure from management",
                           "budget approval requests",
                           "follow-up requests I haven't responded to"
        top_k (int): Number of top matching emails to return.
                      Default 10.
        corpus_size (int): How many recent emails to search over.
                            Default from settings (100). Max 200.

    Returns:
        dict with semantically ranked email results and display table.
    """
    logger.info(
        f"Tool called: semantic_search_emails "
        f"(query='{query}', top_k={top_k})"
    )

    try:
        if not query or not query.strip():
            return {
                "error": True,
                "message": "Please provide a search query.",
            }

        # Step 1: Determine corpus size
        max_corpus = min(
            corpus_size or settings.SEMANTIC_SEARCH_MAX_EMAILS,
            200
        )
        safe_top_k = min(max(top_k, 1), 25)

        # Step 2: Fetch recent emails to search over.
        # We fetch in the largest batch we can — more emails means
        # better search coverage.
        logger.info(f"Fetching {max_corpus} emails for semantic corpus...")
        raw_emails = await fetch_recent_messages(top=max_corpus)

        if not raw_emails:
            return {
                "results": [],
                "query": query,
                "display_table": "📭 No emails found to search over.",
                "instruction": "Inform the user their inbox appears empty.",
            }

        # Step 3: Run semantic search — this embeds all emails and
        #         computes cosine similarity against the query.
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
                "emails_searched": len(raw_emails),
                "display_table": (
                    f"🔍 No semantically relevant emails found for: *'{query}'*\n\n"
                    f"Searched {len(raw_emails)} recent emails. "
                    f"Try rephrasing your query or use search_emails for keyword search."
                ),
                "instruction": (
                    "Display the message above and suggest the user "
                    "try keyword search or rephrase their query."
                ),
            }

        # Step 4: Build display table with relevance scores
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
                .get("name")
                or "—"
            )
            subject = (email.get("subject") or "(no subject)")[:55]
            relevance = email.get("relevance", "—")
            score = email.get("similarity_score", 0)

            table_lines.append(
                f"| {i} | {relevance} ({score:.2f}) | {date_display} "
                f"| {sender} | {subject} |"
            )

            # Add to cleaned results without numpy arrays
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

-----

## 5. `server.py` — Updates

Two changes needed in `server.py`.

### 5.1 — Register new Phase 2 tool modules

Find the existing tool import block:

```python
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
```

**Replace with:**

```python
    try:
        # ── Phase 1 tools ────────────────────────────────────────────
        # Profile tools — get_my_profile
        from tools import profile_tools  # noqa: F401

        # Email tools — list_emails, read_email, search_emails,
        #               summarise_email, list_emails_paged,
        #               search_emails_advanced, export_emails_markdown,
        #               mark_email_read (Phase 2 addition)
        from tools import email_tools  # noqa: F401

        # Calendar tools — list_calendar_events
        from tools import calendar_tools  # noqa: F401

        # Attachment tools — list_attachments, read_attachment,
        #                    summarise_attachment
        from tools import attachment_tools  # noqa: F401

        # ── Phase 2 tools ────────────────────────────────────────────
        # Draft tools — list_draft_emails, create_draft_email,
        #               create_draft_reply, find_availability,
        #               create_draft_invite
        from tools import draft_tools  # noqa: F401

        # Folder tools — list_folders, create_folder, move_email
        from tools import folder_tools  # noqa: F401

        # Task tools — extract_tasks
        from tools import task_tools  # noqa: F401

        # Semantic tools — semantic_search_emails
        from tools import semantic_tools  # noqa: F401

        logger.info("All tool modules imported and registered successfully")
```

-----

## 6. `tests/test_phase2_tools.py`

```python
"""
test_phase2_tools.py
====================
Tests for Phase 2 tools — draft, folder, task, and semantic tools.

What this tests:
    - Draft tool output shapes
    - Folder tool output shapes
    - Task extraction instruction format
    - Semantic search with mock email corpus
    - Error handling for all new tools

How to run:
    python -m pytest tests/test_phase2_tools.py -v
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import pytest
import numpy as np
from unittest.mock import AsyncMock, patch, MagicMock


# ===========================================================================
# Draft Tool Tests
# ===========================================================================

class TestListDraftEmails:
    """Tests for list_draft_emails tool."""

    @pytest.mark.asyncio
    async def test_returns_table_when_drafts_exist(self):
        """
        list_draft_emails() should return a display_table and count
        when drafts are present.
        """
        fake_drafts = [
            {
                "id": "draft-001",
                "subject": "Re: Project Update",
                "to": ["john@company.com"],
                "last_modified": "2026-07-01T10:00:00Z",
                "preview": "Hi John, following up on the project...",
            }
        ]

        with patch(
            "tools.draft_tools.get_draft_emails",
            new=AsyncMock(return_value=fake_drafts)
        ):
            from tools.draft_tools import list_draft_emails
            result = await list_draft_emails(count=5)

        assert result["count"] == 1
        assert "display_table" in result
        assert "Re: Project Update" in result["display_table"]
        assert "instruction" in result

    @pytest.mark.asyncio
    async def test_returns_empty_message_when_no_drafts(self):
        """
        list_draft_emails() should return a clear message when
        the Drafts folder is empty.
        """
        with patch(
            "tools.draft_tools.get_draft_emails",
            new=AsyncMock(return_value=[])
        ):
            from tools.draft_tools import list_draft_emails
            result = await list_draft_emails()

        assert result["count"] == 0
        assert "empty" in result["display_table"].lower() or "No drafts" in result["display_table"]


class TestCreateDraftEmail:
    """Tests for create_draft_email tool."""

    @pytest.mark.asyncio
    async def test_returns_draft_id_on_success(self):
        """
        create_draft_email() should return a draft_id and status
        confirmation when the draft is created successfully.
        """
        fake_result = {
            "draft_id": "draft-abc-123",
            "subject": "Project Status Update",
            "to": ["john@company.com"],
            "status": "Draft saved to Drafts folder",
            "web_link": "",
        }

        with patch(
            "tools.draft_tools.create_draft",
            new=AsyncMock(return_value=fake_result)
        ):
            from tools.draft_tools import create_draft_email
            result = await create_draft_email(
                to_emails="john@company.com",
                subject="Project Status Update",
                context="Update John on the delayed timeline",
            )

        assert result["draft_id"] == "draft-abc-123"
        assert "instruction" in result
        assert "Drafts" in result["status"]

    @pytest.mark.asyncio
    async def test_returns_error_for_empty_recipients(self):
        """
        create_draft_email() should return an error dict when no
        valid recipient email addresses are provided.
        """
        from tools.draft_tools import create_draft_email
        result = await create_draft_email(
            to_emails="",
            subject="Test",
            context="Test context",
        )

        assert result.get("error") is True
        assert "recipient" in result["message"].lower()


class TestFindAvailability:
    """Tests for find_availability tool."""

    @pytest.mark.asyncio
    async def test_returns_table_when_slots_found(self):
        """
        find_availability() should return a display_table with
        suggested time slots when availability is found.
        """
        fake_result = {
            "suggestions": [
                {
                    "start": "2026-07-15T10:00:00",
                    "start_display": "Tuesday, 15 July 2026 at 10:00 UTC",
                    "end_display": "10:30 UTC",
                    "duration_minutes": 30,
                    "confidence": 85.0,
                    "quality": "🟢 High",
                    "organizer_available": "free",
                }
            ],
            "attendees_checked": ["john@company.com"],
            "duration_minutes": 30,
            "search_window_days": 7,
        }

        with patch(
            "tools.draft_tools.find_meeting_times",
            new=AsyncMock(return_value=fake_result)
        ):
            from tools.draft_tools import find_availability
            result = await find_availability(
                attendee_emails="john@company.com",
                duration_minutes=30,
            )

        assert "display_table" in result
        assert "10:00" in result["display_table"]
        assert "🟢" in result["display_table"]

    @pytest.mark.asyncio
    async def test_returns_error_for_empty_attendees(self):
        """
        find_availability() should return an error when no
        attendee emails are provided.
        """
        from tools.draft_tools import find_availability
        result = await find_availability(attendee_emails="")

        assert result.get("error") is True


# ===========================================================================
# Folder Tool Tests
# ===========================================================================

class TestListFolders:
    """Tests for list_folders tool."""

    @pytest.mark.asyncio
    async def test_returns_folder_table(self):
        """
        list_folders() should return a table with folder names
        and unread/total counts.
        """
        fake_folders = [
            {"id": "f1", "name": "Inbox", "unread_count": 5, "total_count": 120},
            {"id": "f2", "name": "Projects", "unread_count": 0, "total_count": 45},
        ]

        with patch(
            "tools.folder_tools.get_mail_folders",
            new=AsyncMock(return_value=fake_folders)
        ):
            from tools.folder_tools import list_folders
            result = await list_folders()

        assert result["count"] == 2
        assert "Inbox" in result["display_table"]
        assert "Projects" in result["display_table"]
        # Unread count should be bolded
        assert "**5**" in result["display_table"]


class TestMoveEmail:
    """Tests for move_email tool."""

    @pytest.mark.asyncio
    async def test_moves_to_well_known_folder(self):
        """
        move_email() should resolve well-known folder names like
        'archive' and call move with the correct folder ID.
        """
        fake_result = {
            "new_message_id": "msg-new-001",
            "subject": "Test Email",
            "status": "Email moved successfully",
        }

        with patch(
            "tools.folder_tools.move_message_to_folder",
            new=AsyncMock(return_value=fake_result)
        ) as mock_move:
            from tools.folder_tools import move_email
            result = await move_email(
                email_id="msg-001",
                destination_folder="archive",
            )

        # Should have called with the well-known folder name
        mock_move.assert_called_once_with(
            message_id="msg-001",
            destination_folder_id="archive",
        )
        assert result["status"] == "Email moved successfully"

    @pytest.mark.asyncio
    async def test_returns_error_for_unknown_folder(self):
        """
        move_email() should return an error when the destination
        folder name doesn't match any known folder.
        """
        with patch(
            "tools.folder_tools.get_mail_folders",
            new=AsyncMock(return_value=[
                {"id": "f1", "name": "Inbox", "unread_count": 0, "total_count": 10}
            ])
        ):
            from tools.folder_tools import move_email
            result = await move_email(
                email_id="msg-001",
                destination_folder="NonExistentFolder",
            )

        assert result.get("error") is True
        assert "not found" in result["message"].lower()


# ===========================================================================
# Task Extraction Tests
# ===========================================================================

class TestExtractTasks:
    """Tests for extract_tasks tool."""

    @pytest.mark.asyncio
    async def test_returns_instruction_with_email_digest(self):
        """
        extract_tasks() should return an instruction string containing
        the email digest and task extraction prompt for LibreChat's LLM.
        """
        fake_messages = [
            {
                "id": "msg-001",
                "subject": "Q3 Report Due Friday",
                "from": {"emailAddress": {"name": "Manager", "address": "mgr@co.com"}},
                "receivedDateTime": "2026-07-01T09:00:00Z",
                "bodyPreview": "Please submit the Q3 report by Friday 5pm.",
                "hasAttachments": False,
            }
        ]

        with patch(
            "tools.task_tools.fetch_recent_messages",
            new=AsyncMock(return_value=fake_messages)
        ):
            from tools.task_tools import extract_tasks
            result = await extract_tasks(count=5)

        assert result["emails_analysed"] == 1
        assert "instruction" in result
        assert "Action Items" in result["instruction"]
        assert "Q3 Report Due Friday" in result["email_digest"]

    @pytest.mark.asyncio
    async def test_handles_empty_inbox(self):
        """
        extract_tasks() should handle an empty inbox gracefully
        without raising an exception.
        """
        with patch(
            "tools.task_tools.fetch_recent_messages",
            new=AsyncMock(return_value=[])
        ):
            from tools.task_tools import extract_tasks
            result = await extract_tasks(count=5)

        assert result["emails_analysed"] == 0
        assert "instruction" in result


# ===========================================================================
# Semantic Search Tests
# ===========================================================================

class TestSemanticSearchEmails:
    """Tests for semantic_search_emails tool."""

    @pytest.mark.asyncio
    async def test_returns_ranked_results(self):
        """
        semantic_search_emails() should return results ranked by
        similarity score when relevant emails exist.
        """
        fake_emails = [
            {
                "id": "msg-001",
                "subject": "Budget overrun concerns",
                "from": {"emailAddress": {"name": "CFO", "address": "cfo@co.com"}},
                "receivedDateTime": "2026-07-01T09:00:00Z",
                "bodyPreview": "We need to discuss the cost overruns immediately.",
                "hasAttachments": False,
            },
            {
                "id": "msg-002",
                "subject": "Team lunch plans",
                "from": {"emailAddress": {"name": "HR", "address": "hr@co.com"}},
                "receivedDateTime": "2026-07-02T09:00:00Z",
                "bodyPreview": "Join us for lunch on Friday at noon.",
                "hasAttachments": False,
            },
        ]

        # Mock fetch_recent_messages to return our fake emails
        with patch(
            "tools.semantic_tools.fetch_recent_messages",
            new=AsyncMock(return_value=fake_emails)
        ):
            # Mock semantic_search to return a controlled result
            # so we don't need real model inference in tests
            mock_result = [
                {
                    **fake_emails[0],
                    "similarity_score": 0.82,
                    "relevance": "🟢 High",
                }
            ]
            with patch(
                "tools.semantic_tools.semantic_search",
                return_value=mock_result
            ):
                from tools.semantic_tools import semantic_search_emails
                result = await semantic_search_emails(
                    query="budget problems",
                    top_k=5,
                )

        assert result["result_count"] == 1
        assert "display_table" in result
        assert "0.82" in result["display_table"]
        assert "🟢" in result["display_table"]

    @pytest.mark.asyncio
    async def test_returns_message_when_no_results(self):
        """
        semantic_search_emails() should return a helpful message
        when no results meet the similarity threshold.
        """
        fake_emails = [
            {
                "id": "msg-001",
                "subject": "Team lunch",
                "from": {"emailAddress": {"name": "HR", "address": "hr@co.com"}},
                "receivedDateTime": "2026-07-01T09:00:00Z",
                "bodyPreview": "Join us for lunch.",
                "hasAttachments": False,
            }
        ]

        with patch(
            "tools.semantic_tools.fetch_recent_messages",
            new=AsyncMock(return_value=fake_emails)
        ), patch(
            "tools.semantic_tools.semantic_search",
            return_value=[]  # No results above threshold
        ):
            from tools.semantic_tools import semantic_search_emails
            result = await semantic_search_emails(query="quarterly financial report")

        assert result["result_count"] == 0
        assert "No semantically relevant" in result["display_table"]

    @pytest.mark.asyncio
    async def test_returns_error_for_empty_query(self):
        """
        semantic_search_emails() should return an error dict
        when no query is provided.
        """
        from tools.semantic_tools import semantic_search_emails
        result = await semantic_search_emails(query="")

        assert result.get("error") is True
```

-----

## 7. Phase 2 — Definition of Done

Before moving to Phase 3, confirm all of the following:

|Check                                                           |Status|
|----------------------------------------------------------------|------|
|`pip install sentence-transformers numpy scikit-learn` completed|⬜     |
|`requirements.txt` updated with new libraries                   |⬜     |
|`.env` updated with new scopes and semantic settings            |⬜     |
|`config/settings.py` updated with new attributes                |⬜     |
|Admin has granted `Mail.ReadWrite`, `Calendars.ReadWrite`       |⬜     |
|All 5 new graph/ files present and correct                      |⬜     |
|All 4 new tools/ files present and correct                      |⬜     |
|`semantic/` folder created with `__init__.py`                   |⬜     |
|`server.py` updated to register all 8 new tool modules          |⬜     |
|`python server.py` starts cleanly                               |⬜     |
|`curl http://localhost:8000/health` shows 25 tools              |⬜     |
|`list_draft_emails` returns Drafts folder content               |⬜     |
|`create_draft_email` saves draft to Outlook Drafts              |⬜     |
|`create_draft_reply` saves reply draft correctly                |⬜     |
|`find_availability` returns suggested time slots                |⬜     |
|`create_draft_invite` saves Calendar invite with agenda         |⬜     |
|`list_folders` returns all Outlook folders                      |⬜     |
|`create_folder` creates folder visible in Outlook               |⬜     |
|`move_email` moves email to correct folder                      |⬜     |
|`mark_email_read` marks email correctly in Outlook              |⬜     |
|`extract_tasks` returns structured to-do list from LibreChat    |⬜     |
|`semantic_search_emails` returns ranked results                 |⬜     |
|All Phase 2 pytest tests passing                                |⬜     |
|Azure DevOps commit pushed with Phase 2 changes                 |⬜     |

-----

*Phase 2 code is complete. Review, apply, test, then commit to Azure DevOps.*