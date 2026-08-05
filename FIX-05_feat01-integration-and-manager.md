# FIX-05 — FEAT-01 integration fixes + manager lookup

> **Session 11 — 2026-08-05.** Fixes the 6 failing tests, makes `bin/diag_tasks.py`
> runnable, and corrects the AI-generated manager code.
>
> **Nothing here is caused by the `.venv` change.** See §0.
>
> Work through §1 → §7 **in order**. Each step has its own verification.

---

## 0. Read this first — three things that are NOT broken

| You thought | Reality |
|---|---|
| "The `.venv` changes affected the project" | **No.** `git rm -r --cached .venv` removes files from git's *index* only — it never touches disk. Your proof: the venv still activates, and **106 of 112 tests pass**. A broken venv fails all 112 with `ModuleNotFoundError`, not 6 with `TypeError`. F-07 landed correctly. |
| "`sed -i` printed nothing, so it failed" | **No.** `sed -i` edits in place and is **silent on success**. Printing nothing *is* the success case. Verify with `grep -c $'\r' .env` → expect `0`. |
| "The 6 failures are unrelated bugs" | **No.** They are **three** helper-signature mismatches. FEAT-01 §4 told you to reconcile them against the live files; that step got skipped, which is entirely fair — this fix does it for you with the real signatures. |

**The three real signatures, confirmed from the codebase:**

```python
# utils/table_utils.py:25
def table_cell(text: str, limit: int) -> str:          # limit is REQUIRED, no default

# utils/audit_logger.py:79
def log_tool_call(tool_name, user_email, inputs, ...)  # 3 required args, not 2

# utils/error_handler.py:44
def format_tool_error(exc, tool_name, logger=None)     # exception FIRST, not the name

# utils/governance.py:165
def get_task_rules(include_sources: bool = True) -> str   # this is the real name
```

`get_governance_rules()` **does not exist anywhere in this project.** That was a
placeholder in FEAT-01 §7 with a note to replace it. The correct function for task
tools is `get_task_rules()`, and `tools/task_tools.py` **already imports it** at the
top of the file — you do not need to add an import.

---

## 1. Fix `utils/task_grouping.py` — closes 3 test failures

### 1.1 Replace the import block

Near the top of the file.

**Find:**

```python
try:
    from utils.table_utils import _table_cell as table_cell
except ImportError:  # pragma: no cover
    from utils.table_utils import table_cell
```

**Replace with:**

```python
# utils/table_utils.py exposes the cell sanitiser that closed F-12/F-15.
# Its real signature is table_cell(text, limit) — the limit is REQUIRED and has
# no default. Wrap it so this module can pass a per-column width and so None
# values (Graph omits fields freely) never reach a str method.
from utils.table_utils import table_cell as _raw_table_cell


def table_cell(text, limit: int = 60) -> str:
    """Sanitise one markdown table cell: collapse whitespace, escape pipes, truncate."""
    return _raw_table_cell("" if text is None else str(text), limit)
```

### 1.2 Replace the `_table` function

**Find the whole function** (it starts `def _table(tasks, container_label, extra_label, now) -> list[str]:`)
and **replace it entirely with:**

```python
def _table(tasks, container_label, extra_label, now) -> list[str]:
    """
    Build one markdown table.

    Every cell goes through table_cell(): task titles routinely contain pipes
    and line breaks, and a markdown table is line-based, so an unescaped title
    splits the row and mangles everything below it. That was F-12/F-15.

    Column widths are explicit. table_cell() takes a REQUIRED limit — passing a
    per-column value keeps the Task column readable without letting a 200-char
    title push the Plan and Status columns off the right edge in LibreChat.
    """
    lines = [
        f"| Due | Task | {container_label} | Status | {extra_label} |",
        "|---|---|---|---|---|",
    ]
    for t in tasks:
        lines.append(
            "| {due} | {title} | {container} | {status} | {extra} |".format(
                due=table_cell(_due_cell(t.get("due"), now), 24),
                title=table_cell(t.get("title", ""), 70),
                container=table_cell(t.get("container", ""), 32),
                status=table_cell(t.get("status", ""), 18),
                extra=table_cell(t.get("extra", ""), 16),
            )
        )
    return lines
```

**Verify:**

```bash
python -m pytest tests/test_task_grouping.py -q
```

Expect **all passing**. (Was 3 failed.)

---

## 2. Fix `tools/task_tools.py` — closes the other 3 test failures

Three separate edits in the block you appended. All three are in the FEAT-01
section at the bottom of the file.

### 2.1 `get_governance_rules()` → `get_task_rules()` — 3 places

This is the one you said you don't understand, so here it is plainly:
**FEAT-01 wrote a function name that does not exist.** Your project's equivalent
is `get_task_rules()`, already imported on line ~36 of this file
(`from utils.governance import get_task_rules`). It is a *function*, so it keeps
its `()`.

Run this **one command** and all three are fixed at once:

```bash
cd /opt/FiGPT_OutlookMCP
sed -i 's/get_governance_rules()/get_task_rules()/g' tools/task_tools.py
grep -n "get_task_rules()" tools/task_tools.py
```

Expect **3 lines** in the grep output (one in `_unavailable`, one in
`list_planner_tasks`, one in `list_todo_tasks`), and `get_governance_rules`
gone entirely:

```bash
grep -c "get_governance_rules" tools/task_tools.py   # expect 0
```

### 2.2 Fix both `log_tool_call(...)` calls

**Find** (in `list_planner_tasks`):

```python
    log_tool_call("list_planner_tasks", {"include_completed": include_completed})
```

**Replace with:**

```python
    # log_tool_call(tool_name, user_email, inputs) — three required args. The
    # user_email is read from the request headers, never from the token body.
    log_tool_call(
        tool_name="list_planner_tasks",
        user_email=get_user_email_from_headers(),
        inputs={"include_completed": include_completed},
    )
```

**Find** (in `list_todo_tasks`):

```python
    log_tool_call("list_todo_tasks", {"include_completed": include_completed})
```

**Replace with:**

```python
    log_tool_call(
        tool_name="list_todo_tasks",
        user_email=get_user_email_from_headers(),
        inputs={"include_completed": include_completed},
    )
```

`get_user_email_from_headers` is **already imported** at the top of this file —
no new import needed.

### 2.3 Fix both `format_tool_error(...)` calls — latent bug, no test covers it

The arguments are **backwards**. This has not failed yet only because no test
exercises the non-403 error path — in production, a network blip would raise
`TypeError` *inside the exception handler*, which is the worst place for it.

**Find** (in `list_planner_tasks`):

```python
        return format_tool_error("list_planner_tasks", exc)
```

**Replace with:**

```python
        return format_tool_error(exc, tool_name="list_planner_tasks", logger=logger)
```

**Find** (in `list_todo_tasks`):

```python
        return format_tool_error("list_todo_tasks", exc)
```

**Replace with:**

```python
        return format_tool_error(exc, tool_name="list_todo_tasks", logger=logger)
```

**Verify §2:**

```bash
python -m pytest tests/ -q
```

Expect **112 passed, 0 failed**.

---

## 3. Make `bin/diag_tasks.py` actually run

Two separate problems, both real:

1. `ModuleNotFoundError: No module named 'graph'` — Python puts the **script's own
   directory** (`bin/`) on `sys.path`, not your current directory. So `graph/` is
   invisible. `bin/agent_test.py` already solves this; `diag_tasks.py` didn't copy it.
2. `AuthorizationError: Missing Authorization header` — `get_current_access_token()`
   reads the token out of the **live HTTP request headers**. A standalone script has
   no request. It needs the token injected.

**Replace the entire contents of `bin/diag_tasks.py` with this.** It also prints
the token's `scp` claim first, which answers the question that otherwise costs a
day: *did the new Azure grant actually reach the token?*

```python
"""
Throwaway diagnostic. Proves the Graph endpoints and SDK paths behind FEAT-01
work against the live tenant BEFORE trusting any tool code.

Run:
    cd /opt/FiGPT_OutlookMCP
    source .venv/bin/activate
    python bin/get_token.py        # mint a fresh token first
    python bin/diag_tasks.py
"""
import asyncio
import base64
import json
import sys
from pathlib import Path

# --- 1. Put the project root on sys.path -----------------------------------
# Python puts the SCRIPT's directory (bin/) on sys.path, not the cwd, so
# "from graph...." fails without this. Same bootstrap bin/agent_test.py uses.
PROJECT_ROOT = Path(__file__).resolve().parents[1]
if str(PROJECT_ROOT) not in sys.path:
    sys.path.insert(0, str(PROJECT_ROOT))

TOKEN_FILE = Path(__file__).resolve().parent / "token.json"


def _load_token() -> str:
    if not TOKEN_FILE.exists():
        print("ERROR: bin/token.json not found.")
        print("       Run:  python bin/get_token.py")
        raise SystemExit(1)
    data = json.loads(TOKEN_FILE.read_text())
    token = data.get("access_token")
    if not token:
        print("ERROR: bin/token.json has no 'access_token' key.")
        raise SystemExit(1)
    return token


def _token_scopes(token: str) -> str:
    """Decode the JWT payload and return the 'scp' claim. No signature check —
    we are reading our own token to see what it carries, not trusting it."""
    try:
        payload = token.split(".")[1]
        payload += "=" * (-len(payload) % 4)          # restore base64 padding
        claims = json.loads(base64.urlsafe_b64decode(payload))
        return claims.get("scp", "") or claims.get("roles", "")
    except Exception as exc:
        return f"<could not decode: {exc}>"


# --- 2. Inject the token BEFORE importing the client factory ---------------
# get_current_access_token() reads live HTTP request headers. A standalone
# script has none, which is the AuthorizationError you hit. Patching the
# module attribute is safe: Python resolves module globals at CALL time, and
# nothing in the production path is modified.
_TOKEN = _load_token()

import graph.graph_client_factory as factory          # noqa: E402
factory.get_current_access_token = lambda: _TOKEN

from graph.graph_client_factory import get_graph_client   # noqa: E402


async def main() -> int:
    failures = 0

    # ---- 0. What does this token actually carry? ------------------------
    print("=" * 70)
    print("0. TOKEN SCOPES  (the 'scp' claim)")
    scopes = _token_scopes(_TOKEN)
    print(f"   {scopes}")
    if "Tasks.Read" not in scopes:
        print("   *** Tasks.Read / Tasks.ReadWrite is NOT in this token. ***")
        print("   *** Granting the permission in Azure does not upgrade an  ***")
        print("   *** existing token. Add it to GRAPH_API_SCOPES in .env,   ***")
        print("   *** then re-run bin/get_token.py. Steps 1-4 will 403.     ***")
    print()

    client = get_graph_client()

    # ---- 1. Planner: tasks assigned to me -------------------------------
    print("=" * 70)
    print("1. GET /me/planner/tasks")
    try:
        resp = await client.me.planner.tasks.get()
        tasks = resp.value if resp and resp.value else []
        print(f"   OK - {len(tasks)} task(s)")
        for t in tasks[:3]:
            print(f"     - {t.title!r}")
            print(f"       plan_id      = {t.plan_id}")
            print(f"       due          = {t.due_date_time!r}")
            print(f"       created      = {t.created_date_time!r}")
            print(f"       percent      = {t.percent_complete}")
            print(f"       priority     = {t.priority}")
        plan_ids = sorted({t.plan_id for t in tasks if t.plan_id})
        print(f"   distinct plans: {len(plan_ids)}")
    except Exception as exc:
        failures += 1
        plan_ids = []
        print(f"   FAILED - {type(exc).__name__}: {exc}")

    # ---- 2. Planner: resolve a plan title -------------------------------
    print("=" * 70)
    print("2. GET /planner/plans/{id}   (needed for 'Plan Name' grouping)")
    if not plan_ids:
        print("   SKIPPED - no plan ids from step 1")
    else:
        try:
            plan = await client.planner.plans.by_planner_plan_id(plan_ids[0]).get()
            print(f"   OK - plan title = {plan.title!r}")
        except Exception as exc:
            failures += 1
            print(f"   FAILED - {type(exc).__name__}: {exc}")

    # ---- 3. To-Do: lists ------------------------------------------------
    print("=" * 70)
    print("3. GET /me/todo/lists")
    lists = []
    try:
        resp = await client.me.todo.lists.get()
        lists = resp.value if resp and resp.value else []
        print(f"   OK - {len(lists)} list(s)")
        for l in lists:
            print(f"     - {l.display_name!r}  (wellknown={l.wellknown_list_name})")
    except Exception as exc:
        failures += 1
        print(f"   FAILED - {type(exc).__name__}: {exc}")

    # ---- 4. To-Do: tasks in the first list ------------------------------
    print("=" * 70)
    print("4. GET /me/todo/lists/{id}/tasks")
    if not lists:
        print("   SKIPPED - no lists from step 3")
    else:
        try:
            builder = client.me.todo.lists.by_todo_task_list_id(lists[0].id).tasks
            resp = await builder.get()
            items = resp.value if resp and resp.value else []
            print(f"   OK - {len(items)} task(s) in {lists[0].display_name!r}")
            for t in items[:3]:
                print(f"     - {t.title!r}")
                print(f"       status   = {t.status!r}")
                print(f"       due      = {t.due_date_time!r}   <-- SHAPE MATTERS")
                print(f"       created  = {t.created_date_time!r}")
            print(f"   has with_url(): {hasattr(builder, 'with_url')}   <-- paging")
            print(f"   next link     : {getattr(resp, 'odata_next_link', None)!r}")
        except Exception as exc:
            failures += 1
            print(f"   FAILED - {type(exc).__name__}: {exc}")

    # ---- 5. Manager (the new profile requirement) -----------------------
    print("=" * 70)
    print("5. GET /me/manager")
    try:
        mgr = await client.me.manager.get()
        if mgr is None:
            print("   returned None - no manager object")
        else:
            print(f"   OK - display_name = {getattr(mgr, 'display_name', None)!r}")
            print(f"        mail         = {getattr(mgr, 'mail', None)!r}")
            print(f"        job_title    = {getattr(mgr, 'job_title', None)!r}")
            print(f"        python type  = {type(mgr).__name__}")
    except Exception as exc:
        print(f"   FAILED - {type(exc).__name__}: {exc}")
        print("   A 404 / ResourceNotFound here means the Manager attribute is")
        print("   simply not set on your account in Entra ID. That is a directory")
        print("   data problem, not a code problem - see FIX-05 section 5.")

    print("=" * 70)
    print("ALL PASSED" if failures == 0 else f"{failures} STEP(S) FAILED")
    return failures


if __name__ == "__main__":
    sys.exit(asyncio.run(main()))
```

---

## 4. Add the new scopes to `.env` — do this BEFORE running the diagnostic

Your Azure permission table is complete: `Tasks.Read`, `Tasks.ReadWrite`,
`Tasks.ReadWrite.Shared`, `Group.Read.All` and `User.Read.All` are all granted.
**B-01 is unblocked.** But a granted permission does not appear in a token that
was minted before the grant — the token must be re-issued with the scope
explicitly requested.

Open `.env` and find the `GRAPH_API_SCOPES` line. Set it to:

```bash
GRAPH_API_SCOPES=User.Read Mail.Read Mail.ReadWrite Mail.Send Calendars.Read Calendars.ReadWrite Tasks.ReadWrite Tasks.ReadWrite.Shared Group.Read.All User.Read.All
```

Notes on that list:
- `Tasks.ReadWrite` **supersedes** `Tasks.Read` — asking for both is redundant, so
  `Tasks.Read` is omitted deliberately.
- Do **not** add `offline_access`, `openid`, `profile` here if they are handled
  elsewhere in `get_token.py`; MSAL adds them automatically and will reject them
  if passed in the scope list.

Then apply the same two lines to `.env.example` (FEAT-01 §8):

```bash
# ── Task tools (FEAT-01) ────────────────────────────────────────
# Horizon in days for the "due soon" group in list_planner_tasks and
# list_todo_tasks. Anything due beyond this falls into the third group.
TASK_HORIZON_DAYS=14

# Optional — only needed for the group-wide Planner extension (FEAT-01 §10).
# Leave unset for the default "tasks assigned to me" behaviour.
# PLANNER_GROUP_ID=
```

Strip CRs and re-mint the token:

```bash
sed -i 's/\r$//' .env .env.example
grep -c $'\r' .env .env.example        # expect 0 for both

python bin/get_token.py                # sign in again -> new token.json
python bin/diag_tasks.py
```

Step 0 of the diagnostic prints the token's scopes. **If `Tasks.ReadWrite` is not
in that line, stop** — nothing downstream will work, and it means `.env` was not
picked up.

---

## 5. The manager requirement — verdict on the AI's code

**Overall: the approach is right, the endpoint is right, no new permission is
needed. But there are three real bugs, and one of them is exactly why you see
`None | None`.**

### 5.1 Why you get `None | None` — two independent causes

**Cause A — the directory.** `GET /me/manager` returns **HTTP 404
`ResourceNotFound`** when the Manager attribute is not set. It does *not* return
a JSON object with null fields. Your `curl` piped that 404 error body into
`d.get('displayName')`, which returned `None`. So `Manager: None | None` is the
404 in disguise. **The manager is genuinely not set on your account in Entra ID.**
The AI's closing advice was correct: an admin must set
*Entra ID → Users → your account → Properties → Job information → Manager*.
No code change makes a missing directory attribute appear.

**Cause B — a real bug in the AI's rendering code**, which would show `None`
*even after* your admin sets the manager:

```python
f"| **Job Title** | {profile_data.get('jobTitle', '—')} |"
```

`dict.get(key, default)` returns the default **only when the key is absent**. The
key is always present here — set to `None` — so this renders the literal string
`None`, never `—`. Every field in that instruction block has the bug. The fix is
`or '—'`, not `, '—'`.

### 5.2 The other two bugs

- **`department` will always be `None`.** `GET /me` returns Graph's *default*
  property set, and `department` is not in it. You must `$select` it explicitly.
  The AI's `getattr(user, "department", None)` reads an attribute the SDK never
  populated.
- **Dead import.** `from msgraph.generated.models.user import User as GraphUser`
  is imported and never used.

### 5.3 One architectural correction

The AI put a full markdown table inside `instruction`. **Every other tool in this
project returns the table in `display_table` and keeps `instruction` as
directions to the model.** Mixing them breaks the convention `utils/validator.py`
and every LibreChat behaviour test assume. The replacement below follows the
house pattern.

### 5.4 Replacement 1 — `graph/graph_client_factory.py`

**Find the entire `get_user_profile` function** (it currently ends with the
`return {"displayName": ..., "jobTitle": user.job_title}` block — including the
AI's version if you already applied it) and **replace it with:**

```python
# ---------------------------------------------------------------------------
# Function: get_user_profile
# ---------------------------------------------------------------------------
async def get_user_profile() -> dict:
    """
    Fetch the signed-in user's profile AND their direct manager.

        GET /me?$select=...   -> the user's own profile
        GET /me/manager       -> the direct manager (a directoryObject)

    Both calls reuse the same authenticated client, so this costs one extra
    round trip and no extra permission — User.Read already covers /me/manager.

    Two traps handled here:
      * $select is REQUIRED for `department`. GET /me returns Graph's default
        property set, and department is not in it — without the select it is
        silently always None.
      * A user with no manager configured makes /me/manager return 404, not an
        empty object. That is normal, not an error: we log it and return None
        for the manager fields rather than failing the whole profile call.

    Returns:
        dict with displayName, mail, userPrincipalName, jobTitle, department,
        managerName, managerEmail, managerJobTitle.
    """
    logger.info("Fetching user profile from Graph API (/me + /me/manager)")

    client = get_graph_client()

    # ── Step 1: own profile, with an explicit $select ──────────────────────
    # Guarded: constructing an SDK query-parameter class is the exact move that
    # caused the L-02 crash (EventsRequestBuilderPostQueryParameters did not
    # exist in this SDK version). If the class shape differs here, fall back to
    # the plain call and accept that `department` will be None.
    user = None
    try:
        from kiota_abstractions.base_request_configuration import RequestConfiguration
        from msgraph.generated.users.item.user_item_request_builder import (
            UserItemRequestBuilder,
        )

        query_params = UserItemRequestBuilder.UserItemRequestBuilderGetQueryParameters(
            select=[
                "displayName",
                "mail",
                "userPrincipalName",
                "jobTitle",
                "department",
                "officeLocation",
            ]
        )
        user = await client.me.get(
            request_configuration=RequestConfiguration(query_parameters=query_params)
        )
    except Exception as select_err:
        logger.warning(
            "GET /me with $select failed (%s: %s) - falling back to the plain "
            "call. 'department' will be None.",
            type(select_err).__name__,
            select_err,
        )
        user = await client.me.get()

    # ── Step 2: manager, graceful on 404 ──────────────────────────────────
    manager_name = None
    manager_email = None
    manager_job_title = None

    try:
        manager_obj = await client.me.manager.get()
        if manager_obj is not None:
            # Graph types this as directoryObject; the SDK upcasts it to User
            # when @odata.type says so. getattr keeps this safe either way.
            manager_name = getattr(manager_obj, "display_name", None)
            manager_email = getattr(manager_obj, "mail", None)
            manager_job_title = getattr(manager_obj, "job_title", None)
    except Exception as manager_err:
        # Expected whenever the Manager attribute is unset in Entra ID: Graph
        # answers 404 ResourceNotFound rather than returning an empty object.
        logger.info(
            "No manager returned for the signed-in user (%s: %s)",
            type(manager_err).__name__,
            manager_err,
        )

    return {
        "displayName": user.display_name,
        "mail": user.mail,
        "userPrincipalName": user.user_principal_name,
        "jobTitle": user.job_title,
        "department": getattr(user, "department", None),
        "managerName": manager_name,
        "managerEmail": manager_email,
        "managerJobTitle": manager_job_title,
    }
```

### 5.5 Replacement 2 — `tools/profile_tools.py`

**Replace the entire `get_my_profile` tool function** (everything from
`@mcp.tool` down to the `format_tool_error` line) **with:**

```python
@mcp.tool
async def get_my_profile() -> dict:
    """
    Get the identity of the currently logged-in Microsoft 365 user, including
    their direct manager's name and email address.

    Use this tool when the user asks things like:
    - "Who am I logged in as?"
    - "What's my email address?"
    - "Who is my manager?"
    - "Who do I report to?"
    - "Confirm my account details"

    This tool takes no parameters — it always returns the identity of
    whoever is currently authenticated in this conversation.

    Returns:
        A dictionary containing:
            - display_name (str): The user's full name as set in Microsoft 365
            - email (str): The user's primary email address
            - job_title (str): The user's job title, if set
            - department (str): The user's department, if set
            - manager (dict | None): name / email / job_title, or None if the
              Manager attribute is not configured in the directory
            - display_table (str): render this verbatim
    """
    logger.info("Tool called: get_my_profile")

    try:
        profile_data = await get_user_profile()

        # NOTE: .get(key, "—") does NOT work here. These keys are always
        # present and set to None, so dict.get() returns None, never the
        # default — the default only fires when the key is ABSENT. Every
        # fallback below must therefore be `or`, not a second .get() argument.
        display_name = profile_data.get("displayName") or "—"
        email = (
            profile_data.get("mail")
            or profile_data.get("userPrincipalName")
            or "—"
        )
        job_title = profile_data.get("jobTitle") or "—"
        department = profile_data.get("department") or "—"

        manager_name = profile_data.get("managerName")
        manager_email = profile_data.get("managerEmail")
        manager_job_title = profile_data.get("managerJobTitle")

        manager_section = None
        if manager_name or manager_email:
            manager_section = {
                "name": manager_name or "—",
                "email": manager_email or "—",
                "job_title": manager_job_title or "—",
            }

        # House pattern: the rendered block goes in display_table, and
        # instruction tells the model what to do with it. Do not merge them.
        table = (
            "**👤 My Profile**\n\n"
            "| Field | Details |\n"
            "|---|---|\n"
            f"| **Name** | {display_name} |\n"
            f"| **Email** | {email} |\n"
            f"| **Job Title** | {job_title} |\n"
            f"| **Department** | {department} |\n"
        )

        if manager_section:
            table += (
                "\n**👔 Manager**\n\n"
                "| Field | Details |\n"
                "|---|---|\n"
                f"| **Name** | {manager_section['name']} |\n"
                f"| **Email** | {manager_section['email']} |\n"
                f"| **Job Title** | {manager_section['job_title']} |\n"
            )
        else:
            table += (
                "\n_No manager is configured for this account in the directory._\n"
            )

        return {
            "display_name": profile_data.get("displayName"),
            "email": profile_data.get("mail") or profile_data.get("userPrincipalName"),
            "job_title": profile_data.get("jobTitle"),
            "department": profile_data.get("department"),
            "manager": manager_section,
            "display_table": table,
            "instruction": (
                "Render `display_table` EXACTLY as provided. "
                "If `manager` is null, the Manager attribute is not set for this "
                "account in the directory — say exactly that. Do NOT guess who "
                "the manager is, do NOT infer it from email signatures, job "
                "titles or org names, and do NOT state or imply that the user "
                "is their own manager or has no manager as a fact about the "
                "organisation. The only true statement is that the directory "
                "does not record one."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="get_my_profile", logger=logger)
```

> **Why that last instruction is worded so hard:** you reported *"AI thinks that
> I'm the manager."* That is the model filling a gap with a guess — the same
> class of failure as F-27, where it substituted To-Do data for Planner. The
> governance layer's rule is: state the absence, never invent the value.

---

## 6. The Group Object ID — where it goes, and why not yet

**You do not need to update it anywhere right now.**

- The **group ID** is only used by FEAT-01 **§10**, the optional group-wide
  Planner extension — the one you have (correctly) not added. The main build
  uses `/me/planner/tasks`, which needs no group ID at all.
- The **group name** is never needed anywhere. Plan titles come back from Graph.

When you *do* add §10, the ID goes in **exactly one place**: the `PLANNER_GROUP_ID`
key in `.env` (the commented line you added in §4 above). Uncomment it and paste
the real GUID:

```bash
PLANNER_GROUP_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Never hardcode it in a `.py` file.

⚠️ **Before you paste it, check it is a valid GUID.** The sample you gave —
`95111a38-36v5-4119-b162-716a3366145f` — contains the letter **`v`**, which is
not a hexadecimal digit. Graph rejects that with HTTP 400 *before* it even checks
permissions, so it looks like a permission failure and is not one. A valid GUID
uses only `0-9` and `a-f`. Get the real one from
*Entra ID → Groups → your group → Overview → Object ID* and copy it with the
clipboard rather than retyping.

**The one-liner you tried will not work as written** for the same reason
`diag_tasks.py` didn't: `python -c` has no token. Use this instead, after §3 and
§4 are done:

```bash
cd /opt/FiGPT_OutlookMCP
source .venv/bin/activate
python - <<'PY'
import asyncio, json, sys
from pathlib import Path
sys.path.insert(0, str(Path.cwd()))
tok = json.loads(Path("bin/token.json").read_text())["access_token"]
import graph.graph_client_factory as factory
factory.get_current_access_token = lambda: tok
from graph.graph_client_factory import get_graph_client

GROUP_ID = "PASTE-THE-REAL-GUID-HERE"

async def m():
    c = get_graph_client()
    r = await c.groups.by_group_id(GROUP_ID).planner.plans.get()
    for p in (r.value or []):
        print(p.id, "-", p.title)

asyncio.run(m())
PY
```

---

## 7. Order of work + final verification

Do these in order. Do not skip ahead — step 4 depends on step 3's token.

| # | Do | Verify |
|---|---|---|
| 1 | §1 — `utils/task_grouping.py` (2 edits) | `python -m pytest tests/test_task_grouping.py -q` → all pass |
| 2 | §2 — `tools/task_tools.py` (1 sed + 4 edits) | `python -m pytest tests/ -q` → **112 passed** |
| 3 | §3 — replace `bin/diag_tasks.py` | — |
| 4 | §4 — `.env` scopes, then `python bin/get_token.py` | `python bin/diag_tasks.py` → step 0 shows `Tasks.ReadWrite` |
| 5 | §4 — run the diagnostic | steps 1–4 **OK**; send me the full output |
| 6 | §5 — the two manager replacements | `python -c "import ast;[ast.parse(open(f).read()) for f in ['graph/graph_client_factory.py','tools/profile_tools.py']]"` → silent |
| 7 | restart the server | `curl -s localhost:8000/health` → `tools_registered: 38` |

**Expected tool count after this: 38** (36 + `list_planner_tasks` + `list_todo_tasks`).

**Then send me back:**
1. The full `python bin/diag_tasks.py` output — especially **step 4's `due = ...`
   line**, which confirms the To-Do `dateTimeTimeZone` shape, and **step 5**, which
   settles the manager question definitively.
2. `python -m pytest tests/ -q` (last line only).
3. `curl -s localhost:8000/health`.

Do **not** commit yet. Once the diagnostic passes we commit FEAT-01 as one
change, separately from the `.gitattributes` / CRLF commit (F-29).

---

## 8. Ledger impact

| ID | Change |
|---|---|
| B-01 | ✅ **UNBLOCKED 2026-08-05** — `Tasks.ReadWrite`, `Tasks.ReadWrite.Shared`, `Group.Read.All`, `User.Read.All` all granted with admin consent. Remaining half is the token re-issue (§4) |
| F-27 | 🟡 code applied, tests green after FIX-05 §1–§2; closes on TEST-01 T9.4/T9.5 |
| F-07 | ✅ confirmed landed — `git status` shows `.venv/` untracked, branch 1 commit ahead |
| **F-30** *(new, Low)* | `get_my_profile` reported no manager and the model inferred the user *was* the manager. Directory data gap (Manager attribute unset in Entra ID) plus a `.get(k, default)` misuse that renders `None` instead of `—`. Both fixed in §5; the directory half needs an admin |
