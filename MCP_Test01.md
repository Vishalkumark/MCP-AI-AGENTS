Perfect. Now let’s capture the token and run the full test. Run this complete script in your terminal:

python -c "
import msal
from config.settings import settings

app = msal.PublicClientApplication(
    settings.AZURE_CLIENT_ID,
    authority=settings.AZURE_AUTHORITY
)

flow = app.initiate_device_flow(scopes=settings.GRAPH_API_SCOPES)
result = app.acquire_token_by_device_flow(flow)

if 'access_token' in result:
    print('TOKEN ACQUIRED SUCCESSFULLY')
    print(result['access_token'])
else:
    print('FAILED:', result.get('error_description'))
"


It will print a long token string. Copy the entire token — it starts with eyJ and is several hundred characters long.
Then create a new file at the project root called quick_test.py and paste this in:

"""
quick_test.py
=============
Standalone terminal test — validates the full MCP server chain
using a real Microsoft OAuth token, without needing LibreChat.
Delete this file after validation is complete.

Usage:
    python quick_test.py <your-access-token>
"""

import asyncio
import httpx
import sys
import json

# ── Configuration ──────────────────────────────────────────────
MCP_URL = "http://localhost:8000/mcp"
TOKEN   = sys.argv[1] if len(sys.argv) > 1 else ""

if not TOKEN:
    print("Usage: python quick_test.py <access-token>")
    sys.exit(1)

HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json",
}


# ── MCP call helper ─────────────────────────────────────────────
async def call_tool(client: httpx.AsyncClient, tool_name: str, arguments: dict = {}) -> dict:
    """Send one MCP tool call and return the result."""

    payload = {
        "jsonrpc": "2.0",
        "id":      1,
        "method":  "tools/call",
        "params":  {
            "name":      tool_name,
            "arguments": arguments,
        }
    }

    response = await client.post(MCP_URL, headers=HEADERS, json=payload)
    response.raise_for_status()
    data = response.json()

    # MCP returns results inside content[0].text as a JSON string
    raw = data.get("result", {}).get("content", [{}])[0].get("text", "{}")
    try:
        return json.loads(raw)
    except Exception:
        return {"raw": raw}


# ── Tests ───────────────────────────────────────────────────────
async def run_tests():
    print("\n" + "="*55)
    print("  outlook-ai-agent-mcp — Terminal Validation")
    print("="*55)

    async with httpx.AsyncClient(timeout=30) as client:

        # ── Test 1: Profile ──────────────────────────────────────
        print("\n[1] get_my_profile")
        result = await call_tool(client, "get_my_profile")
        if result.get("error"):
            print(f"    FAILED — {result['message']}")
        else:
            print(f"    Name  : {result.get('display_name')}")
            print(f"    Email : {result.get('email')}")
            print(f"    Title : {result.get('job_title')}")
            print("    PASSED")

        # ── Test 2: List emails ──────────────────────────────────
        print("\n[2] list_emails (last 3)")
        result = await call_tool(client, "list_emails", {"count": 3})
        if isinstance(result, dict) and result.get("error"):
            print(f"    FAILED — {result['message']}")
            email_id = None
        else:
            emails = result if isinstance(result, list) else []
            for i, m in enumerate(emails, 1):
                print(f"    {i}. [{m.get('received_date','?')[:10]}] "
                      f"{m.get('subject','(no subject)')} "
                      f"— {m.get('sender','?')}")
            email_id = emails[0]["id"] if emails else None
            print("    PASSED")

        # ── Test 3: Read email ───────────────────────────────────
        if email_id:
            print("\n[3] read_email (first result)")
            result = await call_tool(client, "read_email", {"email_id": email_id})
            if result.get("error"):
                print(f"    FAILED — {result['message']}")
            else:
                body_preview = (result.get("body") or "")[:120].replace("\n", " ")
                print(f"    Subject : {result.get('subject')}")
                print(f"    Body    : {body_preview}...")
                print("    PASSED")
        else:
            print("\n[3] read_email — SKIPPED (no email ID from Test 2)")

        # ── Test 4: Search emails ────────────────────────────────
        print("\n[4] search_emails (keyword: 'meeting')")
        result = await call_tool(client, "search_emails", {"keyword": "meeting", "count": 3})
        if isinstance(result, dict) and result.get("error"):
            print(f"    FAILED — {result['message']}")
        else:
            emails = result if isinstance(result, list) else []
            print(f"    Found {len(emails)} result(s)")
            for m in emails:
                print(f"    — {m.get('subject','(no subject)')}")
            print("    PASSED")

        # ── Test 5: Summarise email ──────────────────────────────
        if email_id:
            print("\n[5] summarise_email (first email)")
            result = await call_tool(client, "summarise_email", {"email_id": email_id})
            if result.get("error"):
                print(f"    FAILED — {result['message']}")
            else:
                print(f"    Summary : {result.get('summary','')[:200]}...")
                print("    PASSED")
        else:
            print("\n[5] summarise_email — SKIPPED (no email ID from Test 2)")

        # ── Test 6: Calendar ─────────────────────────────────────
        print("\n[6] list_calendar_events (this week)")
        result = await call_tool(client, "list_calendar_events", {"date_range": "this week"})
        if isinstance(result, dict) and result.get("error"):
            print(f"    FAILED — {result['message']}")
        else:
            events = result if isinstance(result, list) else []
            print(f"    Found {len(events)} event(s)")
            for e in events[:3]:
                print(f"    — {e.get('subject','?')} "
                      f"@ {(e.get('start_time') or '')[:16]}")
            print("    PASSED")

    print("\n" + "="*55)
    print("  Validation complete.")
    print("="*55 + "\n")


asyncio.run(run_tests())


Save quick_test.py, then run it with your token:

python quick_test.py eyJ...your-full-token-here...


Replace eyJ...your-full-token-here... with the full token string printed earlier.
Paste the output and we’ll see exactly which tools are working against your real Outlook data.​​​​​​​​​​​​​​​​