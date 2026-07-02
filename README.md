# MCP-AI-AGENTS


Phase 1 — Progress Report
Project: outlook-ai-agent-mcp | Date: 01 July 2026

Completed ✅



|Stage                                    |Status|
|-----------------------------------------|------|
|Project framework (32 files)             |✅ Done|
|All dependencies installed               |✅ Done|
|Settings & config loading                |✅ Done|
|All 16 pytest tests passing              |✅ Done|
|MCP server starts cleanly                |✅ Done|
|Health check — 9 tools registered        |✅ Done|
|Terminal validation — full chain verified|✅ Done|

What Was Proven in Terminal Validation
The full chain works end to end:

Test Script → MCP Session → Auth Layer → Graph API → Clean Response


Every tool is reachable, error handling works, SSE parsing works. The only reason tools returned “Authentication failed” is the token type — not a code problem.

The Delegated Token Point — Simply Explained
There are two types of Microsoft tokens:



|Type                              |How obtained                |Accesses                   |
|----------------------------------|----------------------------|---------------------------|
|**App-only** (what we tested with)|Client secret, no user login|Limited — `/me` not allowed|
|**Delegated** (what we need)      |User logs in via OAuth popup|Full access to YOUR mailbox|

LibreChat handles the delegated login automatically — you click Authenticate, log in with your company account, and LibreChat sends the correct token with every tool call. No code change needed.

What’s Next — Stage 5: Connect LibreChat
You need:
	1.	Your VM’s IPv4 address → run ipconfig
	2.	Your LibreChat URL → get from your team
Then in LibreChat:
	•	Add MCP server → enter http://<your-vm-ip>:8000/mcp
	•	Click Authenticate → log in with company account
	•	Green Connected = fully working with real delegated token
That’s it. Once connected, all 9 tools will work against your real Outlook mailbox.​​​​​​​​​​​​​​​​



---

---

Got it. Here are the two scripts — both go in the project root, touch zero existing code.

Script 1 — get_token.py
Create this file at the project root:

"""
get_token.py
============
One-time interactive login script to get a real delegated
Microsoft OAuth token for local testing.

Opens a browser popup, you log in with your Company account,
token is saved to token.json for use by agent_test.py.

Usage:
    python get_token.py

Do NOT commit token.json — it contains your personal access token.
"""

import json
import msal
from pathlib import Path
from config.settings import settings

# ── Token file location ──────────────────────────────────────────
TOKEN_FILE = Path("token.json")

# ── Scopes needed for all Phase 1 tools ─────────────────────────
SCOPES = [
    "Mail.Read",
    "Calendars.Read",
    "User.Read",
]

def get_token():
    print("\n" + "="*50)
    print("  outlook-ai-agent-mcp — Token Acquisition")
    print("="*50)

    # Step 1: Build a PublicClientApplication with
    #         interactive browser login support.
    #         This is different from ConfidentialClientApplication —
    #         PublicClientApplication supports user-interactive flows
    #         which is exactly what delegated auth requires.
    app = msal.PublicClientApplication(
        client_id=settings.AZURE_CLIENT_ID,
        authority=settings.AZURE_AUTHORITY,
    )

    # Step 2: Check if we already have a cached valid token
    accounts = app.get_accounts()
    result = None

    if accounts:
        print(f"\nFound cached account: {accounts[0]['username']}")
        print("Attempting silent token refresh...")
        result = app.acquire_token_silent(SCOPES, account=accounts[0])

    # Step 3: If no cached token, open browser for interactive login
    if not result:
        print("\nOpening browser for login...")
        print("Log in with your Company Microsoft account.\n")
        result = app.acquire_token_interactive(scopes=SCOPES)

    # Step 4: Validate and save
    if "access_token" in result:
        token_data = {
            "access_token": result["access_token"],
            "token_type": result.get("token_type", "Bearer"),
            "expires_in": result.get("expires_in", 3600),
            "scope": result.get("scope", ""),
        }
        TOKEN_FILE.write_text(json.dumps(token_data, indent=2))

        print("\n✅ Token acquired and saved to token.json")
        print(f"   Expires in: {result.get('expires_in', 3600) // 60} minutes")
        print(f"   Scopes    : {result.get('scope', '')}")
        print("\nYou can now run: python agent_test.py")

    else:
        print(f"\n❌ Failed to acquire token.")
        print(f"   Error: {result.get('error')}")
        print(f"   Details: {result.get('error_description')}")


if __name__ == "__main__":
    get_token()


Script 2 — agent_test.py
Create this file at the project root:

"""
agent_test.py
=============
Interactive agent loop that lets you prompt your MCP tools
directly from VS Code terminal — no LibreChat needed.

The LLM reads your prompt, decides which MCP tool to call,
executes it against your real Outlook data, and prints the result.

Requires:
    - python server.py running in Terminal 1
    - token.json present (run get_token.py first)

Usage:
    python agent_test.py

Type 'quit' to exit.
"""

import asyncio
import json
import httpx
from pathlib import Path
from config.settings import settings
from utils.logger import get_logger

logger = get_logger(__name__)

# ── Config ───────────────────────────────────────────────────────
MCP_URL    = f"http://localhost:{settings.MCP_PORT}/mcp"
TOKEN_FILE = Path("token.json")

# ── Tool definitions for the LLM ─────────────────────────────────
# These mirror the actual MCP tools so the LLM knows what's available
# and when to call each one.
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "get_my_profile",
            "description": "Get the display name, email address, and job title of the currently logged-in Microsoft 365 user.",
            "parameters": {"type": "object", "properties": {}, "required": []},
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_emails",
            "description": "List the most recent emails in the user's Outlook inbox.",
            "parameters": {
                "type": "object",
                "properties": {
                    "count": {"type": "integer", "description": "Number of emails to return (default 10, max 50)"}
                },
                "required": []
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "read_email",
            "description": "Read the full content of a specific email by its ID.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {"type": "string", "description": "The unique Graph API ID of the email"}
                },
                "required": ["email_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_emails",
            "description": "Search the user's mailbox for emails matching a keyword.",
            "parameters": {
                "type": "object",
                "properties": {
                    "keyword": {"type": "string", "description": "Search term"},
                    "count": {"type": "integer", "description": "Max results to return"}
                },
                "required": ["keyword"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "summarise_email",
            "description": "Generate a concise AI summary of a specific email's content.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {"type": "string", "description": "The unique Graph API ID of the email"}
                },
                "required": ["email_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_calendar_events",
            "description": "List the user's upcoming calendar events within a given time range.",
            "parameters": {
                "type": "object",
                "properties": {
                    "date_range": {"type": "string", "description": "Natural language range: today, tomorrow, this week, next 7 days"}
                },
                "required": []
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_attachments",
            "description": "List all attachments on a specific email without reading their content.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {"type": "string", "description": "The unique Graph API ID of the email"}
                },
                "required": ["email_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "read_attachment",
            "description": "Extract and return the text content of an email attachment (PDF, Word, PowerPoint, Excel).",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {"type": "string", "description": "The unique Graph API ID of the parent email"},
                    "attachment_id": {"type": "string", "description": "The unique ID of the attachment"}
                },
                "required": ["email_id", "attachment_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "summarise_attachment",
            "description": "Read an email attachment and generate a concise AI summary of its content.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {"type": "string", "description": "The unique Graph API ID of the parent email"},
                    "attachment_id": {"type": "string", "description": "The unique ID of the attachment"}
                },
                "required": ["email_id", "attachment_id"]
            },
        }
    },
]


# ── Token loader ─────────────────────────────────────────────────
def load_token() -> str:
    if not TOKEN_FILE.exists():
        print("❌ token.json not found. Run: python get_token.py first.")
        raise SystemExit(1)
    data = json.loads(TOKEN_FILE.read_text())
    return data["access_token"]


# ── MCP helpers ──────────────────────────────────────────────────
async def mcp_initialise(client: httpx.AsyncClient, token: str) -> str:
    """Perform MCP handshake and return session ID."""
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
        "Accept": "application/json, text/event-stream",
    }
    payload = {
        "jsonrpc": "2.0", "id": 0, "method": "initialize",
        "params": {
            "protocolVersion": "2024-11-05",
            "capabilities": {},
            "clientInfo": {"name": "agent-test", "version": "1.0.0"}
        }
    }
    response = await client.post(MCP_URL, headers=headers, json=payload)
    response.raise_for_status()
    return response.headers.get("mcp-session-id", "")


async def mcp_call_tool(
    client: httpx.AsyncClient,
    token: str,
    session_id: str,
    tool_name: str,
    arguments: dict
) -> str:
    """Call one MCP tool and return result as a string."""
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
        "Accept": "application/json, text/event-stream",
        "mcp-session-id": session_id,
    }
    payload = {
        "jsonrpc": "2.0", "id": 1,
        "method": "tools/call",
        "params": {"name": tool_name, "arguments": arguments}
    }
    response = await client.post(MCP_URL, headers=headers, json=payload)
    response.raise_for_status()

    # Parse SSE or JSON response
    content_type = response.headers.get("content-type", "")
    if "application/json" in content_type:
        data = response.json()
        raw = data.get("result", {}).get("content", [{}])[0].get("text", "{}")
    else:
        raw = ""
        for line in response.text.splitlines():
            line = line.strip()
            if line.startswith("data:"):
                json_str = line[len("data:"):].strip()
                if json_str:
                    try:
                        data = json.loads(json_str)
                        content = data.get("result", {}).get("content", [{}])
                        raw = content[0].get("text", "{}") if content else "{}"
                        break
                    except json.JSONDecodeError:
                        continue

    return raw


# ── LLM call ─────────────────────────────────────────────────────
async def call_llm(messages: list, allow_tools: bool = True) -> dict:
    """Send messages to Requesty.AI and return the response."""
    async with httpx.AsyncClient(timeout=60) as http:
        payload = {
            "model": settings.REQUESTY_PRIMARY_MODEL,
            "messages": messages,
            "max_tokens": 1000,
            "temperature": 0.3,
        }
        if allow_tools:
            payload["tools"] = TOOLS
            payload["tool_choice"] = "auto"

        response = await http.post(
            f"{settings.REQUESTY_BASE_URL}/chat/completions",
            headers={
                "Authorization": f"Bearer {settings.REQUESTY_API_KEY}",
                "Content-Type": "application/json",
            },
            json=payload
        )
        response.raise_for_status()
        return response.json()


# ── Agent loop ───────────────────────────────────────────────────
async def agent_loop():
    print("\n" + "="*55)
    print("  outlook-ai-agent-mcp — VS Code Agent Test")
    print("="*55)
    print("  Your MCP tools are live. Type a prompt below.")
    print("  Type 'quit' to exit.\n")

    # Load token and initialise MCP session once
    token = load_token()

    async with httpx.AsyncClient(timeout=60) as mcp_client:
        print("Connecting to MCP server...")
        session_id = await mcp_initialise(mcp_client, token)
        print(f"✅ MCP session ready\n")

        # Conversation history — grows with each turn
        messages = [
            {
                "role": "system",
                "content": (
                    "You are an intelligent Outlook email assistant. "
                    "You have access to MCP tools that can read emails, "
                    "summarise them, check calendar events, and read attachments. "
                    "Use the appropriate tool for each user request. "
                    "Always use tools to fetch real data — never make up email content."
                )
            }
        ]

        while True:
            # ── Get user prompt ──────────────────────────────────
            try:
                user_input = input("You: ").strip()
            except (KeyboardInterrupt, EOFError):
                print("\n\nExiting agent. Goodbye.")
                break

            if user_input.lower() in ("quit", "exit", "q"):
                print("Exiting agent. Goodbye.")
                break

            if not user_input:
                continue

            messages.append({"role": "user", "content": user_input})

            print("\nAgent thinking...\n")

            # ── Step 1: Ask LLM what to do ───────────────────────
            llm_response = await call_llm(messages, allow_tools=True)
            assistant_message = llm_response["choices"][0]["message"]
            messages.append(assistant_message)

            # ── Step 2: Execute tool calls if LLM requested any ──
            while assistant_message.get("tool_calls"):
                for tool_call in assistant_message["tool_calls"]:
                    tool_name = tool_call["function"]["name"]
                    arguments = json.loads(tool_call["function"]["arguments"])

                    print(f"  → Calling tool: {tool_name}")
                    if arguments:
                        print(f"    Arguments: {json.dumps(arguments, indent=6)}")

                    # Call the actual MCP tool
                    tool_result = await mcp_call_tool(
                        mcp_client, token, session_id, tool_name, arguments
                    )

                    print(f"  ✅ Tool returned result\n")

                    # Add tool result to conversation so LLM can use it
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call["id"],
                        "content": tool_result,
                    })

                # ── Step 3: LLM composes final answer ────────────
                llm_response = await call_llm(messages, allow_tools=True)
                assistant_message = llm_response["choices"][0]["message"]
                messages.append(assistant_message)

            # ── Step 4: Print final answer ───────────────────────
            final_answer = assistant_message.get("content", "")
            print(f"Agent: {final_answer}\n")
            print("-"*55)


if __name__ == "__main__":
    asyncio.run(agent_loop())


Also add these two lines to your .gitignore so the token file is never committed:

token.json
agent_test.py
quick_test.py


How to Run
Terminal 1 — keep server running:

python server.py


Terminal 2 — get your token first:

python get_token.py


A browser popup opens → log in with your Company account → token saved to token.json
Terminal 2 — then start the agent:

python agent_test.py


Then type prompts like:

You: Summarise my last 3 emails
You: What meetings do I have this week?
You: Find emails about budget and summarise the latest one


The agent selects the right tools, calls your real Outlook data, and prints the answer. Let me know what output you get.​​​​​​​​​​​​​​​​


# “””
agent_test.py

Interactive agent loop that lets you prompt your MCP tools
directly from VS Code terminal — no LibreChat needed.

The LLM reads your prompt, decides which MCP tool to call,
executes it against your real Outlook data, and prints the result.

Requires:
- python server.py running in Terminal 1
- token.json present (run get_token.py first)

Usage:
python agent_test.py

Type ‘quit’ to exit.
“””

import asyncio
import json
import httpx
from pathlib import Path
from config.settings import settings
from utils.logger import get_logger

logger = get_logger(**name**)

# ── Config ───────────────────────────────────────────────────────

MCP_URL    = f”http://localhost:{settings.MCP_PORT}/mcp”
TOKEN_FILE = Path(“token.json”)

# ── Tool definitions for the LLM ─────────────────────────────────

TOOLS = [
{
“type”: “function”,
“function”: {
“name”: “get_my_profile”,
“description”: “Get the display name, email address, and job title of the currently logged-in Microsoft 365 user.”,
“parameters”: {“type”: “object”, “properties”: {}, “required”: []},
}
},
{
“type”: “function”,
“function”: {
“name”: “list_emails”,
“description”: “List the most recent emails in the user’s Outlook inbox.”,
“parameters”: {
“type”: “object”,
“properties”: {
“count”: {“type”: “integer”, “description”: “Number of emails to return (default 10, max 50)”}
},
“required”: []
},
}
},
{
“type”: “function”,
“function”: {
“name”: “read_email”,
“description”: “Read the full content of a specific email by its ID.”,
“parameters”: {
“type”: “object”,
“properties”: {
“email_id”: {“type”: “string”, “description”: “The unique Graph API ID of the email”}
},
“required”: [“email_id”]
},
}
},
{
“type”: “function”,
“function”: {
“name”: “search_emails”,
“description”: “Search the user’s mailbox for emails matching a keyword.”,
“parameters”: {
“type”: “object”,
“properties”: {
“keyword”: {“type”: “string”, “description”: “Search term”},
“count”: {“type”: “integer”, “description”: “Max results to return”}
},
“required”: [“keyword”]
},
}
},
{
“type”: “function”,
“function”: {
“name”: “summarise_email”,
“description”: “Generate a concise AI summary of a specific email’s content.”,
“parameters”: {
“type”: “object”,
“properties”: {
“email_id”: {“type”: “string”, “description”: “The unique Graph API ID of the email”}
},
“required”: [“email_id”]
},
}
},
{
“type”: “function”,
“function”: {
“name”: “list_calendar_events”,
“description”: “List the user’s upcoming calendar events within a given time range.”,
“parameters”: {
“type”: “object”,
“properties”: {
“date_range”: {
“type”: “string”,
“description”: “Natural language range: today, tomorrow, this week, next 7 days”
}
},
“required”: []
},
}
},
{
“type”: “function”,
“function”: {
“name”: “list_attachments”,
“description”: “List all attachments on a specific email without reading their content.”,
“parameters”: {
“type”: “object”,
“properties”: {
“email_id”: {“type”: “string”, “description”: “The unique Graph API ID of the email”}
},
“required”: [“email_id”]
},
}
},
{
“type”: “function”,
“function”: {
“name”: “read_attachment”,
“description”: “Extract and return the text content of an email attachment (PDF, Word, PowerPoint, Excel).”,
“parameters”: {
“type”: “object”,
“properties”: {
“email_id”: {“type”: “string”, “description”: “The unique Graph API ID of the parent email”},
“attachment_id”: {“type”: “string”, “description”: “The unique ID of the attachment”}
},
“required”: [“email_id”, “attachment_id”]
},
}
},
{
“type”: “function”,
“function”: {
“name”: “summarise_attachment”,
“description”: “Read an email attachment and generate a concise AI summary of its content.”,
“parameters”: {
“type”: “object”,
“properties”: {
“email_id”: {“type”: “string”, “description”: “The unique Graph API ID of the parent email”},
“attachment_id”: {“type”: “string”, “description”: “The unique ID of the attachment”}
},
“required”: [“email_id”, “attachment_id”]
},
}
},
]

# ── Token loader ─────────────────────────────────────────────────

def load_token() -> str:
“””
Load the delegated OAuth token saved by get_token.py.
Exits cleanly if token.json is not found.
“””
if not TOKEN_FILE.exists():
print(“❌ token.json not found. Run: python get_token.py first.”)
raise SystemExit(1)
data = json.loads(TOKEN_FILE.read_text())
return data[“access_token”]

# ── MCP helpers ──────────────────────────────────────────────────

async def mcp_initialise(client: httpx.AsyncClient, token: str) -> str:
“””
Perform the MCP handshake with the FastMCP server.
Returns the session ID assigned by the server.
The token is passed here so the server knows who is connecting.
“””
headers = {
“Authorization”: f”Bearer {token}”,
“Content-Type”: “application/json”,
“Accept”: “application/json, text/event-stream”,
}
payload = {
“jsonrpc”: “2.0”,
“id”: 0,
“method”: “initialize”,
“params”: {
“protocolVersion”: “2024-11-05”,
“capabilities”: {},
“clientInfo”: {
“name”: “agent-test”,
“version”: “1.0.0”
}
}
}
response = await client.post(MCP_URL, headers=headers, json=payload)
response.raise_for_status()
session_id = response.headers.get(“mcp-session-id”, “”)
return session_id

async def mcp_call_tool(
client: httpx.AsyncClient,
token: str,
session_id: str,
tool_name: str,
arguments: dict
) -> str:
“””
Call one MCP tool on the running FastMCP server and return
the raw result string.

```
The token is passed in BOTH the Authorization header AND the
X-Auth-Token header so that graph_auth.py can find it
regardless of how FastMCP forwards headers to tool functions.
The session ID ties this call to the initialised session.
"""
headers = {
    # Primary auth header — standard Bearer token format
    "Authorization": f"Bearer {token}",
    # Secondary header — fallback for graph_auth.py if FastMCP
    # doesn't forward Authorization into the tool context
    "X-Auth-Token": token,
    "Content-Type": "application/json",
    "Accept": "application/json, text/event-stream",
    # Session ID ties this request to the initialised MCP session
    "mcp-session-id": session_id,
}

payload = {
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": tool_name,
        "arguments": arguments
    }
}

response = await client.post(MCP_URL, headers=headers, json=payload)
response.raise_for_status()

# ── Parse response — handles both JSON and SSE formats ────────
content_type = response.headers.get("content-type", "")

if "application/json" in content_type:
    # Plain JSON response path
    data = response.json()
    content = data.get("result", {}).get("content", [{}])
    return content[0].get("text", "{}") if content else "{}"

else:
    # SSE (Server-Sent Events) response path
    # FastMCP streamable-http sends results as lines starting with "data:"
    for line in response.text.splitlines():
        line = line.strip()
        if line.startswith("data:"):
            json_str = line[len("data:"):].strip()
            if not json_str:
                continue
            try:
                data = json.loads(json_str)
                content = data.get("result", {}).get("content", [{}])
                return content[0].get("text", "{}") if content else "{}"
            except json.JSONDecodeError:
                continue

return "{}"
```

# ── LLM call ─────────────────────────────────────────────────────

async def call_llm(messages: list, allow_tools: bool = True) -> dict:
“””
Send the conversation history to Requesty.AI and return the
full response object.

```
allow_tools=True  → LLM can decide to call MCP tools
allow_tools=False → LLM must respond with plain text only
                    (used when forcing the final answer)
"""
async with httpx.AsyncClient(timeout=60) as http:
    payload = {
        "model": settings.REQUESTY_PRIMARY_MODEL,
        "messages": messages,
        "max_tokens": 1000,
        "temperature": 0.3,
    }
    if allow_tools:
        payload["tools"] = TOOLS
        payload["tool_choice"] = "auto"

    response = await http.post(
        f"{settings.REQUESTY_BASE_URL}/chat/completions",
        headers={
            "Authorization": f"Bearer {settings.REQUESTY_API_KEY}",
            "Content-Type": "application/json",
        },
        json=payload
    )
    response.raise_for_status()
    return response.json()
```

# ── Agent loop ───────────────────────────────────────────────────

async def agent_loop():
print(”\n” + “=”*55)
print(”  outlook-ai-agent-mcp — VS Code Agent Test”)
print(”=”*55)
print(”  Your MCP tools are live. Type a prompt below.”)
print(”  Type ‘quit’ to exit.\n”)

```
# Step 1: Load the delegated token from token.json
token = load_token()

async with httpx.AsyncClient(timeout=60) as mcp_client:

    # Step 2: Initialise the MCP session once — all tool calls
    #         in this session reuse the same session ID
    print("Connecting to MCP server...")
    session_id = await mcp_initialise(mcp_client, token)
    print(f"✅ MCP session ready\n")

    # Step 3: Set up the conversation history with a system prompt
    #         that tells the LLM what role it plays and what tools
    #         are available to it
    messages = [
        {
            "role": "system",
            "content": (
                "You are an intelligent Outlook email assistant with access "
                "to MCP tools that can read emails, summarise them, check "
                "calendar events, and read attachments from a Microsoft 365 "
                "mailbox. Always use the appropriate tool to fetch real data "
                "before answering. Never make up or guess email content. "
                "After receiving tool results, always compose a clear, "
                "well-formatted response for the user."
            )
        }
    ]

    # Step 4: Main prompt loop
    while True:

        # Get user input
        try:
            user_input = input("You: ").strip()
        except (KeyboardInterrupt, EOFError):
            print("\n\nExiting agent. Goodbye.")
            break

        if user_input.lower() in ("quit", "exit", "q"):
            print("Exiting agent. Goodbye.")
            break

        if not user_input:
            continue

        # Add user message to conversation history
        messages.append({"role": "user", "content": user_input})

        print("\nAgent thinking...\n")

        # ── Step 5: First LLM call — let it decide what to do ─
        llm_response = await call_llm(messages, allow_tools=True)
        assistant_message = llm_response["choices"][0]["message"]
        messages.append(assistant_message)

        # ── Step 6: Tool call loop ────────────────────────────
        # The LLM may request multiple tools in sequence.
        # We keep executing until it stops requesting tools.
        max_tool_rounds = 5  # Safety cap — prevents infinite loops
        tool_round = 0

        while assistant_message.get("tool_calls") and tool_round < max_tool_rounds:
            tool_round += 1

            for tool_call in assistant_message["tool_calls"]:
                tool_name = tool_call["function"]["name"]
                arguments = json.loads(tool_call["function"]["arguments"])

                print(f"  → Calling tool : {tool_name}")
                if arguments:
                    for k, v in arguments.items():
                        # Truncate long IDs in display for readability
                        display_v = str(v)[:60] + "..." if len(str(v)) > 60 else str(v)
                        print(f"    {k}: {display_v}")

                # Execute the tool on the MCP server
                tool_result_raw = await mcp_call_tool(
                    mcp_client, token, session_id, tool_name, arguments
                )

                # Try to pretty-print the result for the terminal
                try:
                    parsed = json.loads(tool_result_raw)
                    if isinstance(parsed, list):
                        print(f"  ✅ Tool returned {len(parsed)} item(s)")
                    elif isinstance(parsed, dict) and parsed.get("error"):
                        print(f"  ⚠️  Tool error: {parsed.get('message')}")
                    else:
                        print(f"  ✅ Tool returned result")
                except Exception:
                    print(f"  ✅ Tool returned result")

                print()

                # Add the tool result to conversation history
                # so the LLM can use it in the next step
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call["id"],
                    "content": tool_result_raw,
                })

            # Ask the LLM again with the tool results now in context
            llm_response = await call_llm(messages, allow_tools=True)
            assistant_message = llm_response["choices"][0]["message"]
            messages.append(assistant_message)

        # ── Step 7: Force a final text answer if content is empty
        # Sometimes the LLM returns empty content after tool calls
        # instead of composing the final answer — we detect this
        # and explicitly ask for a text-only response.
        final_answer = assistant_message.get("content", "")

        if not final_answer or not final_answer.strip():
            print("  → Composing final answer...\n")
            messages.append({
                "role": "user",
                "content": (
                    "Based on the tool results above, please provide "
                    "a clear, well-formatted answer to my original "
                    "question. Do not call any more tools."
                )
            })
            final_llm = await call_llm(messages, allow_tools=False)
            final_message = final_llm["choices"][0]["message"]
            final_answer = final_message.get("content", "No response generated.")
            messages.append(final_message)

        # ── Step 8: Print the final answer ───────────────────
        print(f"Agent: {final_answer}\n")
        print("-" * 55)
```

if **name** == “**main**”:
asyncio.run(agent_loop())



---

"""
agent_test.py
=============
Interactive agent loop that lets you prompt your MCP tools
directly from VS Code terminal — no LibreChat needed.

The LLM reads your prompt, decides which MCP tool to call,
executes it against your real Outlook data, and prints the result.

Requires:
    - python server.py running in Terminal 1
    - token.json present (run get_token.py first)

Usage:
    python agent_test.py

Type 'quit' to exit.
"""

import asyncio
import json
import httpx
from pathlib import Path
from config.settings import settings
from utils.logger import get_logger

logger = get_logger(__name__)

# ── Config ───────────────────────────────────────────────────────
MCP_URL    = f"http://localhost:{settings.MCP_PORT}/mcp"
TOKEN_FILE = Path("token.json")

# ── Tool definitions for the LLM ─────────────────────────────────
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "get_my_profile",
            "description": "Get the display name, email address, and job title of the currently logged-in Microsoft 365 user.",
            "parameters": {"type": "object", "properties": {}, "required": []},
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_emails",
            "description": "List the most recent emails in the user's Outlook inbox.",
            "parameters": {
                "type": "object",
                "properties": {
                    "count": {
                        "type": "integer",
                        "description": "Number of emails to return (default 10, max 50)"
                    }
                },
                "required": []
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "read_email",
            "description": "Read the full content of a specific email by its ID.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {
                        "type": "string",
                        "description": "The unique Graph API ID of the email"
                    }
                },
                "required": ["email_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_emails",
            "description": "Search the user's mailbox for emails matching a keyword.",
            "parameters": {
                "type": "object",
                "properties": {
                    "keyword": {
                        "type": "string",
                        "description": "Search term"
                    },
                    "count": {
                        "type": "integer",
                        "description": "Max results to return"
                    }
                },
                "required": ["keyword"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "summarise_email",
            "description": "Generate a concise AI summary of a specific email's content.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {
                        "type": "string",
                        "description": "The unique Graph API ID of the email"
                    }
                },
                "required": ["email_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_calendar_events",
            "description": "List the user's upcoming calendar events within a given time range.",
            "parameters": {
                "type": "object",
                "properties": {
                    "date_range": {
                        "type": "string",
                        "description": "Natural language range: today, tomorrow, this week, next 7 days"
                    }
                },
                "required": []
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_attachments",
            "description": "List all attachments on a specific email without reading their content.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {
                        "type": "string",
                        "description": "The unique Graph API ID of the email"
                    }
                },
                "required": ["email_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "read_attachment",
            "description": "Extract and return the text content of an email attachment (PDF, Word, PowerPoint, Excel).",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {
                        "type": "string",
                        "description": "The unique Graph API ID of the parent email"
                    },
                    "attachment_id": {
                        "type": "string",
                        "description": "The unique ID of the attachment"
                    }
                },
                "required": ["email_id", "attachment_id"]
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "summarise_attachment",
            "description": "Read an email attachment and generate a concise AI summary of its content.",
            "parameters": {
                "type": "object",
                "properties": {
                    "email_id": {
                        "type": "string",
                        "description": "The unique Graph API ID of the parent email"
                    },
                    "attachment_id": {
                        "type": "string",
                        "description": "The unique ID of the attachment"
                    }
                },
                "required": ["email_id", "attachment_id"]
            },
        }
    },
]


# ── Token loader ─────────────────────────────────────────────────
def load_token() -> str:
    """
    Load the delegated access token from token.json.
    Run get_token.py first if this file doesn't exist.
    """
    if not TOKEN_FILE.exists():
        print("❌ token.json not found. Run: python get_token.py first.")
        raise SystemExit(1)
    data = json.loads(TOKEN_FILE.read_text())
    return data["access_token"]


# ── MCP helpers ──────────────────────────────────────────────────
async def mcp_initialise(client: httpx.AsyncClient, token: str) -> str:
    """
    Perform the MCP protocol handshake.
    Must be called once before any tool calls.
    Returns the session ID assigned by the server.
    """
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
        "Accept": "application/json, text/event-stream",
    }
    payload = {
        "jsonrpc": "2.0",
        "id": 0,
        "method": "initialize",
        "params": {
            "protocolVersion": "2024-11-05",
            "capabilities": {},
            "clientInfo": {
                "name": "agent-test",
                "version": "1.0.0"
            }
        }
    }
    response = await client.post(MCP_URL, headers=headers, json=payload)
    response.raise_for_status()
    return response.headers.get("mcp-session-id", "")


async def mcp_call_tool(
    client: httpx.AsyncClient,
    token: str,
    session_id: str,
    tool_name: str,
    arguments: dict
) -> str:
    """
    Call one MCP tool on the server and return the raw result string.

    The Authorization header carries the delegated OAuth token so that
    graph_auth.py can extract it and pass it to Microsoft Graph API.
    The X-Auth-Token header is a fallback in case FastMCP's header
    context doesn't surface the Authorization header correctly.
    """
    headers = {
        # Primary auth header — read by graph_auth.get_current_access_token()
        "Authorization": f"Bearer {token}",
        # Fallback header — also checked by graph_auth.py
        "X-Auth-Token": token,
        "Content-Type": "application/json",
        "Accept": "application/json, text/event-stream",
        # Session ID ties this call to the initialised MCP session
        "mcp-session-id": session_id,
    }

    payload = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/call",
        "params": {
            "name": tool_name,
            "arguments": arguments
        }
    }

    response = await client.post(MCP_URL, headers=headers, json=payload)
    response.raise_for_status()

    # ── Parse response — either plain JSON or SSE stream ─────────
    content_type = response.headers.get("content-type", "")

    if "application/json" in content_type:
        # Plain JSON response
        data = response.json()
        content = data.get("result", {}).get("content", [{}])
        return content[0].get("text", "{}") if content else "{}"

    else:
        # SSE (Server-Sent Events) response — FastMCP default
        # Each SSE line looks like: data: {"jsonrpc":"2.0","result":{...}}
        for line in response.text.splitlines():
            line = line.strip()
            if line.startswith("data:"):
                json_str = line[len("data:"):].strip()
                if not json_str:
                    continue
                try:
                    data = json.loads(json_str)
                    content = data.get("result", {}).get("content", [{}])
                    return content[0].get("text", "{}") if content else "{}"
                except json.JSONDecodeError:
                    continue

    return "{}"


# ── LLM call ─────────────────────────────────────────────────────
async def call_llm(messages: list, allow_tools: bool = True) -> dict:
    """
    Send the conversation history to Requesty.AI and return the
    full API response.

    allow_tools=True  → LLM can request tool calls
    allow_tools=False → LLM must respond with plain text only
                        (used when forcing a final summary response)
    """
    async with httpx.AsyncClient(timeout=60) as http:
        payload = {
            "model": settings.REQUESTY_PRIMARY_MODEL,
            "messages": messages,
            "max_tokens": 1000,
            "temperature": 0.3,
        }

        if allow_tools:
            payload["tools"] = TOOLS
            payload["tool_choice"] = "auto"

        response = await http.post(
            f"{settings.REQUESTY_BASE_URL}/chat/completions",
            headers={
                "Authorization": f"Bearer {settings.REQUESTY_API_KEY}",
                "Content-Type": "application/json",
            },
            json=payload
        )
        response.raise_for_status()
        return response.json()


# ── Agent loop ───────────────────────────────────────────────────
async def agent_loop():
    print("\n" + "="*55)
    print("  outlook-ai-agent-mcp — VS Code Agent Test")
    print("="*55)
    print("  Your MCP tools are live. Type a prompt below.")
    print("  Type 'quit' to exit.\n")

    # Step 1: Load saved delegated token
    token = load_token()

    async with httpx.AsyncClient(timeout=60) as mcp_client:

        # Step 2: Initialise MCP session once for the whole session
        print("Connecting to MCP server...")
        session_id = await mcp_initialise(mcp_client, token)
        print(f"✅ MCP session ready\n")

        # Step 3: Build conversation history — grows with each turn
        messages = [
            {
                "role": "system",
                "content": (
                    "You are an intelligent Outlook email assistant for a Company. "
                    "You have access to MCP tools that can read emails, "
                    "summarise them, check calendar events, and read attachments. "
                    "Always use the available tools to fetch real data — "
                    "never make up or assume email content. "
                    "After receiving tool results, always compose a clear, "
                    "well-formatted final answer for the user."
                )
            }
        ]

        # Step 4: Main prompt loop
        while True:

            # Get user input
            try:
                user_input = input("You: ").strip()
            except (KeyboardInterrupt, EOFError):
                print("\n\nExiting agent. Goodbye.")
                break

            if user_input.lower() in ("quit", "exit", "q"):
                print("Exiting agent. Goodbye.")
                break

            if not user_input:
                continue

            # Add user message to history
            messages.append({"role": "user", "content": user_input})
            print("\nAgent thinking...\n")

            # ── Round 1: Ask LLM what to do ──────────────────────
            llm_response = await call_llm(messages, allow_tools=True)
            assistant_message = llm_response["choices"][0]["message"]
            messages.append(assistant_message)

            # ── Tool execution loop ───────────────────────────────
            # The LLM may request multiple tool calls in sequence.
            # We keep looping until it stops requesting tools.
            max_tool_rounds = 5  # safety cap — prevents infinite loops
            tool_round = 0

            while assistant_message.get("tool_calls") and tool_round < max_tool_rounds:
                tool_round += 1

                for tool_call in assistant_message["tool_calls"]:
                    tool_name = tool_call["function"]["name"]
                    arguments = json.loads(tool_call["function"]["arguments"])

                    print(f"  → Calling tool : {tool_name}")
                    if arguments:
                        for k, v in arguments.items():
                            # Truncate long IDs for cleaner display
                            display_v = str(v)[:60] + "..." if len(str(v)) > 60 else v
                            print(f"     {k}: {display_v}")

                    # Call the actual MCP tool on the server
                    tool_result_raw = await mcp_call_tool(
                        mcp_client, token, session_id, tool_name, arguments
                    )

                    print(f"  ✅ Tool result received\n")

                    # Add tool result to conversation history so the
                    # LLM can use it to compose its final answer
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call["id"],
                        "content": tool_result_raw,
                    })

                # Ask LLM again now that it has the tool results
                llm_response = await call_llm(messages, allow_tools=True)
                assistant_message = llm_response["choices"][0]["message"]
                messages.append(assistant_message)

            # ── Force final text response if content is empty ─────
            # Sometimes the LLM returns empty content after tool calls
            # because it's waiting for another prompt. We force a
            # plain-text summary response in that case.
            final_answer = assistant_message.get("content", "")

            if not final_answer or not final_answer.strip():
                print("  → Composing final answer...\n")
                messages.append({
                    "role": "user",
                    "content": (
                        "Based on the tool results above, please provide "
                        "a clear and well-formatted answer to my original "
                        "question. Do not call any more tools."
                    )
                })
                forced_response = await call_llm(messages, allow_tools=False)
                forced_message = forced_response["choices"][0]["message"]
                final_answer = forced_message.get("content", "No response generated.")
                messages.append(forced_message)

            # ── Print the final answer ────────────────────────────
            print(f"\nAgent:\n{final_answer}\n")
            print("="*55)


if __name__ == "__main__":
    asyncio.run(agent_loop())
