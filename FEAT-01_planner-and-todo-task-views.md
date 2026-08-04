# FEAT-01 — Planner & Microsoft To-Do task views

**Target:** `/opt/FiGPT_OutlookMCP` on FGSV2187
**Written:** 2026-08-04
**Tool count:** 36 → **38**
**Closes:** F-27 (structurally, not by instruction-tweak)
**Blocked by:** B-01 until the Azure permission lands — **read §2 before writing any code**

---

## 1. What we are building, and the two decisions behind it

Two new MCP tools:

| Tool | Reads | Never reads |
|---|---|---|
| `list_planner_tasks` | Microsoft Planner (`/me/planner/tasks`) | To-Do |
| `list_todo_tasks` | Microsoft To-Do (`/me/todo/lists` + tasks) | Planner |

Both return the same three-tier structure:

```
🔴 Overdue                      — flat, ordered by due date ASC (most overdue first)
🟠 Due in the next 14 days      — sub-grouped by Plan / List name, due date ASC
⚪ No due date / due later      — sub-grouped by Plan / List name, created date DESC
```

### Decision 1 — two tools, not one tool with a `source` parameter

There is already a `list_tasks(source=...)` tool, and it is the cause of **F-27**: asked
explicitly for Planner, the model called it, got "Planner unavailable", then silently
re-called it for To-Do and presented that as the answer.

That is not a prompt problem. **A single tool with a source switch makes substitution a
legal move** — the model can satisfy "call the task tool" while changing which system it
read. Two separately-named tools make it structurally impossible: `list_planner_tasks`
cannot return To-Do data, so there is nothing to substitute.

This is the same move that fixed F-17 — splitting `create_draft_invite` into
`compose_invite` / `save_invite` (D-15). Splitting fixed the behaviour that hardened
docstrings had not.

> **`list_tasks` is left in place for now.** Retiring it is §11, to be done *after* the new
> tools are verified — not in the same change. One task at a time (D-16).

### Decision 2 — grouping logic lives in `utils/`, not in the tool

`utils/task_grouping.py` is pure functions: no Graph calls, no I/O. The ordering rules are
the part most likely to be subtly wrong, and they become directly unit-testable without
mocking Graph. Mocking an HTTP client to test a sort is wasted effort, and TEST-01 already
proved that unit tests which don't exercise the real shaping code miss real bugs.

---

## 2. Pre-requisites — do these first

### 2.1 ⛔ Azure AD permission (this is B-01, and it blocks everything)

Planner has returned **403** since 20 July. Nothing in this document works until that is
resolved. **Do not write the code first and hope.**

**Delegated permissions to request, with admin consent:**

| Permission | Needed for | Status |
|---|---|---|
| `Tasks.Read` | `GET /me/planner/tasks`, `GET /planner/plans/{id}`, all To-Do reads | ⛔ request this |
| `Tasks.ReadWrite` | supersedes `Tasks.Read`; also covers the existing `add_task_*` tools | ⛔ request this instead, it covers both |
| `Tasks.ReadWrite.Shared` | To-Do lists **shared with** the user, not owned by them | 🟡 only if you need shared lists |
| `Group.Read.All` | **probably not needed** — see below | 🟢 drop from the ask |

**Narrow the B-01 ask.** The ledger records B-01 as needing
`Tasks.ReadWrite` + `Tasks.ReadWrite.Shared` + `Group.Read.All`. For *reads*,
`Group.Read.All` is almost certainly unnecessary: `/me/planner/tasks` and
`/planner/plans/{id}` are both gated on `Tasks.Read`, and access to a specific plan is
decided by **your group membership**, not by a directory-read scope. `Group.Read.All` is a
broad, tenant-wide, admin-consent-heavy permission — asking for it makes the approval
harder to get.

> **Go back to the Azure admin asking only for `Tasks.ReadWrite` (delegated, admin
> consent).** If, and only if, plan-title resolution still 403s afterwards, escalate to add
> `Group.Read.All`. A smaller ask approved this week beats a bigger ask approved never.

### 2.2 ⚠️ The token must be re-issued after the grant

This catches people out. **Granting a permission in Azure does not upgrade tokens that
already exist.** LibreChat is holding an access token minted under the old scope set; it
will keep 403-ing after the admin clicks Grant.

After the grant:

1. Sign out of LibreChat completely.
2. Sign back in so the OAuth flow re-runs and re-consents.
3. Confirm the new scope is actually present — decode the token at
   [jwt.ms](https://jwt.ms) and look at the `scp` claim for `Tasks.ReadWrite`.

If `scp` does not list it, the grant did not apply to this app registration and it is still
an admin-side problem, not a code problem.

### 2.3 ⚠️ Your Group Object ID is not a valid GUID

You gave:

```
95111a38-36v5-4119-b162-716a3366145f
             ^
```

GUIDs are hexadecimal — `0-9` and `a-f` only. **`v` is not a hex digit**, so this string
will be rejected by Graph with a `400 Bad Request` before it ever reaches a permission
check. It is a transcription slip in the second segment (`36v5`).

Please re-copy it from the Teams admin centre or from Azure AD → Groups → F_FiGPT →
Overview → Object ID. You only need it for the optional group-wide extension in §10 — the
main build in §4–§8 does not use it at all, so this does not block you.

### 2.4 ⚠️ Scoping — `/me/planner/tasks` is narrower than it sounds

You specified `https://graph.microsoft.com/v1.0/me/planner/tasks`. That endpoint returns
**tasks assigned to you**, across every plan you can see.

It does **not** return:

- tasks in the F_FiGPT plans that are assigned to somebody else
- tasks in those plans that are assigned to nobody

Given that you supplied a Group Object ID and mentioned the admin account's Teams group, I
suspect you may eventually want *"everything in the F_FiGPT plans"*, which is a different
call chain (`/groups/{id}/planner/plans` → `/planner/plans/{id}/tasks`).

**I have built what you asked for** — `/me/planner/tasks` — as the main path, and written
the group-wide variant as an optional, self-contained extension in §10. Add it only if the
`/me` view turns out to be too narrow in practice. Start with `/me`; it needs no group ID
and no extra permission.

### 2.5 ✅ No new Python packages

`python-dateutil` is already in use (`graph/mail_client.py` imports it), and `zoneinfo` is
stdlib on Python 3.12. `requirements.txt` does not change.

---

## 3. Step 0 — Prove Graph works before writing a line of code

This project has burned three sessions on code written against an endpoint that turned out
to be broken (L-01) or an SDK class that did not exist (L-02). `bin/diag_list_email.py` is
what finally closed L-01. Same approach here.

**Create `bin/diag_tasks.py`:**

```python
"""
Throwaway diagnostic. Proves the Graph endpoints and SDK paths behind FEAT-01
work against the live tenant BEFORE any tool code is written.

Run:  python bin/diag_tasks.py
"""
import asyncio
import sys

from graph.graph_client_factory import get_graph_client


async def main() -> int:
    client = get_graph_client()
    failures = 0

    # ---- 1. Planner: tasks assigned to me -------------------------------
    print("=" * 70)
    print("1. GET /me/planner/tasks")
    try:
        resp = await client.me.planner.tasks.get()
        tasks = resp.value if resp and resp.value else []
        print(f"   OK — {len(tasks)} task(s)")
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
        print(f"   FAILED — {type(exc).__name__}: {exc}")

    # ---- 2. Planner: resolve a plan title -------------------------------
    print("=" * 70)
    print("2. GET /planner/plans/{id}   (needed for 'Project Name' grouping)")
    if not plan_ids:
        print("   SKIPPED — no plan ids from step 1")
    else:
        try:
            plan = await client.planner.plans.by_planner_plan_id(plan_ids[0]).get()
            print(f"   OK — plan title = {plan.title!r}")
        except Exception as exc:
            failures += 1
            print(f"   FAILED — {type(exc).__name__}: {exc}")

    # ---- 3. To-Do: lists ------------------------------------------------
    print("=" * 70)
    print("3. GET /me/todo/lists")
    lists = []
    try:
        resp = await client.me.todo.lists.get()
        lists = resp.value if resp and resp.value else []
        print(f"   OK — {len(lists)} list(s)")
        for l in lists:
            print(f"     - {l.display_name!r}  (wellknown={l.wellknown_list_name})")
    except Exception as exc:
        failures += 1
        print(f"   FAILED — {type(exc).__name__}: {exc}")

    # ---- 4. To-Do: tasks in the first list ------------------------------
    print("=" * 70)
    print("4. GET /me/todo/lists/{id}/tasks")
    if not lists:
        print("   SKIPPED — no lists from step 3")
    else:
        try:
            builder = client.me.todo.lists.by_todo_task_list_id(lists[0].id).tasks
            resp = await builder.get()
            items = resp.value if resp and resp.value else []
            print(f"   OK — {len(items)} task(s) in {lists[0].display_name!r}")
            for t in items[:3]:
                print(f"     - {t.title!r}")
                print(f"       status   = {t.status!r}")
                print(f"       due      = {t.due_date_time!r}   <-- SHAPE MATTERS")
                print(f"       created  = {t.created_date_time!r}")
            print(f"   has with_url(): {hasattr(builder, 'with_url')}   <-- paging support")
            print(f"   next link     : {getattr(resp, 'odata_next_link', None)!r}")
        except Exception as exc:
            failures += 1
            print(f"   FAILED — {type(exc).__name__}: {exc}")

    print("=" * 70)
    print(f"{'ALL PASSED' if failures == 0 else str(failures) + ' STEP(S) FAILED'}")
    return failures


if __name__ == "__main__":
    sys.exit(asyncio.run(main()))
```

**Run it:**

```bash
cd /opt/FiGPT_OutlookMCP
source .venv/bin/activate
python bin/diag_tasks.py
```

### What the output tells you

| Result | Meaning | Action |
|---|---|---|
| All four pass | Permissions and SDK paths are good | Proceed to §4 |
| Step 1 gives **403 / accessDenied** | B-01 is still live | **Stop.** Go back to §2.1 — no code will help |
| Step 1 passes, step 2 403s | `Tasks.Read` granted but plan access denied | Plan titles will show `(plan name unavailable)`. Feature still works. Consider `Group.Read.All` |
| Step 2 gives `AttributeError` | SDK fluent path differs in this version | Send me the traceback — this is the L-02 failure mode |
| Step 4 prints `has with_url(): False` | No paging helper in this SDK build | Tell me; §5 has a guard but I'll switch approach |

**Send me the full output of step 4's `due = ...` line.** To-Do returns due dates as a
`dateTimeTimeZone` object, not a datetime, and confirming its exact shape saves a debugging
round. This is the same class of trap as F-08's tz-aware/naive comparison.

---

## 4. Step 1 — Reconcile helper names (6 greps, 2 minutes)

The code below uses six existing symbols. I do not have repo access, so **verify the names
match before pasting.** If any differ, change only the import line — nothing else in the
code depends on the spelling.

```bash
cd /opt/FiGPT_OutlookMCP
grep -n "def get_graph_client"  graph/graph_client_factory.py
grep -n "^def \|^[A-Z_]\+ =" utils/error_handler.py
grep -n "^def \|^[A-Z_]\+ =" utils/governance.py
grep -n "^def \|^[A-Z_]\+ =" utils/table_utils.py
grep -n "^def \|^[A-Z_]\+ =" utils/audit_logger.py
sed -n '1,40p' tools/task_tools.py
```

| Used below | Expected source | If different |
|---|---|---|
| `get_graph_client()` | `graph/graph_client_factory.py` | adjust import |
| `format_tool_error(...)` | `utils/error_handler.py` | adjust import + call |
| `get_governance_rules()` | `utils/governance.py` | adjust — may be a constant, not a function |
| `_table_cell(...)` | `utils/table_utils.py` | handled by a try/except import already |
| `log_tool_call(...)` | `utils/audit_logger.py` | adjust, or delete the audit lines |
| `mcp` | `tools/mcp_instance.py` | already imported at the top of `task_tools.py` |

The last `sed` shows you the existing import block in `task_tools.py` — **copy the
governance and audit imports verbatim from there** rather than trusting my guesses.

---

## 5. Step 2 — NEW FILE: `utils/task_grouping.py`

Create this file. It is new, so there is nothing to merge.

```python
"""
Grouping and ordering engine shared by the Planner and To-Do task tools.

Pure functions only — no Graph calls, no I/O, no SDK imports. Everything here
is directly unit-testable, which is the point: the ordering rules are the part
most likely to be subtly wrong, and mocking an HTTP client to test a sort is
wasted effort.

Normalised task shape consumed by this module:

    {
        "id":        str,
        "title":     str,
        "container": str,              # Plan title (Planner) / List name (To-Do)
        "due":       datetime | None,  # timezone-aware, UTC
        "created":   datetime | None,  # timezone-aware, UTC
        "status":    str,
        "extra":     str,              # Priority (Planner) / Importance (To-Do)
        "completed": bool,
        "source":    str,              # "planner" | "todo"
    }
"""

from __future__ import annotations

import os
import re
from datetime import datetime, timedelta, timezone

from dateutil import parser as _dateparser

try:
    from zoneinfo import ZoneInfo
except ImportError:  # pragma: no cover - stdlib on 3.9+
    ZoneInfo = None

# utils/table_utils.py exposes the cell sanitiser that closed F-12/F-15.
# Accept either spelling so this module does not care which one landed.
try:
    from utils.table_utils import _table_cell as table_cell
except ImportError:  # pragma: no cover
    from utils.table_utils import table_cell


# Horizon for the "due soon" group. Env-configurable so the two-week window can
# be changed without a code edit or a redeploy.
HORIZON_DAYS = int(os.getenv("TASK_HORIZON_DAYS", "14"))

GROUP_OVERDUE = "Overdue"
GROUP_UPCOMING = f"Due in the next {HORIZON_DAYS} days"
GROUP_REST = "No due date / due later"

# Sort floor for tasks with no created date, so the sort key is never None.
_EPOCH = datetime(1970, 1, 1, tzinfo=timezone.utc)


def utc_now() -> datetime:
    """
    Current time, timezone-aware, UTC.

    F-26: datetime.utcnow() is deprecated in 3.12 AND returns a naive value,
    which is exactly what produced the F-08 comparison TypeError. Never use it.
    """
    return datetime.now(timezone.utc)


def to_utc(value, default_tz: str = "UTC") -> datetime | None:
    """
    Coerce whatever Graph hands us into a timezone-aware UTC datetime.

    Handles the three shapes that actually turn up:
      * datetime          — Planner dueDateTime / createdDateTime via the SDK
      * str               — raw ISO 8601, including Graph's 7-digit fractions
      * dateTimeTimeZone  — To-Do dueDateTime: .date_time (str) + .time_zone (str)

    Returns None for anything unparseable rather than raising. A task with a
    malformed date should drop into the "no due date" group, not kill the
    entire listing — one bad row must not cost the user all the others.
    """
    if value is None:
        return None

    # dateTimeTimeZone object (To-Do). Duck-typed, so tests can pass a stub.
    raw_tz = default_tz
    if hasattr(value, "date_time"):
        raw_tz = getattr(value, "time_zone", None) or default_tz
        value = value.date_time
        if not value:
            return None

    if isinstance(value, datetime):
        dt = value
    else:
        text = str(value)
        # Graph emits 7 fractional digits ("...T00:00:00.0000000") and dateutil's
        # parser rejects more than 6. Sub-second precision is meaningless on a
        # due date, so drop the fraction entirely rather than truncating it.
        text = re.sub(r"\.\d+", "", text)
        try:
            dt = _dateparser.parse(text)
        except (ValueError, OverflowError, TypeError):
            return None
        if dt is None:
            return None

    if dt.tzinfo is None:
        tzinfo = timezone.utc
        if ZoneInfo is not None and raw_tz and raw_tz.upper() != "UTC":
            try:
                tzinfo = ZoneInfo(raw_tz)
            except Exception:
                # Graph sometimes returns Windows zone names ("W. Europe Standard
                # Time") that zoneinfo does not recognise. UTC is Graph's documented
                # default, so fall back rather than fail the whole request.
                tzinfo = timezone.utc
        dt = dt.replace(tzinfo=tzinfo)

    return dt.astimezone(timezone.utc)


def _by_container(tasks: list[dict], key, reverse: bool) -> dict[str, list[dict]]:
    """
    Sub-group tasks by container name and sort within each group.

    The containers themselves are ordered by their own leading task, so the most
    urgent plan/list floats to the top of the section rather than appearing in
    whatever order Graph happened to return.
    """
    buckets: dict[str, list[dict]] = {}
    for task in tasks:
        name = task.get("container") or "(none)"
        buckets.setdefault(name, []).append(task)

    for items in buckets.values():
        items.sort(key=key, reverse=reverse)

    return dict(sorted(buckets.items(), key=lambda kv: key(kv[1][0]), reverse=reverse))


def group_tasks(tasks: list[dict], *, now=None, horizon_days=None) -> dict:
    """
    Apply the agreed grouping and ordering.

      Group 1  Overdue                 — flat, due date ASC (most overdue first)
      Group 2  Due in the next N days  — sub-grouped by container, due date ASC
      Group 3  Everything else         — sub-grouped by container, created DESC

    Group 3 deliberately holds BOTH "no due date" and "due beyond the horizon":
    from the user's point of view neither needs attention this fortnight.

    Ordering note: group 3 is newest-created first. If you would rather surface
    stale backlog items, change `reverse=True` to `reverse=False` on the "rest"
    line below — that single flag is the whole change.
    """
    now = now or utc_now()
    days = HORIZON_DAYS if horizon_days is None else horizon_days
    horizon = now + timedelta(days=days)

    overdue: list[dict] = []
    upcoming: list[dict] = []
    rest: list[dict] = []

    for task in tasks:
        due = task.get("due")
        if due is None or due > horizon:
            rest.append(task)
        elif due < now:
            overdue.append(task)
        else:
            upcoming.append(task)

    overdue.sort(key=lambda t: t["due"])

    grouped = {
        "overdue": overdue,
        "upcoming": _by_container(upcoming, key=lambda t: t["due"], reverse=False),
        "rest": _by_container(rest, key=lambda t: t.get("created") or _EPOCH, reverse=True),
    }
    grouped["counts"] = {
        "overdue": len(overdue),
        "upcoming": len(upcoming),
        "rest": len(rest),
        "total": len(overdue) + len(upcoming) + len(rest),
    }
    return grouped


def _due_cell(due, now) -> str:
    """Absolute date plus a relative hint — 'in 3d' is what the user actually reads."""
    if due is None:
        return "—"
    stamp = due.strftime("%d %b %Y")
    delta = (due.date() - now.date()).days
    if delta < 0:
        return f"{stamp} ({abs(delta)}d ago)"
    if delta == 0:
        return f"{stamp} (today)"
    if delta == 1:
        return f"{stamp} (tomorrow)"
    return f"{stamp} (in {delta}d)"


def _table(tasks, container_label, extra_label, now) -> list[str]:
    """
    Build one markdown table.

    Every cell goes through table_cell(): task titles routinely contain pipes
    and line breaks, and a markdown table is line-based, so an unescaped title
    splits the row and mangles everything below it. That was F-12/F-15.
    """
    lines = [
        f"| Due | Task | {container_label} | Status | {extra_label} |",
        "|---|---|---|---|---|",
    ]
    for t in tasks:
        lines.append(
            "| {due} | {title} | {container} | {status} | {extra} |".format(
                due=table_cell(_due_cell(t.get("due"), now)),
                title=table_cell(t.get("title", "")),
                container=table_cell(t.get("container", "")),
                status=table_cell(t.get("status", "")),
                extra=table_cell(t.get("extra", "")),
            )
        )
    return lines


def render_markdown(
    grouped: dict,
    *,
    container_label: str = "Plan",
    extra_label: str = "Priority",
    now=None,
) -> str:
    """Render the grouped structure as the markdown block the model must echo verbatim."""
    now = now or utc_now()
    lines: list[str] = []

    overdue = grouped["overdue"]
    lines.append(f"### 🔴 {GROUP_OVERDUE} — {len(overdue)}")
    lines.append("")
    if overdue:
        lines.extend(_table(overdue, container_label, extra_label, now))
    else:
        lines.append("_Nothing overdue._")
    lines.append("")

    upcoming = grouped["upcoming"]
    total_upcoming = sum(len(v) for v in upcoming.values())
    lines.append(f"### 🟠 {GROUP_UPCOMING} — {total_upcoming}")
    if not upcoming:
        lines.append("")
        lines.append("_Nothing due in this window._")
    for name, items in upcoming.items():
        lines.append("")
        lines.append(f"**{name}** — {len(items)}")
        lines.extend(_table(items, container_label, extra_label, now))
    lines.append("")

    rest = grouped["rest"]
    total_rest = sum(len(v) for v in rest.values())
    lines.append(f"### ⚪ {GROUP_REST} — {total_rest}")
    if not rest:
        lines.append("")
        lines.append("_Nothing else outstanding._")
    for name, items in rest.items():
        lines.append("")
        lines.append(f"**{name}** — {len(items)}")
        lines.extend(_table(items, container_label, extra_label, now))

    return "\n".join(lines)
```

---

## 6. Step 3 — Extend `graph/task_client.py`

**Append** these to the existing file. Do not remove anything already there — the existing
`add_task_todo` / `add_task_planner` paths stay as they are.

First, make sure these imports are present at the top of the file:

```python
import asyncio

from utils.task_grouping import to_utc
```

Then append:

```python
# ---------------------------------------------------------------------------
# FEAT-01 — read paths for the Planner and To-Do task views
# ---------------------------------------------------------------------------

# Plan titles are resolved one HTTP call at a time and effectively never change,
# so they are cached for the life of the process. A plan RENAME therefore needs a
# server restart to show up — an accepted trade for not making N extra Graph calls
# on every single listing. This is a plain dict, not state: D-06 (stateless server)
# is about persistence, and losing this on restart costs nothing but a re-fetch.
_PLAN_TITLE_CACHE: dict[str, str] = {}


def _enum_value(value):
    """Kiota enums expose .value; plain strings and None pass through unharmed."""
    if value is None:
        return None
    return getattr(value, "value", str(value))


def _planner_status(percent) -> str:
    """Planner has no status field — it infers state from percentComplete (0/50/100)."""
    pct = percent or 0
    if pct >= 100:
        return "Completed"
    if pct > 0:
        return "In progress"
    return "Not started"


def _planner_priority(value) -> str:
    """Planner priority is an int 0-10. Its own UI buckets it into four labels."""
    if value is None:
        return "—"
    if value <= 1:
        return "Urgent"
    if value <= 3:
        return "Important"
    if value <= 5:
        return "Medium"
    return "Low"


async def fetch_plan_titles(plan_ids) -> dict:
    """
    Resolve Planner plan IDs to their titles — this is the 'Project Name' grouping.

    /me/planner/tasks returns planId only, never the plan's name, so one extra
    call per distinct plan is unavoidable. They run concurrently; a typical user
    sees fewer than ten distinct plans.

    A plan that cannot be read degrades to a placeholder rather than raising:
    a permission gap on one plan must not lose the user every other task.
    """
    client = get_graph_client()
    missing = [p for p in plan_ids if p and p not in _PLAN_TITLE_CACHE]

    async def _one(plan_id):
        try:
            plan = await client.planner.plans.by_planner_plan_id(plan_id).get()
            return plan_id, (plan.title if plan and plan.title else "(untitled plan)")
        except Exception as exc:
            logger.warning("Could not resolve Planner plan %s: %s", plan_id, exc)
            return plan_id, "(plan name unavailable)"

    if missing:
        for plan_id, title in await asyncio.gather(*(_one(p) for p in missing)):
            _PLAN_TITLE_CACHE[plan_id] = title

    return dict(_PLAN_TITLE_CACHE)


async def fetch_planner_tasks(include_completed: bool = False) -> list[dict]:
    """
    GET /me/planner/tasks — Planner tasks ASSIGNED TO the signed-in user.

    Note the scope: this does not return unassigned tasks, nor tasks assigned to
    other people, even in plans the user can see. See FEAT-01 §2.4.
    """
    client = get_graph_client()
    response = await client.me.planner.tasks.get()
    raw = response.value if response and response.value else []

    titles = await fetch_plan_titles({t.plan_id for t in raw if t.plan_id})

    tasks = []
    for t in raw:
        completed = (t.percent_complete or 0) >= 100
        if completed and not include_completed:
            continue
        tasks.append({
            "id": t.id,
            "title": t.title or "(untitled task)",
            "container": titles.get(t.plan_id, "(unknown plan)"),
            "due": to_utc(t.due_date_time),
            "created": to_utc(t.created_date_time),
            "status": _planner_status(t.percent_complete),
            "extra": _planner_priority(t.priority),
            "completed": completed,
            "source": "planner",
        })
    return tasks


async def fetch_todo_lists() -> list[dict]:
    """GET /me/todo/lists — every To-Do list the user owns or has shared to them."""
    client = get_graph_client()
    response = await client.me.todo.lists.get()
    lists = response.value if response and response.value else []
    return [
        {"id": l.id, "name": l.display_name or "(untitled list)"}
        for l in lists
        if l and l.id
    ]


async def fetch_todo_tasks(
    list_id: str,
    list_name: str,
    include_completed: bool = False,
    max_pages: int = 10,
) -> list[dict]:
    """
    GET /me/todo/lists/{id}/tasks, following @odata.nextLink.

    max_pages is a deliberate backstop, not a real limit: at Graph's default page
    size ten pages is a very large list, and an unbounded loop against a paging
    bug would hammer Graph until the token throttles. Rate limiting is still
    missing server-wide (§5b of PROJECT_STATE), so loops guard themselves.
    """
    client = get_graph_client()
    builder = client.me.todo.lists.by_todo_task_list_id(list_id).tasks
    response = await builder.get()

    tasks: list[dict] = []
    pages = 0

    while response is not None:
        for t in (response.value or []):
            status = _enum_value(t.status) or "notStarted"
            completed = status == "completed"
            if completed and not include_completed:
                continue
            tasks.append({
                "id": t.id,
                "title": t.title or "(untitled task)",
                "container": list_name,
                # To-Do returns dueDateTime as a dateTimeTimeZone OBJECT, not a
                # datetime. to_utc() handles it; passing it to a comparison raw
                # is the F-08 failure mode.
                "due": to_utc(t.due_date_time),
                "created": to_utc(t.created_date_time),
                "status": _TODO_STATUS_LABELS.get(status, status),
                "extra": (_enum_value(t.importance) or "normal").capitalize(),
                "completed": completed,
                "source": "todo",
            })

        pages += 1
        next_link = getattr(response, "odata_next_link", None)
        if not next_link or pages >= max_pages:
            break
        if not hasattr(builder, "with_url"):
            logger.warning(
                "SDK build has no with_url(); To-Do paging stopped at page %s for list %r",
                pages, list_name,
            )
            break
        response = await builder.with_url(next_link).get()

    return tasks


_TODO_STATUS_LABELS = {
    "notStarted": "Not started",
    "inProgress": "In progress",
    "completed": "Completed",
    "waitingOnOthers": "Waiting on others",
    "deferred": "Deferred",
}


async def fetch_all_todo_tasks(include_completed: bool = False):
    """
    Every task across every To-Do list, plus the list inventory itself.

    Returns (tasks, lists). The list inventory is returned separately so the tool
    can honestly report "4 lists scanned" even when three of them are empty —
    an empty result and a failed result must not look the same to the user.

    A list that fails is logged and skipped, not fatal: one inaccessible shared
    list must not cost the user every other list.
    """
    lists = await fetch_todo_lists()
    if not lists:
        return [], []

    results = await asyncio.gather(
        *(fetch_todo_tasks(l["id"], l["name"], include_completed) for l in lists),
        return_exceptions=True,
    )

    tasks: list[dict] = []
    for meta, result in zip(lists, results):
        if isinstance(result, Exception):
            logger.warning("To-Do list %r could not be read: %s", meta["name"], result)
            continue
        tasks.extend(result)

    return tasks, lists
```

> **Check the logger name.** I used `logger`. If `task_client.py` calls it `log` or
> `_log`, change these four references. `grep -n "logger\|^log = " graph/task_client.py`.

---

## 7. Step 4 — Extend `tools/task_tools.py`

**Append** to the existing file. `list_tasks` and the `add_task_*` tools stay untouched.

Add to the imports at the top:

```python
from graph.task_client import fetch_planner_tasks, fetch_all_todo_tasks
from utils.task_grouping import group_tasks, render_markdown, utc_now
```

Then append:

```python
# ---------------------------------------------------------------------------
# FEAT-01 — Planner and To-Do task views
#
# Two SEPARATE tools, deliberately. A single tool with a source parameter is
# what produced F-27: asked for Planner, the model called the task tool, got
# "unavailable", silently re-called it for To-Do, and presented that as the
# answer. Separate names make substitution structurally impossible — this tool
# CANNOT return To-Do data, so there is nothing to swap in.
# ---------------------------------------------------------------------------

def _substitution_ban(other_tool: str, this_system: str, other_system: str) -> str:
    """The anti-F-27 clause. Identical wording both ways, so neither can drift."""
    return (
        f"This tool returns {this_system} data ONLY. "
        f"If it is empty or unavailable, say so plainly and STOP. "
        f"Do NOT call `{other_tool}` and present {other_system} data as the answer to a "
        f"question about {this_system} — they are different systems and the user asked for "
        f"one specifically. You may OFFER to check {other_system} as a follow-up question, "
        f"but only as a question, and only after clearly stating that {this_system} "
        f"returned nothing."
    )


def _unavailable(source_label: str, other_tool: str, this_system: str, other_system: str,
                 reason: str) -> dict:
    """Shared shape for 'the source could not be read' — never an empty success."""
    return {
        "available": False,
        "source": source_label,
        "lists_scanned": [],
        "counts": {"overdue": 0, "upcoming": 0, "rest": 0, "total": 0},
        "groups": {"overdue": [], "upcoming": {}, "rest": {}},
        "display_table": (
            f"### ❌ {this_system} is not available\n\n{reason}"
        ),
        "instruction": (
            f"Tell the user plainly that {this_system} could not be read, and give the "
            f"reason from `display_table`. Then STOP. "
            + _substitution_ban(other_tool, this_system, other_system)
            + get_governance_rules()
        ),
    }


@mcp.tool
async def list_planner_tasks(include_completed: bool = False) -> dict:
    """
    List MICROSOFT PLANNER tasks assigned to the signed-in user.

    Use this ONLY when the user asks about Planner, plans, boards, buckets, or
    project tasks. For Microsoft To-Do, personal task lists, or "my to-do list",
    use `list_todo_tasks` instead — this tool cannot read To-Do.

    Results are pre-grouped and pre-ordered:
      1. Overdue                     — ordered by due date, most overdue first
      2. Due in the next two weeks   — grouped by plan name, ordered by due date
      3. No due date or due later    — grouped by plan name, newest first

    Args:
        include_completed: include tasks at 100% complete. Default False.

    Returns:
        dict with `display_table` (render verbatim), `groups`, `counts`, `instruction`.
    """
    # -> dict, NOT -> list[dict]. FastMCP 3.x derives an output schema from this
    # annotation and rejects every response that does not match it. That was L-01,
    # and it took three sessions to find because the error path returns a dict too,
    # so a broken tool and a throwing tool looked identical.
    try:
        tasks = await fetch_planner_tasks(include_completed=include_completed)
    except Exception as exc:
        message = str(exc)
        if any(m in message for m in ("403", "Forbidden", "accessDenied")):
            return _unavailable(
                "planner", "list_todo_tasks", "Microsoft Planner", "Microsoft To-Do",
                "Graph returned **403 Forbidden**. The signed-in token does not carry the "
                "`Tasks.Read` permission, or this account is not a member of the group that "
                "owns the plan. This is an Azure AD consent issue, not a temporary error — "
                "retrying will not help.",
            )
        return format_tool_error("list_planner_tasks", exc)

    now = utc_now()
    grouped = group_tasks(tasks, now=now)
    table = render_markdown(
        grouped, container_label="Plan", extra_label="Priority", now=now,
    )

    log_tool_call("list_planner_tasks", {"include_completed": include_completed})

    return {
        "available": True,
        "source": "planner",
        "lists_scanned": sorted({t["container"] for t in tasks}),
        "counts": grouped["counts"],
        "groups": {
            "overdue": grouped["overdue"],
            "upcoming": grouped["upcoming"],
            "rest": grouped["rest"],
        },
        "display_table": table,
        "instruction": (
            "Render `display_table` EXACTLY as provided. It is already grouped and ordered "
            "to the user's agreed scheme: overdue first, then the next two weeks by plan, "
            "then everything else. Do NOT re-sort it, do NOT merge the groups, do NOT "
            "summarise it into prose unless the user explicitly asks for a summary, and do "
            "NOT invent a different ordering. "
            "If `counts.total` is 0, say there are no outstanding Planner tasks — do not "
            "imply the tool failed. "
            + _substitution_ban(
                "list_todo_tasks", "Microsoft Planner", "Microsoft To-Do",
            )
            + get_governance_rules()
        ),
    }


@mcp.tool
async def list_todo_tasks(include_completed: bool = False) -> dict:
    """
    List MICROSOFT TO-DO tasks across every one of the user's To-Do lists.

    Use this ONLY when the user asks about To-Do, their task list, personal
    reminders, or "my to-do list". For Microsoft Planner, plans, boards, or
    project tasks, use `list_planner_tasks` instead — this tool cannot read
    Planner.

    Results are pre-grouped and pre-ordered:
      1. Overdue                     — ordered by due date, most overdue first
      2. Due in the next two weeks   — grouped by list name, ordered by due date
      3. No due date or due later    — grouped by list name, newest first

    Args:
        include_completed: include completed tasks. Default False.

    Returns:
        dict with `display_table` (render verbatim), `groups`, `counts`, `instruction`.
    """
    try:
        tasks, lists = await fetch_all_todo_tasks(include_completed=include_completed)
    except Exception as exc:
        message = str(exc)
        if any(m in message for m in ("403", "Forbidden", "accessDenied")):
            return _unavailable(
                "todo", "list_planner_tasks", "Microsoft To-Do", "Microsoft Planner",
                "Graph returned **403 Forbidden**. The signed-in token does not carry the "
                "`Tasks.Read` permission. This is an Azure AD consent issue, not a temporary "
                "error — retrying will not help.",
            )
        return format_tool_error("list_todo_tasks", exc)

    now = utc_now()
    grouped = group_tasks(tasks, now=now)
    table = render_markdown(
        grouped, container_label="List", extra_label="Importance", now=now,
    )

    log_tool_call("list_todo_tasks", {"include_completed": include_completed})

    return {
        "available": True,
        "source": "todo",
        # Reported separately from the task counts so "4 lists, 0 tasks" is
        # distinguishable from "the call failed". An empty success and a silent
        # failure must never look the same.
        "lists_scanned": [l["name"] for l in lists],
        "counts": grouped["counts"],
        "groups": {
            "overdue": grouped["overdue"],
            "upcoming": grouped["upcoming"],
            "rest": grouped["rest"],
        },
        "display_table": table,
        "instruction": (
            "Render `display_table` EXACTLY as provided. It is already grouped and ordered "
            "to the user's agreed scheme: overdue first, then the next two weeks by list, "
            "then everything else. Do NOT re-sort it, do NOT merge the groups, do NOT "
            "summarise it into prose unless the user explicitly asks for a summary. "
            "`lists_scanned` names every list that was read — if `counts.total` is 0 but "
            "`lists_scanned` is non-empty, the lists exist and are genuinely empty; say that "
            "rather than implying an error. "
            + _substitution_ban(
                "list_planner_tasks", "Microsoft To-Do", "Microsoft Planner",
            )
            + get_governance_rules()
        ),
    }
```

> **`get_governance_rules()`** — replace with whatever `list_tasks` already uses in this
> file. If governance is a module-level constant rather than a function, drop the `()`.

---

## 8. Step 5 — `.env.example`

Append:

```bash
# ── Task tools (FEAT-01) ────────────────────────────────────────
# Horizon in days for the "due soon" group in list_planner_tasks and
# list_todo_tasks. Anything due beyond this falls into the third group.
TASK_HORIZON_DAYS=14

# Optional — only needed for the group-wide Planner extension (FEAT-01 §10).
# Leave unset for the default "tasks assigned to me" behaviour.
# PLANNER_GROUP_ID=
```

⚠️ **F-29:** this file is CRLF. After editing, run:

```bash
sed -i 's/\r$//' .env.example
```

Then add the same keys to the real `.env` (also CRLF — `sed` it too, or just let
`python-dotenv` strip them; it does, as the running server proves).

**No registration change is needed.** Both tools live in `tools/task_tools.py`, which
`server.py` already imports, and `@mcp.tool` registers them at import time.

---

## 9. Step 6 — Tests

### 9.1 NEW FILE: `tests/test_task_grouping.py`

Flat functions, no classes — pytest-asyncio STRICT mode reads `self` as a missing fixture
(D-03). Named for what it tests, not for a phase (D-04).

```python
"""Unit tests for the FEAT-01 grouping engine. Pure functions, no Graph mocks."""

from datetime import datetime, timedelta, timezone

import pytest

from utils.task_grouping import (
    group_tasks,
    render_markdown,
    to_utc,
    utc_now,
)

NOW = datetime(2026, 8, 4, 12, 0, 0, tzinfo=timezone.utc)


def _task(title, container="Plan A", due=None, created=None, **kw):
    return {
        "id": title,
        "title": title,
        "container": container,
        "due": due,
        "created": created or (NOW - timedelta(days=30)),
        "status": "Not started",
        "extra": "Medium",
        "completed": False,
        "source": "planner",
        **kw,
    }


# --- bucketing -------------------------------------------------------------

def test_overdue_task_lands_in_overdue_group():
    grouped = group_tasks([_task("late", due=NOW - timedelta(days=3))], now=NOW)
    assert grouped["counts"]["overdue"] == 1
    assert grouped["counts"]["upcoming"] == 0


def test_task_inside_horizon_lands_in_upcoming():
    grouped = group_tasks([_task("soon", due=NOW + timedelta(days=5))], now=NOW)
    assert grouped["counts"]["upcoming"] == 1
    assert "Plan A" in grouped["upcoming"]


def test_task_beyond_horizon_lands_in_rest():
    grouped = group_tasks([_task("later", due=NOW + timedelta(days=40))], now=NOW)
    assert grouped["counts"]["rest"] == 1


def test_task_with_no_due_date_lands_in_rest():
    grouped = group_tasks([_task("someday", due=None)], now=NOW)
    assert grouped["counts"]["rest"] == 1


def test_horizon_boundary_is_inclusive():
    """A task due exactly on the horizon is 'upcoming', not 'later'."""
    grouped = group_tasks(
        [_task("edge", due=NOW + timedelta(days=14))], now=NOW, horizon_days=14
    )
    assert grouped["counts"]["upcoming"] == 1


# --- ordering --------------------------------------------------------------

def test_overdue_is_ordered_most_overdue_first():
    grouped = group_tasks([
        _task("2d", due=NOW - timedelta(days=2)),
        _task("9d", due=NOW - timedelta(days=9)),
        _task("5d", due=NOW - timedelta(days=5)),
    ], now=NOW)
    assert [t["title"] for t in grouped["overdue"]] == ["9d", "5d", "2d"]


def test_upcoming_is_grouped_by_container_and_ordered_by_due_date():
    grouped = group_tasks([
        _task("b-late", container="Beta", due=NOW + timedelta(days=9)),
        _task("a-early", container="Alpha", due=NOW + timedelta(days=1)),
        _task("b-early", container="Beta", due=NOW + timedelta(days=3)),
    ], now=NOW)
    assert list(grouped["upcoming"].keys()) == ["Alpha", "Beta"]
    assert [t["title"] for t in grouped["upcoming"]["Beta"]] == ["b-early", "b-late"]


def test_rest_is_ordered_newest_created_first():
    grouped = group_tasks([
        _task("old", due=None, created=NOW - timedelta(days=90)),
        _task("new", due=None, created=NOW - timedelta(days=1)),
    ], now=NOW)
    assert [t["title"] for t in grouped["rest"]["Plan A"]] == ["new", "old"]


def test_task_with_no_created_date_does_not_crash_the_sort():
    grouped = group_tasks([
        _task("nocreated", due=None, created=None),
        _task("hascreated", due=None, created=NOW),
    ], now=NOW)
    assert grouped["counts"]["rest"] == 2


# --- date coercion (F-08 / F-26 regression guards) -------------------------

def test_to_utc_parses_planner_iso_string():
    assert to_utc("2026-08-10T00:00:00Z") == datetime(2026, 8, 10, tzinfo=timezone.utc)


def test_to_utc_handles_graph_seven_digit_fractional_seconds():
    """dateutil rejects >6 fractional digits; Graph emits 7. Must not raise."""
    result = to_utc("2026-08-10T00:00:00.0000000")
    assert result is not None
    assert result.tzinfo is not None


def test_to_utc_handles_todo_datetimetimezone_object():
    class Stub:
        date_time = "2026-08-10T09:30:00.0000000"
        time_zone = "UTC"

    result = to_utc(Stub())
    assert result == datetime(2026, 8, 10, 9, 30, tzinfo=timezone.utc)


def test_to_utc_always_returns_aware_datetimes():
    """F-08 was a naive/aware comparison TypeError. Nothing here may be naive."""
    for value in ("2026-08-10T00:00:00", datetime(2026, 8, 10)):
        assert to_utc(value).tzinfo is not None


def test_to_utc_returns_none_for_junk_instead_of_raising():
    assert to_utc("not a date at all") is None
    assert to_utc(None) is None


def test_utc_now_is_timezone_aware():
    """F-26: datetime.utcnow() is naive AND deprecated. Must never come back."""
    assert utc_now().tzinfo is not None


# --- rendering (F-12 / F-15 regression guards) -----------------------------

def test_pipes_in_task_titles_do_not_break_the_table():
    grouped = group_tasks([_task("Budget | Q3 | final", due=NOW - timedelta(days=1))], now=NOW)
    table = render_markdown(grouped, now=NOW)
    for line in table.splitlines():
        if line.startswith("| ") and "Budget" in line:
            assert line.count("|") == 6  # 5 columns => 6 delimiters, no phantom column


def test_newlines_in_task_titles_do_not_split_the_row():
    grouped = group_tasks([_task("line one\r\nline two", due=NOW - timedelta(days=1))], now=NOW)
    table = render_markdown(grouped, now=NOW)
    assert "line one" in table
    assert "\r\n" not in table


def test_empty_result_renders_a_sentence_not_a_bare_zero():
    table = render_markdown(group_tasks([], now=NOW), now=NOW)
    assert "_Nothing overdue._" in table


def test_container_label_switches_between_plan_and_list():
    grouped = group_tasks([_task("x", due=NOW + timedelta(days=2))], now=NOW)
    assert "| List |" in render_markdown(grouped, container_label="List", now=NOW)
    assert "| Plan |" in render_markdown(grouped, container_label="Plan", now=NOW)
```

### 9.2 Add to `tests/test_task_tools.py`

```python
import pytest

import tools.task_tools as task_tools


@pytest.mark.asyncio
async def test_list_planner_tasks_returns_a_dict_not_a_list(monkeypatch):
    """L-01: a -> list[dict] annotation makes FastMCP reject every response."""
    async def fake(**_kw):
        return []
    monkeypatch.setattr(task_tools, "fetch_planner_tasks", fake)

    result = await task_tools.list_planner_tasks()
    assert isinstance(result, dict)
    assert {"display_table", "instruction", "counts"} <= set(result)


@pytest.mark.asyncio
async def test_planner_403_refuses_and_forbids_todo_substitution(monkeypatch):
    """F-27: the whole point of this feature."""
    async def boom(**_kw):
        raise Exception("403 Forbidden: accessDenied")
    monkeypatch.setattr(task_tools, "fetch_planner_tasks", boom)

    result = await task_tools.list_planner_tasks()
    assert result["available"] is False
    assert "list_todo_tasks" in result["instruction"]
    assert "Do NOT call" in result["instruction"]


@pytest.mark.asyncio
async def test_todo_reports_lists_scanned_separately_from_task_count(monkeypatch):
    """'4 lists, 0 tasks' must be distinguishable from 'the call failed'."""
    async def fake(**_kw):
        return [], [{"id": "1", "name": "Tasks"}, {"id": "2", "name": "Work"}]
    monkeypatch.setattr(task_tools, "fetch_all_todo_tasks", fake)

    result = await task_tools.list_todo_tasks()
    assert result["available"] is True
    assert result["counts"]["total"] == 0
    assert result["lists_scanned"] == ["Tasks", "Work"]
```

### 9.3 Run

```bash
cd /opt/FiGPT_OutlookMCP
source .venv/bin/activate
python -m pytest tests/ -q
```

**Expect 90 → ~111 tests, all passing.** If `test_horizon_boundary_is_inclusive` fails, the
`due > horizon` comparison in `group_tasks` needs to be `>=` — decide which you want and
make the test match.

---

## 10. Optional Extension — group-wide Planner (F_FiGPT)

Add this **only if** the `/me` view proves too narrow (§2.4). It needs the corrected Group
Object ID from §2.3 and may need `Group.Read.All`.

**In `graph/task_client.py`:**

```python
async def fetch_group_planner_tasks(group_id: str, include_completed: bool = False) -> list[dict]:
    """
    Every task in every plan owned by a Microsoft 365 group — not just yours.

    Chain: GET /groups/{id}/planner/plans  ->  GET /planner/plans/{id}/tasks

    Unlike /me/planner/tasks this includes unassigned tasks and tasks assigned to
    other people, so it is the right call for a "what is the whole team carrying"
    view and the wrong one for "what is on my plate".
    """
    client = get_graph_client()

    plans_resp = await client.groups.by_group_id(group_id).planner.plans.get()
    plans = plans_resp.value if plans_resp and plans_resp.value else []

    async def _tasks_for(plan):
        title = plan.title or "(untitled plan)"
        _PLAN_TITLE_CACHE[plan.id] = title
        try:
            resp = await client.planner.plans.by_planner_plan_id(plan.id).tasks.get()
            return title, (resp.value if resp and resp.value else [])
        except Exception as exc:
            logger.warning("Plan %r could not be read: %s", title, exc)
            return title, []

    results = await asyncio.gather(*(_tasks_for(p) for p in plans if p and p.id))

    tasks = []
    for title, raw in results:
        for t in raw:
            completed = (t.percent_complete or 0) >= 100
            if completed and not include_completed:
                continue
            tasks.append({
                "id": t.id,
                "title": t.title or "(untitled task)",
                "container": title,
                "due": to_utc(t.due_date_time),
                "created": to_utc(t.created_date_time),
                "status": _planner_status(t.percent_complete),
                "extra": _planner_priority(t.priority),
                "completed": completed,
                "source": "planner",
            })
    return tasks
```

**Verify the group ID before wiring it in** — one command, and it fails fast on a bad GUID:

```bash
python -c "
import asyncio
from graph.graph_client_factory import get_graph_client
async def m():
    c = get_graph_client()
    r = await c.groups.by_group_id('PASTE-CORRECTED-GUID-HERE').planner.plans.get()
    for p in (r.value or []):
        print(p.id, '-', p.title)
asyncio.run(m())
"
```

> I have deliberately **not** added a third MCP tool for this. D-05 records that 36 tools is
> already at the edge of reliable LLM tool selection (~20–30 is the practical limit), and
> "my Planner tasks" vs "the team's Planner tasks" are close enough that a fourth task tool
> invites exactly the mis-selection F-27 is about. If you want group-wide, my
> recommendation is to **switch** `list_planner_tasks` to this function rather than add a
> tool alongside it.

---

## 11. Follow-up — retire `list_tasks` (do this LAST, in a separate change)

Once both new tools are verified in LibreChat, `list_tasks` is dead weight and the last
remaining route to F-27. Retiring it:

1. Delete the `@mcp.tool` decorator from `list_tasks` (keep the function if anything else
   calls it — `grep -rn "list_tasks" --include=*.py .` first).
2. Tool count 38 → 37.
3. Re-run the suite; update any test that asserts on the registered tool list.

**Not in this change.** Verify the new path works before removing the old one.

---

## 12. Step 7 — LibreChat behaviour tests (TEST-01 Round 9)

The unit tests do not exercise tool *selection*, which is where F-27 lived. These do.
Run them in order and record pass/fail.

| # | Prompt | Pass criteria |
|---|---|---|
| **T9.1** | `How many tools do you have?` | **38**. If 36, the module did not reload — restart `server.py` and reconnect LibreChat |
| **T9.2** | `Show me my Planner tasks.` | Calls `list_planner_tasks`. Renders three headed sections (🔴/🟠/⚪) with counts. Overdue rows show "(Nd ago)" |
| **T9.3** | `Show me my Microsoft To-Do tasks.` | Calls `list_todo_tasks`, **not** `list_planner_tasks`. Container column header reads **List**, not Plan |
| **T9.4** ⭐ | `Show me my Planner tasks.` *(run while Planner is 403 / has no data)* | **The F-27 test.** Must say Planner is unavailable and STOP. Must **not** show To-Do data. Offering "shall I check To-Do instead?" is a pass; *showing* To-Do is a fail |
| **T9.5** ⭐ | Answer `no` to T9.4's offer | Nothing further is fetched. No To-Do table appears |
| **T9.6** | `Which of my Planner tasks are overdue?` | Answers from the 🔴 section only. Does not re-sort or re-count |
| **T9.7** | `What's due in the next two weeks, grouped by project?` | 🟠 section, sub-headed by plan name, each table ordered by due date |
| **T9.8** | `Show my To-Do tasks` *(with at least one list empty)* | Reports the lists scanned. An empty list is described as empty, not as an error |
| **T9.9** | Create a Planner task titled `Budget \| Q3 \| final`, then re-run T9.2 | Table renders unbroken — one row, five columns. Guards F-12/F-15 |
| **T9.10** | `Show me all my tasks.` *(ambiguous on purpose)* | Model should **ask which system**, or call both and label each clearly. A silent pick of one is a soft fail — tell me which it chose |

⭐ = the two that matter. T9.4 and T9.5 are the reason this feature is shaped as two tools.

---

## 13. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `403` on `/me/planner/tasks` | B-01 — `Tasks.Read` not granted | §2.1, then §2.2 |
| `403` persists after the admin grants it | Token predates the grant | §2.2 — sign out of LibreChat, sign back in, check `scp` at jwt.ms |
| Plans show `(plan name unavailable)` | `/planner/plans/{id}` denied for that plan | Not fatal. You are not a member of the owning group, or `Group.Read.All` is genuinely needed |
| `AttributeError: ... has no attribute 'planner'` | SDK fluent path differs in this version | The L-02 failure mode. Send me the traceback and the output of `pip show msgraph-sdk` |
| `TypeError: can't compare offset-naive and offset-aware` | A date bypassed `to_utc()` | F-08 again. Every date must go through `to_utc()` — check the normalisation dicts in `task_client.py` |
| To-Do due dates all show `—` | `dueDateTime` arrived as `dateTimeTimeZone` and was not unwrapped | `to_utc()` handles it via `hasattr(value, "date_time")`. Confirm with step 4 of `bin/diag_tasks.py` |
| Table rows split across lines | A cell bypassed `table_cell()` | F-12/F-15. Every cell goes through it, no exceptions |
| `400 Bad Request` on the group call | Invalid GUID | §2.3 — the `v` in `36v5` |
| Tool count still 36 | Server not restarted, or LibreChat cached the tool list | Restart `python server.py`, then reconnect the LibreChat MCP entry |
| `ValueError: invalid literal for int()` on startup | `TASK_HORIZON_DAYS` has a CR or a non-number | F-29 — `sed -i 's/\r$//' .env` |

---

## 14. Ledger impact

| ID | Effect |
|---|---|
| **F-27** | ✅ **Closes structurally** once T9.4 + T9.5 pass. Two tools cannot substitute for one another |
| **B-01** | 🟡 **Ask narrowed** — `Tasks.ReadWrite` only; `Group.Read.All` dropped unless plan titles 403 |
| **OQ-3** | ✅ **Answered.** Workstream 2 is the task-view feature |
| **D-05** | Respected — 2 tools added, not 5; group-wide is a *replacement*, not an addition |
| **D-16** | Respected — `list_tasks` retirement is deliberately deferred to §11 |
| Tool count | 36 → **38** (→ 37 after §11) |
| Test count | 90 → **~111** |

---

## 15. Order of work

1. §2 — get `Tasks.ReadWrite` granted, re-issue the token. **Everything else waits on this.**
2. §3 — run `bin/diag_tasks.py`. Send me the output, especially step 4's `due = ...` line.
3. §4 — the six greps; adjust imports if any name differs.
4. §5 → §6 → §7 → §8 — paste in order. Each file is independent of the next.
5. §9 — run the suite. Expect ~111 passing.
6. Restart the server, reconnect LibreChat, confirm **38** tools.
7. §12 — Round 9, in order. **T9.4 and T9.5 are the ones to report first.**
8. Only then: §10 (if needed) and §11.

Stop at step 2 if the diagnostic fails. Do not paste code against a 403.


---

Delivered: E:\Projects\Outlook-mcp-proj\FEAT-01_planner-and-todo-task-views.md — 15 sections, complete build. PROJECT_STATE.md updated (OQ-3 closed, F-27 and B-01 rewritten, session logged).

Four things you need to know before you start

1. ⛔ B-01 blocks all of it — but I've narrowed the ask. The ledger had you requesting Tasks.ReadWrite + Tasks.ReadWrite.Shared + Group.Read.All. For reads, Group.Read.All is almost certainly unnecessary — /me/planner/tasks and /planner/plans/{id} are gated on Tasks.Read, and plan access is decided by your group membership, not a directory-read scope. Go back to the admin asking only for Tasks.ReadWrite (delegated, admin consent). It's a far easier approval, and it also covers your existing add_task_* tools.

2. ⚠️ The grant alone won't fix the 403. Granting a permission doesn't upgrade tokens that already exist. LibreChat is holding a token minted under the old scopes and will keep 403-ing. Sign out, sign back in, then check the scp claim at jwt.ms. This wastes a day if you don't know it.

3. ⚠️ Your Group Object ID isn't valid. 95111a38-36v5-4119-b162-716a3366145f — v isn't a hex digit, so Graph will 400 before any permission check. Re-copy it from Azure AD → Groups → F_FiGPT → Object ID. Doesn't block you: the main build never uses it.

4. ⚠️ /me/planner/tasks is narrower than it sounds. It returns tasks assigned to you — not unassigned tasks, not other people's tasks, even in plans you can see. You gave me a Group Object ID, which makes me think you may actually want "everything in the F_FiGPT plans". That's a different call chain. I built what you specified as the main path and wrote the group-wide variant as an optional §10 — start with /me, it needs no group ID and no extra permission.

The design call worth flagging

Two tools, not one with a source parameter — and that's the real F-27 fix.

F-27 was logged as needing an instruction-level fix. I don't think that's right. A single tool with a source switch makes substitution a legal move: the model satisfies "call the task tool" while quietly changing which system it read. No docstring hardening removes that affordance. list_planner_tasks structurally cannot return To-Do data, so there's nothing to swap in.

That's the D-15 precedent exactly — splitting create_draft_invite fixed F-17 after hardened docstrings hadn't. I've left list_tasks in place and deferred its retirement to §11, after the new tools verify.

Three traps written in rather than discovered later

- To-Do returns dueDateTime as a dateTimeTimeZone object, not a datetime — F-08's failure mode waiting to happen
- Graph emits 7 fractional-second digits; dateutil rejects more than 6
- Planner has no status field — it infers state from percentComplete (0/50/100)

Grouping logic went into utils/task_grouping.py as pure functions, so the ordering rules are unit-testable without mocking Graph. ~21 new tests (90 → ~111), including explicit regression guards for F-08, F-26, and F-12/F-15 (pipes and newlines in task titles).

Where to actually begin

Step 0 is bin/diag_tasks.py, §3 of the doc — before any tool code. Same pattern that closed L-01. It proves all four Graph paths and prints the exact SDK shapes.

Send me its output — particularly step 4's due = ... line — and I'll confirm the parsing before you paste 400 lines. Stop there if it 403s; don't write code against a permission error.

✻ Cooked for 8m 45s