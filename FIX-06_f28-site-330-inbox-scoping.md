# FIX-06 — F-28 final site: Inbox-scoping `search_messages_advanced`

**File:** `graph/mail_client.py`
**Date:** 2026-08-10 (session 13)
**Status of F-28 after this fix:** closed, pending the two verification runs in §7.

---

## 1. What problem are we solving?

**Symptom you reported:** `list_emails_paged` showed emails **you had sent** mixed in
with emails you had **received**. Ask for "my last 50 emails" and the list came back
polluted with Sent Items.

**Cause.** Microsoft Graph has two different endpoints that look almost identical:

| Endpoint | What it actually returns |
|---|---|
| `GET /me/messages` | **Every message in the entire mailbox** — Inbox, Sent Items, Archive, Deleted Items, custom folders. All of it. |
| `GET /me/mailFolders/inbox/messages` | Only what is in the Inbox. |

In the Graph SDK those are written as:

```python
client.me.messages.get(...)                                     # all folders
client.me.mail_folders.by_mail_folder_id("inbox").messages.get(...)   # inbox only
```

Every mail-reading function in `mail_client.py` used the **first** one. The existing
`filter="isDraft eq false"` removed drafts — which is why nobody noticed for weeks —
but nothing at all removed Sent Items.

It only showed up on the paged tool because that one asks for 50 messages. With
`top=10` the ten newest messages happened to all be inbox mail, so the earlier tests
passed by luck, not by correctness.

**The fix is one endpoint swap**, applied only where it belongs.

---

## 2. Why this is *not* a global find-and-replace

There were **five** `client.me.messages.get(...)` call sites. They are not all the
same kind of call, and blanket-scoping them to Inbox would have broken keyword search.

| Original line | Function | Query type | Verdict |
|---|---|---|---|
| 117 | `fetch_recent_messages` | `$filter` — recent inbox listing | **FIX** ✅ done (patch A) |
| 191 | `search_messages_by_keyword` | `$search` | **LEAVE** — see below |
| 259 | `search_messages_advanced`, keyword+date branch | `$search` | **LEAVE** — see below |
| 327 | `search_messages_advanced`, shared call | mixed — **three branches reach it** | **FIX, carefully** ← *this document* |
| 368 | `fetch_messages_paged` | `$filter` — the original symptom | **FIX** ✅ done (patch B) |

**Why `$search` must stay unscoped.** A keyword search is supposed to search your whole
mailbox. If you search for "invoice", you want to find the invoice email *you sent*
just as much as the one you received. Scoping search to the Inbox would not be a fix —
it would be a new bug, and a much worse one, because it silently loses results instead
of adding extra ones.

So the rule is:

> **Listing** the mailbox → Inbox only.
> **Searching** the mailbox → everywhere.

---

## 3. The one genuinely tricky part

Site 327 (now line 330) is **not** a simple call. Look at the structure of
`search_messages_advanced`:

```python
if keyword and filter_str:
    ...
    return [...]                      # ← RETURNS EARLY. Never reaches the shared call.

elif keyword:
    request_config = ...              # $search        → must stay ALL FOLDERS
elif filter_str:
    request_config = ...              # $filter        → must be INBOX
else:
    request_config = ...              # no filter      → must be INBOX

response = await client.me.messages.get(request_configuration=request_config)
#          ^^^^^^^^^^^^^^^^^^^^^^^^^ ONE line, reached by THREE different branches
```

Three branches share a single execution line. Scoping that line outright — which is
what the original triage implied — would have scoped the `$search` branch too, silently
breaking keyword search while appearing to fix the bug.

**Therefore the patch branches at the call itself, on `keyword`.**

That test is safe and unambiguous because the *combined* `keyword and filter_str`
branch has already returned by then. So at the shared call, `keyword` is truthy in
**exactly one** case: the pure `$search` branch. Nothing else can make it true.

```python
if keyword:      # only reachable from the elif keyword: ($search) branch
    → all folders
else:            # elif filter_str:  or  else:
    → inbox
```

**Two things that were planned and are being deliberately dropped:**

- **Adding `filter="isDraft eq false"`** — unnecessary. Once the call is Inbox-scoped,
  drafts disappear for free: they live in `/me/mailFolders/drafts`, which is a
  different folder. Adding the filter would be dead weight.
- **Adding the `select=[...]` list** — that controls response *payload size*, not
  *correctness*. It is a performance tidy-up, not part of this bug, and mixing it in
  would mean two behaviour changes landing in one paste.

Three planned edits collapsed to one. That is the right outcome.

---

## 4. What your terminal output already proves

```
194:    response = await client.me.messages.get(request_configuration=request_config)
262:        response = await client.me.messages.get(request_configuration=request_config)
330:    response = await client.me.messages.get(request_configuration=request_config)
```

Five hits became three, and the three survivors are the old 191 / 259 / 327 shifted by
exactly **+3 lines** — the net lines patch A added to `fetch_recent_messages`. That
arithmetic is the confirmation that:

- **Patch A landed** (site 117 is gone — `fetch_recent_messages` is Inbox-scoped)
- **Patch B landed** (site 368 is gone — `fetch_messages_paged` is Inbox-scoped)
- **Line 330 is the site 327 this document patches**
- 194 and 262 are the two `$search` calls that must be left exactly as they are

> ⚠️ **Before you paste anything below, answer one question:** have you *restarted the
> server and actually run* `list_emails_paged` since applying A and B? The code being in
> the file is not the same as the code working. See §7.0 — it matters, and it takes two
> minutes.

---

## 5. The complete updated file

Replace the entire contents of `graph/mail_client.py` with this.

Only **two** things differ from what you pasted, both marked with `F-28 patch C` /
`FIX-06` comments and both explained in §6. Everything else — including the
unusual indentation in a couple of places and the commented-out `"content":` line — is
preserved byte-for-byte on purpose, so the diff stays honest.

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

Folder scoping (F-28):
    client.me.messages                is GET /me/messages
                                      = EVERY folder: Inbox, Sent Items,
                                        Archive, Deleted Items, ...
    client.me.mail_folders
        .by_mail_folder_id("inbox")
        .messages                     is GET /me/mailFolders/inbox/messages
                                      = Inbox only.

    Rule: LISTING the mailbox is Inbox-scoped. SEARCHING it ($search) is
    deliberately NOT scoped — a keyword search should still find a message
    you sent. Do not "tidy" the remaining unscoped calls; they are correct.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
from graph.graph_client_factory import get_graph_client
from utils.logger import get_logger
import re

logger = get_logger(__name__)

def _clean_body(raw_content: str) -> str:
    """
    Strip HTML tags from email body content so the LLM receives
    clean plain text instead of raw HTML markup.
    Graph API returns HTML bodies by default for rich-text emails.
    """
    if not raw_content:
        return ""
    # Remove HTML tags
    text = re.sub(r"<[^>]+>", " ", raw_content)
    # Collapse multiple whitespace/newlines into single spaces
    text = re.sub(r"\s+", " ", text).strip()
    return text

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
        "is_read": message.is_read if message.is_read is not None else True,
        "bodyPreview": message.body_preview or "",
        "body": {
            # "content": message.body.content if message.body else "",
            "content": _clean_body(message.body.content if message.body else ""),
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
            filter="isDraft eq false",
            select=["id", "subject", "from", "receivedDateTime",
                    "hasAttachments", "bodyPreview", "isRead"],
    )

    request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
        query_parameters=query_params
    )

    # Step 3: Execute the call, scoped to Inbox (F-28).
    #         client.me.messages is GET /me/messages = ALL folders,
    #         which mixed Sent Items into inbox listings.
    inbox = client.me.mail_folders.by_mail_folder_id("inbox")
    response = await inbox.messages.get(request_configuration=request_config)

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
    # NOT folder-scoped, and correctly so: fetching one message by its own
    # unique ID must work regardless of which folder that message lives in.
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
        filter="isDraft eq false",
        select=["id", "subject", "from", "receivedDateTime",
                "hasAttachments", "bodyPreview", "isRead"],
    )

    # Step 2: $search has a quirk in Graph API — it requires a special
    #         header (ConsistencyLevel: eventual) to work correctly
    #         alongside certain other query options. The SDK exposes
    #         this via request headers on the configuration object.
    request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
        query_parameters=query_params,
    )
    request_config.headers.add("ConsistencyLevel", "eventual")

    # F-28: deliberately NOT Inbox-scoped. $search must span the whole
    # mailbox — a keyword search should find a message you SENT too.
    response = await client.me.messages.get(request_configuration=request_config)

    messages = response.value if response and response.value else []
    return [_message_to_dict(m) for m in messages]


# ---------------------------------------------------------------------------
# Function: search_emails_advanced
# ---------------------------------------------------------------------------
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
        # F-28: deliberately NOT Inbox-scoped — this is a $search branch.
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

    # -----------------------------------------------------------------------
    # F-28 patch C — branch-aware execution.
    #
    # THREE branches above fall through to this single call, and they do NOT
    # want the same endpoint. Do not collapse this back into one line.
    #
    #   elif keyword:     $search   -> ALL FOLDERS is correct and intended.
    #                                  A keyword search must still find mail
    #                                  the user SENT. Scoping it to Inbox
    #                                  would silently lose results.
    #   elif filter_str:  $filter   -> INBOX, or Sent Items leak in (F-28).
    #   else:             listing   -> INBOX, same reason.
    #
    # Testing `keyword` here is unambiguous: the combined `keyword and
    # filter_str` branch RETURNED EARLY above, so by this point `keyword` is
    # truthy in exactly one case — the pure $search branch.
    #
    # Inbox scoping also removes drafts for free (they live in
    # /me/mailFolders/drafts), so no `isDraft eq false` filter is needed here.
    # -----------------------------------------------------------------------
    if keyword:
        response = await client.me.messages.get(request_configuration=request_config)
    else:
        inbox = client.me.mail_folders.by_mail_folder_id("inbox")
        response = await inbox.messages.get(request_configuration=request_config)

    messages = response.value if response and response.value else []
    return [_message_to_dict(m) for m in messages]


# ---------------------------------------------------------------------------
# Function: fetch_messages_paged
# ---------------------------------------------------------------------------

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
        filter="isDraft eq false",
        select=["id", "subject", "from", "receivedDateTime",
                "hasAttachments", "bodyPreview", "isRead"],
    )

    request_config = MessagesRequestBuilder.MessagesRequestBuilderGetRequestConfiguration(
            query_parameters=query_params
        )

    # Inbox-scoped (F-28) — see fetch_recent_messages.
    inbox = client.me.mail_folders.by_mail_folder_id("inbox")
    response = await inbox.messages.get(request_configuration=request_config)
    messages = response.value if response and response.value else []
    return [_message_to_dict(m) for m in messages]
```

---

## 6. Exactly what changed

Two edits. Nothing else.

### Change 1 — the actual fix (`search_messages_advanced`, old line 330)

**Was:**
```python
    response = await client.me.messages.get(request_configuration=request_config)
```

**Now:**
```python
    if keyword:
        response = await client.me.messages.get(request_configuration=request_config)
    else:
        inbox = client.me.mail_folders.by_mail_folder_id("inbox")
        response = await inbox.messages.get(request_configuration=request_config)
```
(plus the block comment above it explaining why, so nobody "simplifies" it later)

### Change 2 — dead variable left behind by patch A (`fetch_recent_messages`)

**Was:**
```python
    inbox = client.me.mail_folders.by_mail_folder_id("inbox")
    response = await client.me.mail_folders.by_mail_folder_id("inbox").messages.get(
        request_configuration=request_config
    )
```
The `inbox` variable was assigned and then never used — the call rebuilt the same
builder inline. Harmless, but it means the two Inbox-scoped functions are written
differently, which is exactly how a later "cleanup" introduces a bug.

**Now:**
```python
    inbox = client.me.mail_folders.by_mail_folder_id("inbox")
    response = await inbox.messages.get(request_configuration=request_config)
```
Identical to how `fetch_messages_paged` already does it. No behaviour change.

### Also added (comments only, zero behaviour)

- A folder-scoping note in the module docstring, so the rule is stated once at the top.
- `# F-28: deliberately NOT Inbox-scoped` on both surviving `$search` calls (lines 194
  and 262 in your listing). Those comments exist to stop a future reader — including me
  — from "finishing the job" and breaking search.
- A note on `fetch_message_by_id` explaining why fetching one message by ID is
  correctly unscoped.

---

## 7. Verification

### 7.0 — Do this FIRST, before pasting anything

Patches A and B are in the file, but *code being present is not the same as code
working*. All three patches use the same construct: passing the existing
`MessagesRequestBuilderGetRequestConfiguration` object to a **folder-scoped** builder.

That should be fine — in the Graph SDK those per-builder config classes are aliases of
the same generic `RequestConfiguration` type, so the folder-scoped builder accepts it —
but "should be fine" is exactly what was said about `EventsRequestBuilder` before L-02
crashed on it. Prove it on one site before betting three on it.

```bash
# restart the server, then in LibreChat:
#   "show me my last 50 emails"
```

- **Returns mail** → the construct works. Proceed to §7.1.
- **Throws an SDK / type error** → **stop, do not paste FIX-06.** Send me the traceback.
  The fix is a one-line import swap to the folder-scoped builder class, and it will fix
  all three sites at once.

### 7.1 — Apply and restart

```bash
cd /opt/FiGPT_OutlookMCP
cp graph/mail_client.py /tmp/mail_client.py.bak     # 5-second insurance
# ...paste the file from §5...
python -c "import ast;ast.parse(open('graph/mail_client.py').read())" && echo "SYNTAX OK"
```

### 7.2 — Confirm the call sites are now what we expect

```bash
grep -n 'client.me.messages.get\|inbox.messages.get' graph/mail_client.py
```

Expect **five** hits: two `client.me.messages.get` (the `$search` calls), and three
`inbox.messages.get`. If you still see a bare `client.me.messages.get` that is *not*
inside a `$search` branch, the paste didn't fully land.

### 7.3 — The bug is fixed

```bash
pytest -q                          # expect 112 passing
```
Then in LibreChat:
1. *"Show me my last 50 emails"* → **no Sent Items.** This is the original F-28 symptom.
2. *"Show me emails from `<a colleague>` in the last month"* → hits the `$filter`
   branch, the one this patch scopes. **No Sent Items.**

### 7.4 — ⚠️ The regression test that actually matters

**Do not skip this one.** Without it, "no Sent Items in the results" is
indistinguishable from having accidentally narrowed the entire tool.

> In LibreChat: search by **keyword** for a distinctive word that appears in an email
> **you sent** — not one you received.
>
> **It must still be found.**

If a keyword search can no longer find your own sent mail, the `if keyword:` branch is
not being taken and the patch has broken search. Revert from `/tmp/mail_client.py.bak`
and tell me.

---

## 8. ⚠️ One trap with pasting the whole file — read before you commit

`graph/mail_client.py` is one of the 38 files carrying **whole-file CRLF** line endings
(F-29 — 369 carriage returns in this file, inherited from the project's Windows origin).

The file in §5 has **LF** endings. So when you paste it, `git diff` will report that
**every single line changed**, and the two real edits will be invisible in the noise.

That is cosmetic, not dangerous — but check the real change before committing:

```bash
git diff --stat graph/mail_client.py            # will look alarming. It is fine.
git diff --ignore-cr-at-eol graph/mail_client.py   # ← the TRUE change. Should be small.
```

The second command should show only the two edits from §6 plus the added comments.
**If it shows anything else, stop and tell me** — something got mangled in the paste.

Two clean ways forward, your call:

- **(a) Accept it.** This file becomes LF, which is where F-29 is heading anyway. Commit
  it on its own: `git commit -m "F-28: inbox-scope search_messages_advanced (+ normalise line endings)"`.
  Don't bundle it with other work — the diff is too noisy to review alongside anything else.
- **(b) Keep CRLF** so the diff stays surgical:
  ```bash
  sed -i 's/$/\r/' graph/mail_client.py     # AFTER pasting — restores CRLF
  git diff graph/mail_client.py             # now shows only the real change
  ```

Reminder from F-07: `.gitignore` still has no `.venv/` line, so **stage by explicit path
only** — `git add graph/mail_client.py`. Never `git add -A` or `git add .` until that
line exists, or 44,962 venv files come straight back into the index.

---

## 9. Appendix — surgical alternative

If you would rather not replace the whole file, this is the minimum change. Apply
**by position** — use the line numbers, because the "Find" text below is not unique
in the file.

**Edit 1 — line 330** (the last `response = await client.me.messages.get(...)` in
`search_messages_advanced`, the one immediately followed by
`messages = response.value ...` and `return [_message_to_dict(m) for m in messages]`).

Find (line 330 only — the identical line also exists at 194 and 262, **leave those**):
```python
    response = await client.me.messages.get(request_configuration=request_config)
```

Replace with:
```python
    # F-28 patch C — branch-aware. Three branches reach this call and they do
    # NOT want the same endpoint. $search must span ALL folders (a keyword
    # search should still find mail you SENT); $filter and the no-filter
    # listing must be Inbox-scoped or Sent Items leak in. `keyword` is truthy
    # here only in the $search branch — the combined branch returned early.
    if keyword:
        response = await client.me.messages.get(request_configuration=request_config)
    else:
        inbox = client.me.mail_folders.by_mail_folder_id("inbox")
        response = await inbox.messages.get(request_configuration=request_config)
```

**Edit 2 — in `fetch_recent_messages`** (optional tidy, no behaviour change).

Find:
```python
    response = await client.me.mail_folders.by_mail_folder_id("inbox").messages.get(
        request_configuration=request_config
    )
```

Replace with:
```python
    response = await inbox.messages.get(request_configuration=request_config)
```

Verification is the same — §7.2, §7.3 and especially **§7.4**.

---

## 10. Observations noted, deliberately NOT changed

Recorded for the ledger. None of these are part of this fix; raising them here so they
are not rediscovered as new findings later.

1. **`search_messages_by_keyword` passes `$search` *and* `$filter` together** (`search=`
   plus `filter="isDraft eq false"`) — while the comment inside `search_messages_advanced`
   states these cannot be combined, which is why that function goes to the trouble of
   filtering client-side. One of the two is wrong. Empirically the keyword tool works
   (TEST-01 T2.1/T2.2 passed), so Graph is evidently tolerating it in this tenant —
   but the codebase is holding two contradictory beliefs about the same API rule, and
   the tolerant behaviour is not something to rely on. **Worth one deliberate check**,
   not a blind edit.

2. **The three branches in `search_messages_advanced` have no `select=[...]`**, unlike
   every other function here. Not a correctness bug — omitting `$select` returns *more*
   fields, not fewer, so `is_read` and the rest still populate. It does mean those
   branches pull full message bodies over the wire. Payload size only; fold into a
   later performance pass.

3. **`from dateutil import parser as dp`** inside the combined branch is imported and
   never used. Dead line, F-22 family.

4. **`fetch_recent_messages` requests `select=[...]` without `body`**, so
   `_message_to_dict` renders `"body": {"content": ""}` for every message from that
   path — preview only, by design. Just noting it is intentional, so it does not get
   "fixed" into a large payload later.

---


 Recalled 1 memory, wrote 1 memory

Delivered: E:\Projects\Outlook-mcp-proj\FIX-06_f28-site-330-inbox-scoping.md

What your terminal output told me

Your grep returned 3 hits where there were 5 — at lines 194, 262, 330. Those are the old sites 191/259/327 shifted +3, exactly the net lines patch A added.

So patches A and B are already applied. Sites 117 and 368 are gone from the grep, and the file you pasted confirms both now use mail_folders.by_mail_folder_id("inbox"). That was the confusion — the ledger still listed them as pending.

The problem, in one line

client.me.messages is GET /me/messages, which means every folder — Inbox, Sent Items, Archive, everything. The isDraft eq false filter removed drafts, so nobody noticed, but nothing removed Sent Items. Listing 50 emails reached back far enough to surface them.

What's actually left: one call, and it isn't a simple swap

Of the three remaining sites, two must stay exactly as they are — 194 and 262 are $search. A keyword search is supposed to span the whole mailbox; if you search "invoice" you want the one you sent too. Scoping those wouldn't be a fix, it'd be a worse bug that silently loses results.

Line 330 is the awkward one: three branches fall through to that single line, and they don't all want the same endpoint.

if keyword and filter_str:   ... return [...]   # returns EARLY — never gets there
elif keyword:                # $search  → ALL folders (correct)
elif filter_str:             # $filter  → must be Inbox
else:                        # listing  → must be Inbox
response = await client.me.messages.get(...)     # ← all three land here

So the patch branches at the call, on keyword. That test is safe precisely because the combined branch already returned — keyword is truthy there in exactly one case.

What's in the file

- Plain-English problem statement and the two-endpoint explanation
- The complete graph/mail_client.py, ready to paste — preserved byte-for-byte apart from two marked changes
- A section listing exactly what changed (the fix, plus a dead inbox variable patch A left behind in fetch_recent_messages)
- Verification, including §7.4 — the regression test that matters: a keyword search must still find a message you sent. Without it, "no Sent Items" is indistinguishable from having narrowed the whole tool.

Two things to flag before you paste

§7.0 is a gate, not a suggestion. A and B are in the file but have never actually run. All three patches rest on the same assumption — that the folder-scoped builder accepts the existing MessagesRequestBuilderGetRequestConfiguration. It very likely does (those per-builder config classes are aliases of one generic type), but that is exactly the confidence level that preceded the L-02 EventsRequestBuilder crash. Restart, ask for your last 50 emails. If it throws instead of returning mail, stop and send me the traceback — one import swap fixes all three.

§8 — the CRLF trap. This file has 369 carriage returns (F-29). Pasting LF content makes git diff show every line changed and hides the real edit. Check with git diff --ignore-cr-at-eol — it should show only the two changes. .gitignore still lacks .venv/, so stage by explicit path.

State file updated: F-28 row, §7, and a session-13 log entry. I've also recorded complete-file delivery as the default going forward rather than positional patch blocks — you're right that asking you to apply at line 330 and not the identical line at 194 puts the risk on the person who can't see the file.

✻ Churned for 5m 38s

※ recap: Goal is hardening the Outlook MCP server; current task is finishing F-28 so email listings stop showing Sent Items. Next: restart and run a 50-email listing to prove patches A and B work before pasting FIX-06. (disable recaps in /config)

---

