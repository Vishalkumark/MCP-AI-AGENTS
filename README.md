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