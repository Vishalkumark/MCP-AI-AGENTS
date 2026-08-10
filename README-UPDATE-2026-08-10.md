# README update — areas to change only

**Target:** `OL-MCP-README.md` (v1.0.0, 2026-07-31)
**Date:** 2026-08-10 (session 13)
**Why:** the README predates FEAT-01 (08-05), the B-01 permission grant (08-05), F-30 and the
F-28 fixes. Its headline numbers — 36 tools, 90 tests, "Planner blocked" — are all wrong now.

**26 areas.** Line numbers are from your current file. Each `Find` string carries enough
context to be unique; where a bare number like `36` appears many times I've included the
surrounding sentence deliberately — **do not** global-replace `36` or `90`.

Work top-down: the line numbers shift as you go, but only ever *forward*, and only in the
few places where the replacement is longer than the original (areas 15, 20, 22, 23, 24, 26).

---

## ⚠️ Read first — three numbers I could not verify

Two of these you should confirm before pasting, because this README's own §15.3 rule 2 says
never state an unverified fact as verified.

| Fact | Value I've used | Basis | Action |
|---|---|---|---|
| Registered tools | **38** | `/health` on 2026-08-05 | Re-run `curl -s http://localhost:8000/health` to be safe |
| Unit tests | **112** | your run on 2026-08-05 | Re-run `pytest -q` |
| Behaviour tests | **41 across 9 rounds** | 36 in `TEST-01` + T9.1–T9.5 added by `FEAT-01` §12 | **How many now pass is unknown** — I've written it as "re-verify pending", see area 25. Don't let me invent a number here |

Also note: **round 9 lives in `FEAT-01`, not in `TEST-01`.** The behaviour protocol is now
split across two documents. Area 22 flags that; folding T9.x into `TEST-01` is a separate
tidy-up, not part of this update.

---

## Part 1 — Headline counts (areas 1–10)

### Area 1 — badges, lines 8–9

**Find:**
```markdown
![Tests](https://img.shields.io/badge/tests-90%20passing-brightgreen)
![Tools](https://img.shields.io/badge/MCP%20tools-36-blue)
```

**Replace with:**
```markdown
![Tests](https://img.shields.io/badge/tests-112%20passing-brightgreen)
![Tools](https://img.shields.io/badge/MCP%20tools-38-blue)
```

### Area 2 — Table of Contents, line 26

**Find:** `| [4. Tool Catalogue](#4-tool-catalogue) | All 36 MCP tools and their execution flow |`

**Replace with:** `| [4. Tool Catalogue](#4-tool-catalogue) | All 38 MCP tools and their execution flow |`

### Area 3 — §1.1 Short description, line 56

**Find:** `FiGPT OutlookMCP is a Python server that exposes 36 Outlook capabilities as MCP tools. LibreChat's language model calls those tools to read mail, search a mailbox semantically, draft replies, check colleague availability, send meeting invites, generate minutes of meeting and track follow-ups.`

**Replace with:** `FiGPT OutlookMCP is a Python server that exposes 38 Outlook capabilities as MCP tools. LibreChat's language model calls those tools to read mail, search a mailbox semantically, draft replies, check colleague availability, send meeting invites, generate minutes of meeting, track follow-ups and list Microsoft Planner and To-Do tasks in priority order.`

### Area 4 — §1.5, line 89

**Find:** `| No mailbox access | 36 MCP tools over Microsoft Graph, delegated to the signed-in user only |`

**Replace with:** `| No mailbox access | 38 MCP tools over Microsoft Graph, delegated to the signed-in user only |`

### Area 5 — §1.8 Who uses it, line 117

The Azure row still describes Planner as blocked. It was granted on 2026-08-05.

**Find:** `| **Azure or Entra ID administrators** | Grant Graph API permissions; currently the blocker for Microsoft Planner |`

**Replace with:**
```markdown
| **Azure or Entra ID administrators** | Grant Graph API permissions. All scopes the server needs are now consented (2026-08-05). Remaining administrator action is a directory one, not a permission one: the **Manager** attribute is unset in Entra ID, so `get_my_profile` cannot report a manager |
```

### Area 6 — §1.10 Non-functional requirements, line 145

**Find:** ``utils/error_handler.py` wraps every tool; `utils/validator.py` inspects every output. 90 unit tests plus a 36-test live behaviour protocol |`

**Replace with:** `` `utils/error_handler.py` wraps every tool; `utils/validator.py` inspects every output. 112 unit tests plus a 41-test live behaviour protocol | ``

### Area 7 — §2.2 Layered design, line 183

**Find:** `A["server.py<br/><i>entry point — wiring only</i>"] --> B["tools/<br/><i>36 @mcp.tool wrappers</i>"]`

**Replace with:** `A["server.py<br/><i>entry point — wiring only</i>"] --> B["tools/<br/><i>38 @mcp.tool wrappers</i>"]`

### Area 8 — §2.3 Request lifecycle, lines 211 and 224

Two separate edits in the same section.

**Find:** `The same nine steps run for every one of the 36 tools.`
**Replace with:** `The same nine steps run for every one of the 38 tools.`

**Find:** `L->>L: Model selects a tool from 36 docstrings`
**Replace with:** `L->>L: Model selects a tool from 38 docstrings`

### Area 9 — §2.5 Integrations, line 289

**Find:** `| Microsoft Planner | Outbound | Graph | **Blocked** — 403, awaiting admin consent |`

**Replace with:** `| Microsoft Planner | Outbound | Graph | **Live** — `Tasks.ReadWrite` consented 2026-08-05; reads and writes both working |`

### Area 10 — §3 repository tree, line 328

**Find:** `├── tools/                     # 36 MCP tool definitions`

**Replace with:** `├── tools/                     # 38 MCP tool definitions`

---

## Part 2 — Structure and catalogue (areas 11–17)

### Area 11 — §3.4 `tools/` table, line 382 + the note at 384–385

`task_tools.py` gained two tools. The "ledger correction" note is now half-wrong: it says
`task_tools.py` contains *only* `extract_tasks`, which stopped being true on 08-05.

**Find:**
```markdown
| `task_tools.py` | 1 | Action-item extraction from email text |

> **Ledger correction.** The findings ledger lists `list_tasks` under `tools/task_tools.py`. It is
> actually defined in `tools/followup_tools.py`. `task_tools.py` contains only `extract_tasks`.
```

**Replace with:**
```markdown
| `task_tools.py` | 3 | Action-item extraction from email text, plus the grouped Planner and To-Do task views |

> **Where the task tools actually live — this trips people up.** `list_tasks` (the original,
> now superseded) is defined in `tools/followup_tools.py`, **not** `task_tools.py`, despite what
> the findings ledger says. The two FEAT-01 views, `list_planner_tasks` and `list_todo_tasks`,
> *are* in `task_tools.py` alongside `extract_tasks`. So task functionality is split across two
> files until `list_tasks` is retired (FEAT-01 §11).
```

### Area 12 — §3.5 `graph/` table, lines 392 and 397

Two rows. `mail_client.py` needs its folder-scoping rule recorded where a reader will hit it,
and `task_client.py` gained five functions.

**Find:** `| `mail_client.py` | `fetch_recent_messages()`, `fetch_message_by_id()` | Message retrieval, search and normalisation to plain dicts |`

**Replace with:**
```markdown
| `mail_client.py` | `fetch_recent_messages()`, `fetch_message_by_id()`, `search_messages_by_keyword()`, `search_messages_advanced()`, `fetch_messages_paged()` | Message retrieval, search and normalisation to plain dicts. **Folder scoping is deliberate and asymmetric (F-28): *listing* is Inbox-scoped via `mail_folders.by_mail_folder_id("inbox")`; `$search` stays unscoped across all folders, because a keyword search must still find mail the user sent.** Do not "tidy" the remaining `client.me.messages` calls — they are correct |
```

**Find:** `| `task_client.py` | `create_todo_task()`, `create_planner_task()`, `get_todo_tasks()` | Microsoft To-Do and Planner |`

**Replace with:**
```markdown
| `task_client.py` | `create_todo_task()`, `create_planner_task()`, `get_todo_tasks()`, `fetch_planner_tasks()`, `fetch_plan_titles()`, `fetch_todo_lists()`, `fetch_todo_tasks()`, `fetch_all_todo_tasks()` | Microsoft To-Do and Planner. The `fetch_*` family backs the FEAT-01 grouped views; plan-title resolution works on `Tasks.Read` alone — `Group.Read.All` proved unnecessary for it |
```

### Area 13 — §4 intro, line 479

**Find:** `All 36 registered tools. Verify the live count at any time with `curl -s http://localhost:8000/health`.`

**Replace with:** `` All 38 registered tools. Verify the live count at any time with `curl -s http://localhost:8000/health`. ``

### Area 14 — §4.1 Profile, line 487

F-30 landed: manager and department were added.

**Find:** `| `get_my_profile` | R | Display name, email and job title of the signed-in user | `profile_tools` → `graph_client_factory.get_user_profile` |`

**Replace with:**
```markdown
| `get_my_profile` | R | Display name, email, job title, **department** and **direct manager** of the signed-in user | `profile_tools` → `graph_client_factory.get_user_profile` + a second call to `GET /me/manager` |

> **Two traps behind this one tool.** `department` is **not** in Graph's default `/me` property
> set — without an explicit `$select` it is always `None`. And `GET /me/manager` returns **404
> `Request_ResourceNotFound`**, not a null field, when the Manager attribute is unset; the tool
> degrades cleanly and says the value is unavailable. It is explicitly forbidden from *inferring*
> a manager — filling an absence with a guess is the same failure class as F-27.
```

### Area 15 — §4.2 Email, line 494

F-28 patches A and B are applied. Drop the warning — **but only once you have run the
`list_emails_paged` check in `FIX-06` §7.0.** If you haven't, leave this row alone for now.

**Find:** `| `list_emails_paged` | R | Large-volume paged listing | `email_tools` → `mail_client` — ⚠ currently returns Sent Items mixed with Inbox |`

**Replace with:** `` | `list_emails_paged` | R | Large-volume paged listing | `email_tools` → `mail_client` → inbox-scoped | ``

### Area 16 — §4.7 heading, line 538

Unchanged in content, but read it against area 17: follow-ups stay at 3 tools, and the tools
that moved out of the count are in §4.8. No edit needed here — listed only so you don't
re-count and think something is missing.

### Area 17 — §4.8 Tasks, lines 546–553

The big one. 4 tools → 6, Planner unblocked, `list_tasks` marked for retirement.

**Find:**
```markdown
### 4.8 Tasks — 4 tools

| Tool | Type | Purpose | Execution path |
|---|---|---|---|
| `extract_tasks` | R | Pull action items out of email text | `task_tools` → `mail_client` → `governance` |
| `add_task_todo` | W | Create a Microsoft To-Do task | `followup_tools` → `task_client.create_todo_task` |
| `add_task_planner` | W | Create a Planner task. **Team-visible.** Currently 403-blocked | `followup_tools` → `task_client.create_planner_task` |
| `list_tasks` | R | List tasks from To-Do or Planner | `followup_tools` → `task_client.get_todo_tasks` |
```

**Replace with:**
```markdown
### 4.8 Tasks — 6 tools

| Tool | Type | Purpose | Execution path |
|---|---|---|---|
| `extract_tasks` | R | Pull action items out of email text | `task_tools` → `mail_client` → `governance` |
| `add_task_todo` | W | Create a Microsoft To-Do task | `followup_tools` → `task_client.create_todo_task` |
| `add_task_planner` | W | Create a Planner task. **Team-visible — still wants an explicit confirm gate** | `followup_tools` → `task_client.create_planner_task` |
| `list_planner_tasks` | R | Planner tasks grouped Overdue → due within 14 days → everything else, sub-grouped by plan | `task_tools` → `task_client.fetch_planner_tasks` + `fetch_plan_titles` |
| `list_todo_tasks` | R | The same three-group view for Microsoft To-Do, sub-grouped by list | `task_tools` → `task_client.fetch_all_todo_tasks` |
| `list_tasks` | R | *Superseded.* The original combined To-Do/Planner listing | `followup_tools` → `task_client.get_todo_tasks` |

> **Why there are two named tools instead of one with a `source` parameter (F-27).** A single
> tool with a source switch makes substitution a *legal move*: the model can satisfy "call the
> task tool" while quietly changing which system it read, and no amount of docstring hardening
> removes that affordance. It did exactly that — asked for Planner, it answered with To-Do data
> under an honest heading but a dishonest answer. `list_planner_tasks` has **no code path that
> can reach To-Do**, so the substitution is now structurally impossible. Same move that fixed
> F-17 via the `compose_invite` / `save_invite` split.
>
> **`list_tasks` is scheduled for retirement** (FEAT-01 §11) — deliberately *after* the new
> tools proved themselves, not in the same change. Until then three tools can answer a task
> question, which is one more than is healthy for model tool-selection.
```

---

## Part 3 — Setup, testing, operations (areas 18–22)

### Area 18 — §5.1 Prerequisites, lines 622–623

**Find:**
```markdown
- **Granted delegated scopes:** `Mail.Read`, `Mail.ReadWrite`, `Mail.Send`, `Calendars.Read`, `Calendars.Read.Shared`, `Calendars.ReadWrite`, `User.Read`.
- **Not yet granted:** `Tasks.ReadWrite`, `Tasks.ReadWrite.Shared`, `Group.Read.All`. Without these, Planner returns 403 and only Microsoft To-Do works.
```

**Replace with:**
```markdown
- **Granted delegated scopes (all consented as of 2026-08-05):** `Mail.Read`, `Mail.ReadWrite`, `Mail.Send`, `Calendars.Read`, `Calendars.Read.Shared`, `Calendars.ReadWrite`, `User.Read`, `Tasks.Read`, `Tasks.ReadWrite`, `Tasks.ReadWrite.Shared`, `Group.Read.All`, `User.Read.All`.
- ⚠️ **Granting a scope does not upgrade a token that already exists.** This costs a day if you miss it. After any grant: add the scope to `GRAPH_API_SCOPES` in `.env`, re-run `bin/get_token.py`, and have LibreChat sign out and back in so its OAuth flow re-consents. Prove it by decoding the token's `scp` claim (`bin/diag_tasks.py` step 0, or paste the token into jwt.ms) **before** blaming the code for a 403.
```

### Area 19 — §5.2 Installation, lines 687 and 701

**Find:** `Expected: **90 passed**.`
**Replace with:** `Expected: **112 passed**.`

**Find:** `Expected: `"status": "healthy"` and `"tools_registered": 36`.`
**Replace with:** `` Expected: `"status": "healthy"` and `"tools_registered": 38`. ``

### Area 20 — §8.1 Unit suite, line 982

**Find:** `pytest -q                            # full suite — expect 90 passed`

**Replace with:** `pytest -q                            # full suite — expect 112 passed`

### Area 21 — §8.2 Live behaviour protocol, line 1000

**Find:** `` `TEST-01_librechat-behaviour-tests.md` defines 36 behaviour tests across 8 rounds, executed manually through LibreChat against a real mailbox. Each test names the specific regression it detects, so a failure is diagnosable without a round trip. ``

**Replace with:**
```markdown
The behaviour protocol is **split across two documents** — fold them together when convenient:

- `TEST-01_librechat-behaviour-tests.md` — 36 tests across rounds 1–8.
- `FEAT-01_planner-and-todo-task-views.md` §12 — round 9 (**T9.1–T9.5**), covering the two task views.

41 tests across 9 rounds, executed manually through LibreChat against a real mailbox. Each test
names the specific regression it detects, so a failure is diagnosable without a round trip.
```

### Area 22 — §11.3 Change workflow, line 1222

**Find:** `4. Run `pytest -q` — expect 90 passing.`

**Replace with:** `` 4. Run `pytest -q` — expect 112 passing. ``

---

## Part 4 — Governance, troubleshooting, roadmap (areas 23–26)

### Area 23 — §11.4 Delivery model, lines 1232–1233

Changed by your instruction on 2026-08-10.

**Find:**
```markdown
- **A small fix** — a paste-ready `Find:` / `Replace with:` block.
- **Several fixes** — one structured `.md` file with a checklist and per-block verification.
- **Always** step-by-step, always explained, always verifiable.
```

**Replace with:**
```markdown
- **A change to one file** — the **complete updated file**, plus a separate section stating
  exactly what changed and why. Positional patches ("apply at line 330, not the identical line
  at 194") put the risk on the person who cannot see the diff; handing over the whole file moves
  it back to the author.
- **Several fixes, or edits scattered across many files** — one structured `.md` file with a
  checklist, per-area `Find:` / `Replace with:` blocks and per-block verification.
- **Never** ask the reader to reconcile helper names or signatures against the codebase. Resolve
  them from the snapshot before delivery. That footnote caused six test failures on 2026-08-05.
- **Always** step-by-step, always explained, always verifiable.

> ⚠️ **Whole-file delivery meets a line-endings trap.** 38 files in this repository carry
> whole-file CRLF, inherited from the project's Windows origin (F-29). Pasting LF content over
> one of them makes `git diff` report every line as changed, burying the real edit. Check with
> `git diff --ignore-cr-at-eol <file>`, and either commit the normalisation on its own or restore
> CRLF with `sed -i 's/$/\r/' <file>` afterwards.
```

### Area 24 — §12 Troubleshooting, line 1251 + three new rows

**Find:** `| **Planner returns 403** | `Tasks.ReadWrite`, `Tasks.ReadWrite.Shared` and `Group.Read.All` are not consented | Azure administrator action. To-Do continues to work |`

**Replace with:**
```markdown
| **Planner returns 403** | The scopes **are** consented (2026-08-05) — so a 403 now means the *token* predates the grant, not that permission is missing | Add the scope to `GRAPH_API_SCOPES`, re-run `bin/get_token.py`, sign LibreChat out and back in. Decode the `scp` claim to confirm before touching code |
| **`/groups/{id}/...` returns 403 with a valid-looking ID** | Graph checks authorisation **before** existence, so a wrong or placeholder group ID 403s rather than 404s | **A 403 from a group endpoint is not evidence about permissions unless the ID is real.** Verify the GUID first. This has already caused one false alarm |
| **`get_my_profile` reports no manager** | `GET /me/manager` returns 404 `Request_ResourceNotFound` — the Manager attribute is unset in Entra ID | Directory change, not a code change: Entra ID → Users → account → Properties → Job information → Manager. The tool degrades cleanly and must never infer a value |
| **A profile field renders as the literal string `None`** | `dict.get(key, "—")` returns `None` when the key **exists and is None** — the default only fires when the key is *absent* | Test the value, not the key's presence: `value = d.get(key) or "—"` |
| **Sent Items appear in an inbox listing** | A `client.me.messages` call is `GET /me/messages` = every folder | Scope it: `client.me.mail_folders.by_mail_folder_id("inbox").messages`. **But not on `$search` calls** — keyword search must span all folders (F-28) |
```

### Area 25 — §13 ADR-05, line 1272 + one new ADR

**Find:** `| **ADR-05** | 2026-07-22 | **Microsoft Teams gets its own MCP server**, not tools inside this one | 36 tools is already at the edge of reliable model tool selection (20–30 is the practical limit); mature community Teams servers exist; separate servers fail independently |`

**Replace with:** `| **ADR-05** | 2026-07-22 | **Microsoft Teams gets its own MCP server**, not tools inside this one | 38 tools is already past the edge of reliable model tool selection (20–30 is the practical limit); mature community Teams servers exist; separate servers fail independently |`

Then **append a new row** at the end of the ADR table:

```markdown
| **ADR-17** | 2026-08-04 | **Where two data sources could be confused, split them into two separately-named tools** rather than one tool with a `source` parameter | A source switch makes substitution a legal move — the model satisfies "call the task tool" while silently changing which system it read, and docstring hardening cannot remove that affordance. Proven twice: the `compose_invite`/`save_invite` split fixed F-17, and `list_planner_tasks`/`list_todo_tasks` fixed F-27. Cost is tool count, which ADR-05 already flags as the binding constraint |
```

### Area 26 — §14 Roadmap, lines 1287–1331 (replace the whole section)

Almost every row is stale: repository hygiene largely landed, reply-draft threading was never
a bug, the substitution and paged-scope items are fixed, and the "not yet scoped" capability
shipped on 08-05.

**Replace §14 in full with:**

```markdown
## 14. Roadmap

Priority order. Current status: no live bugs and every High-severity defect closed. Workstream 2
(the grouped task views) shipped on 2026-08-05. What remains is Medium-or-below.

### 14.1 Immediate

| Item | Severity | Description |
|---|---|---|
| F-28 final call site | Medium | `search_messages_advanced` still lists across all folders on its `$filter` and no-filter branches. Patches A and B are applied; the last site is delivered as `FIX-06`. **The `$search` branches must stay unscoped** — that is the trap in this fix |
| `.gitignore` missing `.venv/` | Medium | The 44,962 venv files are out of the index, but no ignore rule matches them, so one `git add -A` puts them straight back. **Until the line exists, stage by explicit path only** |
| Three uncommitted changes | Medium | FEAT-01, FIX-05 and the F-30 manager work are live on the server but unpushed |

### 14.2 Next — planned `FIX-04`

| Item | Severity | Description |
|---|---|---|
| Dependency pinning | Medium | Upper bounds and a lockfile. `fastmcp>=2.0.0` spans the 2→3 major that caused the HTTP 421 incident. Drop the now-unused `matplotlib` |
| Chart `x_label` | Low | Accepted and documented but never emitted. Confirm the bar branch renders before fixing — the one recorded test result used the pie-chart wording |
| `run_all_test.sh` | Low | Smart quotes and en-dashes make it non-executable. Confirmed a one-part fix — the file carries no CRs |
| `server.py` cleanup | Low | Unused imports, an unapplied CORS placeholder, and a docstring still claiming 9 tools. Also verify whether `mcp.run(host=, port=)` is honoured or silently ignored |
| Dead code removal | Low | A parked Requesty string block, a duplicate import, unused chart constants, an unused `dateutil` import in `mail_client.py`, and a `_needs_followup` check against a key that is never set |
| Audit allowlist review | Low | `title` is allowlisted while `subject` is excluded as sensitive — but task titles are often derived from the subject |
| Deprecated `datetime.utcnow()` | Low | Replace with `datetime.now(timezone.utc)` in `availability_client.py` |
| `$search` + `$filter` contradiction | Low | `search_messages_by_keyword` combines them; `search_messages_advanced`'s own comment says that is impossible and filters client-side instead. One of the two is wrong. Decide deliberately — do not blind-edit |
| Line endings | Low | `.gitattributes` (`* text=auto eol=lf`) plus `git add --renormalize .`, **as its own commit** — it touches 38 files and would bury anything bundled with it |

### 14.3 Hardening backlog

Rate limiting and scope validation first — see [Known hardening gaps](#104-known-hardening-gaps).
Then temp-file cleanup, the remaining HITL gates, `ServiceException` handling, and the GDPR
data-flow document.

### 14.4 Infrastructure

- **systemd unit** — the last piece of the deployment story. Persistent logging already exists.

### 14.5 Externally blocked

| Blocker | Owner | Status |
|---|---|---|
| Manager attribute unset in Entra ID — `get_my_profile` cannot report a manager | Azure administrator | Open. Directory change, not a permission or code change |
| `find_availability` is tenant-internal only | — | Graph behaviour, not a defect. Documented |
| No systemd service | Server team | Open |

### 14.6 Retirement

- **`list_tasks`** — superseded by `list_planner_tasks` and `list_todo_tasks`. Deliberately kept
  until the replacements had proved themselves in live use; that has now happened. Removing it
  drops the tool count to 37 and takes a redundant option away from model tool-selection
  (see ADR-05).
```

---

## Part 5 — Version stamp (do this last)

### §15.2 Versioning

**Find:** `- **Document version:** 1.0.0 — 2026-07-31.`

**Replace with:** `- **Document version:** 1.1.0 — 2026-08-10. *(Minor: two new tools.)*`

### §15.4 Last verified — replace the whole table

**Find:**
```markdown
| Fact | Value | Verified |
|---|---|---|
| Registered tools | 36 | 2026-07-28, live `/health` |
| Unit tests | 90 passing | 2026-07-28 |
| Behaviour tests | 34 of 36 passing | 2026-07-28 |
| FastMCP | 3.4.3 | 2026-07-28 |
| Python | 3.12 | 2026-07-28 |
| `Calendars.ReadWrite` | Granted | 2026-07-28, proven by a real `POST /me/events` |
| Planner access | 403, blocked | 2026-07-28 |
```

**Replace with:**
```markdown
| Fact | Value | Verified |
|---|---|---|
| Registered tools | 38 | 2026-08-05, live `/health` |
| Unit tests | 112 passing | 2026-08-05 |
| Behaviour tests | 41 defined across 9 rounds | **Pass count not re-verified since round 9 was added — re-run before quoting a number** |
| FastMCP | 3.4.3 | 2026-07-28 |
| Python | 3.12 | 2026-07-28 |
| `Calendars.ReadWrite` | Granted | 2026-07-28, proven by a real `POST /me/events` |
| Planner and To-Do access | Granted and working | 2026-08-05, proven by `bin/diag_tasks.py` passing end to end and both tools returning data in LibreChat |
| `GET /me/manager` | 404 — attribute unset in Entra ID | 2026-08-05 |
| Inbox scoping (F-28) | 2 of 3 sites applied | 2026-08-10, from `grep -n 'client.me.messages.get'`. **In the file, not yet runtime-proven** |
```

---

## Checklist

```
[ ] Areas 1–10   headline counts (36→38, 90→112)
[ ] Areas 11–17  structure and tool catalogue     ← area 15 waits on FIX-06 §7.0
[ ] Areas 18–22  setup, testing, operations
[ ] Areas 23–26  governance, troubleshooting, roadmap
[ ] Part 5       version stamp and Last-verified table
```

Then confirm nothing was missed:

```bash
grep -n '\b36\b\|\b90 \|403-blocked\|Not yet granted\|not yet scoped' OL-MCP-README.md
```

Every surviving hit should be a *deliberate* 36 or 90 (there may be none). Anything about
Planner being blocked or the capability being unscoped means an area was skipped.
