# TEST-01 — LibreChat Behaviour Test Script

**Project:** FiGPT_OutlookMCP
**Target:** `/opt/FiGPT_OutlookMCP` on FGSV2187
**Date:** 27 July 2026
**Tests:** everything applied in `FIX-03_complete-remediation.md` (Blocks A–F), plus regression cover for the Session 4 fixes (L-01, F-12, F-03, FIX-02 B/C/D)

Unit tests prove the code runs. These prove the **model actually behaves correctly** when it
uses the tools — which is where every single one of L-01, L-02 and L-03 actually lived.

---

## How to use this document

Work **top to bottom**. Rounds are ordered from safest to most consequential — Round 7
contacts real people, so it comes last and has its own protocol.

For each test:

- **Prompt** — paste it into LibreChat verbatim (substituting `{{...}}` placeholders)
- **Pass** — what a correct result looks like
- **Fail** — what the specific regression looks like, so you can tell a real failure from a
  model being chatty

**If a test fails, capture two things before moving on:**

1. LibreChat's full response text
2. `tail -50` of the terminal running `server.py` (the traceback lands there)

Without the traceback a failure is a guessing game — that cost us two round trips on 26 July.

Fill in the results table at the end and send it back.

---

## Pre-flight — before opening LibreChat

```bash
cd /opt/FiGPT_OutlookMCP
source .venv/bin/activate

# 1. Tool count — this is the single most informative check
curl -s http://localhost:8000/health | python3 -m json.tool
```

| Result | Meaning |
|---|---|
| `"tools_registered": 36` | ✅ Correct. 35 − `create_draft_invite` + `compose_invite` + `save_invite` |
| `35` | `save_invite` or `compose_invite` didn't register — check for a duplicate function name or a missing `@mcp.tool` in `tools/draft_tools.py` |
| `34` | The old `create_draft_invite` was deleted but only one replacement registered |

```bash
# 2. Confirm the split actually took, by name
curl -s http://localhost:8000/health | python3 -c "import sys,json; t=json.load(sys.stdin); print([x for x in str(t) if 0])" 2>/dev/null
grep -n "async def compose_invite\|async def save_invite\|async def create_draft_invite" tools/draft_tools.py
```

Expect exactly two hits: `compose_invite` and `save_invite`. **Any hit for
`create_draft_invite` means the old tool is still registered** and the model may still call it.

```bash
# 3. Persistent logging (E1) — should exist and be growing
ls -la logs/
```

Expect a rotating log file alongside `audit.log`. If `logs/` holds only `audit.log`, E1 didn't
take effect — the tracebacks you need for the rest of this script will only exist in terminal
scrollback (F-11).

**Then, in LibreChat:** disconnect and reconnect the `Fichtner-Outlook` MCP server.
`startup: false` means it will not pick up the new tool list on its own.

---

## Round 0 — Smoke

### T0.1 — Tool list

**Prompt:**
```
What Outlook tools do you have available? Just list their names.
```

- **Pass:** 36 tools listed, including `compose_invite` and `save_invite`.
- **Fail:** `create_draft_invite` appears → LibreChat is still on the old tool list; reconnect
  again. Do **not** proceed to Round 7 until this is clean.

---

## Round 1 — Inbox read paths
*Regression cover for L-01 (closed 26 Jul) and F-12. If these break, something in Block C
undid Session 4's work.*

### T1.1 — Basic listing

**Prompt:**
```
Show me my latest 10 emails.
```

- **Pass:** A markdown table. Sender column shows **real display names**. A 🔵/⚪ column is
  present. Every row is on one line.
- **Fail (L-01 relapse):** senders blank or literal `Ellipsis`; or an "Output validation error";
  or no read-status column.
- **Fail (F-12 relapse):** a row splits across lines, or a phantom extra column appears —
  that's an unescaped `|` or `\r\n` in the preview text.

### T1.2 — The original complaint

**Prompt:**
```
Which of those are unread?
```

- **Pass:** A filtered list or a count.
- **Fail:** *"the inbox listing doesn't expose a read/unread flag"* — the exact sentence that
  started this whole investigation. If you see it, L-01 has regressed.

### T1.3 — Paged listing

**Prompt:**
```
Show me the next page of my inbox.
```

- **Pass:** `list_emails_paged` runs, and it **also** shows the 🔵/⚪ column (that was FIX-02
  Fix E — the paged tool originally had no read status at all).

### T1.4 — Table integrity under load

**Prompt:**
```
Show me my latest 25 emails.
```

- **Pass:** 25 clean rows. With that many, at least one preview almost certainly contains a
  line break — this is the real F-12 / C2 test.
- **Fail:** any mangled row.

---

## Round 2 — Search paths
*Covers C3, C4, C5, C8 and F-16 — the L-01 pattern in the six tools it was never fixed in.*

### T2.1 — Keyword search + read status

**Prompt:**
```
Search my emails for "report".
```

- **Pass:** Results table **with a 🔵/⚪ column**, rows unbroken. The Preview column was
  flagged as the highest-risk one for the pipe/newline bug — check it closely.
- **Fail:** no read column → C3 not applied.

### T2.2 — Read status on search results

**Prompt:**
```
Which of those search results are unread?
```

- **Pass:** A filtered answer.
- **Fail:** the L-01 refusal sentence, now coming from `search_emails` instead of
  `list_emails` — that's exactly what F-16 predicted.

### T2.3 — The timezone crash

**Prompt:**
```
Find emails about "report" between 1 July 2026 and 26 July 2026.
```

- **Pass:** Results, or a clean "no matches found".
- **Fail (C8):** `TypeError: can't compare offset-naive and offset-aware datetimes`. This is
  the keyword **+** date-filter path specifically — keyword-only and date-only both worked
  before, which is why it stayed hidden.

### T2.4 — Advanced search

**Prompt:**
```
Find emails from {{a colleague's email}} that have attachments, from the last 30 days.
```

- **Pass:** Clean table with the read column.

### T2.5 — Semantic search

**Prompt:**
```
Semantically search my emails for anything about budget approval delays.
```

- **Pass:** Results with a 🔵/⚪ column (C5), and clean cells.
- **Note:** first call is slow — it embeds up to 200 emails with no cache (F-10, deliberately
  not fixed yet). 10–30 s is expected, not a failure. If it errors on a **model download**,
  that answers OQ-8 about outbound access on the VM — capture the error.

---

## Round 3 — Drafts and replies
*Covers A4 / F-14, C7, and regression cover for L-03.*

### T3.1 — Draft listing with safe key access

**Prompt:**
```
Show me my draft emails.
```

- **Pass:** Clean table. Critically, **a draft with an empty subject or no body must not
  crash it** — C7 replaced `d["subject"]`-style bracket indexing with safe access.
- **Fail:** `KeyError: 'preview'` or similar.
- **Tip:** if you have no odd drafts, make one in Outlook with no subject and no body first.
  That's the case C7 exists for.

### T3.2 — The compose gate (D-02)

**Prompt:**
```
Compose an email to {{your own address}} with the subject "MCP test — draft path"
saying the draft pipeline is being verified.
```

- **Pass:** The email is shown **in chat** and you are asked to approve. Nothing saved yet.
- **Fail:** it saves straight to Outlook without asking → the D-02 gate is broken.

### T3.3 — The wrong-confirmation bug

**Prompt:**
```
Yes, save it.
```

- **Pass:** Confirmation reads **"Draft Saved"** (or similar), and the table below it renders
  correctly.
- **Fail (F-14 / A4):** *"✅ Meeting invite created and notifications sent to all attendees."*
  on an ordinary email draft. Also check the table immediately below the confirmation isn't
  broken — the original bug had no trailing newline.
- **Then:** open Outlook, confirm the draft body contains **only** your message — no
  instruction text, no governance rules, no raw context. That's what D-02 exists to prevent.

### T3.4 — Reply threading (L-03)

Pick an email in your inbox with at least one prior message in the thread.

**Prompt:**
```
Compose a reply to the email from {{sender}} about {{subject}}, saying I'll review it
by Friday.
```
then
```
Yes, save it.
```

- **Pass:** In Outlook the draft appears **nested inside the original conversation**.
- **Fail (L-03):** it appears as a **standalone email** in Drafts, detached from the thread —
  meaning the model chose `save_draft_to_outlook` instead of `save_reply_draft`. Tell me and
  I'll harden the docstring; the instruction text is already hardened, so this would mean
  tool-selection ambiguity rather than a code bug.

---

## Round 4 — Follow-ups
*Covers D1, D2, C6, C9 — and F-05 / F-18, the two findings that made this feature
quietly wrong rather than broken.*

### T4.1 — Follow-ups that respect replies

**Prompt:**
```
Which of my emails need a follow-up?
```

- **Pass:** Clean table (C6). Crucially, **emails you have already replied to are absent**.
- **Fail (D1 / F-05):** the list includes threads you answered days ago. Before D1,
  `_needs_followup` filtered only on age and "not from me" — `check_email_replied` existed and
  worked but was never called. This is the whole point of Block D.

### T4.2 — Verify against a known case

Pick one email you **know** you replied to.

**Prompt:**
```
Does the email from {{sender}} about {{subject}} still need a follow-up?
```

- **Pass:** "No — you replied on {{date}}."
- **Fail:** it says a follow-up is needed → D1 isn't wired in.

### T4.3 — Empty-thread edge case

Pick a **standalone** email with no thread history (a newsletter works well).

**Prompt:**
```
Has anyone replied to the email from {{sender}} about {{subject}}?
```

- **Pass:** A clean "no replies" answer.
- **Fail (C9 / F-10):** `IndexError: list index out of range` — `thread[0]` on an empty list.

### T4.4 — Follow-up threading

Pick an email from T4.1's list.

**Prompt:**
```
Compose a follow-up for the email from {{sender}} about {{subject}}.
```
then
```
Yes, save it.
```

- **Pass:** Shown for approval first, then saved **threaded under the original**.
- **Fail (D2 / F-18):** a standalone draft with a manufactured `Re:` subject. This is L-03
  living in a second tool — `compose_reply` was hardened in July, `compose_followup` wasn't,
  and it explicitly instructed the model to call `save_draft_to_outlook` with a fake subject.

---

## Round 5 — MOM and task extraction
*Covers B1, B2, F2, C6, F7 — the prompt-corruption block.*

### T5.1 — MOM generation

Pick a thread with a few messages.

**Prompt:**
```
Generate minutes of meeting from the email thread with {{sender}} about {{subject}}.
```

- **Pass:** Clean MOM. Read the output carefully for:
  - No literal `f"` or `"` fragments in the text
  - No literal `\n\n` rendered as characters
  - No **duplicated** governance/anti-hallucination block
- **Fail (B1):** any of the above. The original had `f"{governance}\n\n"` sitting *inside* the
  triple-quoted block as raw text, so the f-string source itself leaked into the prompt sent to
  the model — and governance was then injected a second time on top.

### T5.2 — MOM as HTML (the bare-"1." crash)

**Prompt:**
```
Give me that MOM as HTML.
```

- **Pass:** HTML output.
- **Fail (F2 / F-23):** `IndexError: string index out of range`. `_convert_mom_to_html` tested
  `stripped[2] == " "` on any line starting `1.` — a line that is *just* `"1."` with nothing
  after it has no index 2. MOM text is model-generated, so this is genuinely reachable; it just
  needs the model to emit a bare numbered line.

### T5.3 — Task extraction

**Prompt:**
```
Extract action items from my recent emails.
```

- **Pass:** A task list, with governance rules appearing **once**.
- **Fail (B2):** the same rules block printed twice — `extract_tasks` appended
  `get_task_rules()` on top of the governance already injected.

### T5.4 — Task listing

**Prompt:**
```
Show me my tasks.
```

- **Pass:** Clean table (C6).

### T5.5 — Planner honesty

**Prompt:**
```
Show me my Planner tasks.
```

- **Pass:** A clear message that Planner isn't available / a clean 403 explanation.
- **Fail:** an unhandled crash, or a claim that it worked. Planner is externally blocked
  (B-01 — needs `Group.Read.All` admin consent, confirmed 403 on 20 July). F7 fixed the
  docstring that overpromised this.

---

## Round 6 — Calendar reads
*Regression cover for FIX-02 Fix C — `list_calendar_events` had been failing on **every**
call before 26 July.*

### T6.1 — Calendar listing

**Prompt:**
```
What's on my calendar this week?
```

- **Pass:** Events listed.
- **Fail:** "Output validation error" or a fallback apology → Fix C regressed.

### T6.2 — Availability

**Prompt:**
```
When is {{an internal colleague}} free tomorrow?
```

- **Pass:** Free/busy blocks.
- **Note (B-03):** this only works for accounts **inside the tenant**. An external address
  returning nothing is correct Graph behaviour, not a bug. Use an internal colleague.

---

## Round 7 — ⚠️ The invite gate (Block A)

**This round sends real calendar invitations to real people. It cannot be undone.**

That is the entire finding: F-17 established that `POST /me/events` emails every attendee the
moment it runs, while the old tool told the user it had only saved a draft. The crash (L-02)
was the only thing preventing that from happening silently.

**Run the four tests in order. Do not skip to T7.4.**

| Test | Attendee | Sends anything? |
|---|---|---|
| T7.1 | your own address | No — gate test |
| T7.2 | your own address | No — refusal test |
| T7.3 | your own address | **Yes, to you only** |
| T7.4 | one internal colleague | **Yes, to them** |

### T7.1 — The gate holds

**Prompt:**
```
Draft a meeting invite to {{your own address}} for tomorrow at 10am, 30 minutes,
subject "MCP invite gate test", agenda: verify the approval gate works.
```

- **Pass — all three required:**
  1. A rendered preview table of the invite
  2. An explicit warning that saving will **create the event and immediately email every
     attendee, and cannot be undone**
  3. A question asking whether to send
- **Then verify:** open your Outlook calendar. **There must be no event yet**, and no email.
- **Fail (critical):** the model calls `save_invite` directly and the invite goes out with no
  preview. The approval gate is the only thing standing between a user's first request and an
  email to external parties. If this fails, stop Round 7 and tell me — the fix is docstring and
  instruction wording, and I'd rather harden it than have you test the send path with a gate
  that doesn't hold.

### T7.2 — The gate holds on refusal

**Prompt:**
```
Actually, cancel that — don't send it.
```

- **Pass:** Nothing created, nothing sent. Confirm again in your calendar.
- **Fail:** the event exists → the model called `save_invite` regardless of your answer.

### T7.3 — The send path, to yourself only

Re-run the T7.1 prompt, then:

**Prompt:**
```
Yes, send it.
```

- **Pass — all four:**
  1. The event appears in your calendar
  2. You receive the invitation email
  3. The confirmation is **past tense** — "Meeting Invite Sent", "invitations emailed to…"
  4. It does **not** describe the result as a draft, and does **not** tell you to send it
     yourself from Outlook Calendar
- **Fail (A2 / A3):** *"Draft saved to your Calendar. Review and send from Outlook Calendar."*
  The invitation has gone out and the tool is telling you it hasn't — that is F-17 exactly, and
  it means A2 or the A3 return block didn't take.
- **Fail (L-02 relapse):** `AttributeError: type object 'EventsRequestBuilder' has no attribute
  'EventsRequestBuilderPostQueryParameters'` → A1 didn't apply.
- **Special case — a clean `403`:** the code is **correct** and `Calendars.ReadWrite` simply
  isn't granted (B-02). That's an Azure admin task, not a code bug. Confirm it with the curl
  permission test in `FIX-01` §2.3, then request the grant. Record it as "blocked on B-02", not
  as a failure.

### T7.4 — One real colleague

Only after T7.1–T7.3 all pass. **Tell them first** — they're about to get a test invite.

**Prompt:**
```
Draft a meeting invite to {{internal colleague}} for tomorrow at 3pm, 15 minutes,
subject "MCP integration test — please ignore", agenda: ignore, this is a system test.
```
then, after checking the preview:
```
Yes, send it.
```

- **Pass:** They receive it; your confirmation says sent.
- **Cleanup:** delete the event in Outlook. Note that this sends them a **cancellation** — the
  confirmation text says so, which is itself part of what A3 fixed.

---

## Round 8 — Charts

### T8.1 — Pie

**Prompt:**
```
Chart my email volume by folder as a pie chart.
```

- **Pass:** A Mermaid pie chart renders in LibreChat.
- **Cosmetic (F-13/F5):** values may show as `120.0` rather than `120` in the pie branch —
  known, low severity, documented not masked.

### T8.2 — Bar with an axis label

**Prompt:**
```
Chart my emails per day for the last week as a bar chart, with the x-axis labelled "Day".
```

- **Pass:** The chart renders **and carries the x-axis label**. Before F5, `x_label` was
  accepted and documented but silently discarded — only `y-axis` was ever emitted.

---

## Post-run terminal checks

Run these after finishing the rounds — several verify Block E and F, which have no
LibreChat-visible behaviour.

```bash
cd /opt/FiGPT_OutlookMCP

# E1 — persistent logs. Should have grown during the test run.
ls -la logs/
tail -30 logs/*.log

# F3 — the audit log must NOT contain task titles derived from email subjects
grep -i "title" logs/audit.log | tail -20
```

- **Pass (F3):** no user task text / email subjects in the `title` field.
- **Fail:** real subjects visible → F3 didn't apply. `_sanitise_inputs` allowlisted `title`
  while its own comment excluded `subject` as sensitive — but `add_task_todo` passes the
  subject-derived text *as* `title` (F-24).

```bash
# E2 — secrets must be ignored. Both should print the file path.
git check-ignore -v .env
git check-ignore -v bin/token.json
git check-ignore -v logs/audit.log

# And nothing sensitive staged or tracked:
git status --short
git ls-files | grep -E "\.env$|token\.json|del\.txt|audit\.log"
```

- **Pass:** all three `check-ignore` calls print a rule; the last command prints **nothing**.
- **Fail (F-07):** if `logs/` isn't ignored, real user email addresses are in version control.
  Also confirm `bin/del.txt` — it held a full (expired) Graph JWT.

```bash
# E3 — dependency pinning
grep -c "==" requirements.txt
grep -n "matplotlib" requirements.txt
```

- **Pass:** pins present; **no matplotlib** (dead since the Mermaid rewrite).

```bash
# E4 — the env template must not reproduce the HTTP 421 bug
grep -n "FASTMCP_HOST\|MCP_TRANSPORT\|GRAPH_API_SCOPES" .env.example
```

- **Pass:** `FASTMCP_HOST=0.0.0.0` **active, not commented out**; no duplicate
  `MCP_TRANSPORT` or `GRAPH_API_SCOPES` keys. Provisioning a second environment from the old
  template reproduced the 421 saga exactly (F-20).

```bash
# E5 — the test runner must actually execute
bash run_all_test.sh
```

- **Pass:** it runs and reports 90 passed. Before E5 it couldn't execute at all — smart quotes
  (`'tools_registered'`) and en-dashes (`–tb=short`) instead of ASCII (F-09).

---

## Results table

Copy this back to me filled in.

| # | Test | Pass / Fail / Blocked | Notes |
|---|---|---|---|
| Pre-flight | `tools_registered` count | | expect **36** |
| T0.1 | Tool list, no `create_draft_invite` | | |
| T1.1 | Inbox listing, 🔵 column | | |
| T1.2 | "Which are unread?" | | **the L-01 sentence** |
| T1.3 | Paged listing, 🔵 column | | |
| T1.4 | 25 rows, table intact | | |
| T2.1 | Search + 🔵 column | | |
| T2.2 | Unread search results | | |
| T2.3 | Keyword + date range | | **C8 tz crash** |
| T2.4 | Advanced search | | |
| T2.5 | Semantic search | | note first-call latency |
| T3.1 | Draft listing, odd draft | | |
| T3.2 | Compose gate holds | | |
| T3.3 | "Draft Saved", not invite text | | **F-14** |
| T3.4 | Reply threaded | | **L-03** |
| T4.1 | Follow-ups exclude replied | | **D1 — the real test** |
| T4.2 | Known-replied email | | |
| T4.3 | Empty thread | | **C9** |
| T4.4 | Follow-up threaded | | **F-18** |
| T5.1 | MOM clean, no leaked source | | **B1** |
| T5.2 | MOM → HTML | | **F2** |
| T5.3 | Tasks, governance once | | **B2** |
| T5.4 | Task listing | | |
| T5.5 | Planner honest failure | | B-01 |
| T6.1 | Calendar listing | | |
| T6.2 | Availability, internal | | B-03 |
| T7.1 | **Invite gate holds** | | **most important test here** |
| T7.2 | Gate holds on refusal | | |
| T7.3 | Send to self, past-tense text | | or **blocked on B-02** |
| T7.4 | Send to colleague | | |
| T8.1 | Pie chart | | |
| T8.2 | Bar chart with x-label | | **F5** |
| Post | logs/ growing | | E1 |
| Post | audit log has no titles | | F3 |
| Post | secrets gitignored | | E2 / F-07 |
| Post | `.env.example` correct | | E4 |
| Post | `run_all_test.sh` runs | | E5 |

---

## Priority if you're short on time

If you can only run a subset today, run these seven. They cover every **High** finding and the
two behaviours that were wrong rather than merely broken:

1. **T1.2** — the original L-01 complaint
2. **T2.3** — the C8 timezone crash
3. **T3.3** — F-14, the wrong confirmation text
4. **T4.1** — D1, follow-ups that respect replies
5. **T7.1** — the invite gate holds *(critical)*
6. **T7.3** — the send path is honest about having sent *(F-17)*
7. Post-run **E2** — secrets gitignored

---

## What to send back

1. The filled results table (or just the failures)
2. `curl -s http://localhost:8000/health | python3 -m json.tool`
3. For any failure: LibreChat's response **and** `tail -50` of the server terminal
4. The outcome of T7.3 specifically — pass, or a clean 403 (B-02)
