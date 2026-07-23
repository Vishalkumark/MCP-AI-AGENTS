Great question. I just read the official Microsoft documentation. Let me answer all 4 questions clearly.

1. Yes — Microsoft has their own official MCP Server
The Microsoft MCP Server for Enterprise https://mcp.svc.cloud.microsoft/enterprise is a programmatic interface for AI agents to query enterprise data in your Microsoft Entra tenant using natural language. Built on the Model Context Protocol, it translates natural language requests into Microsoft Graph API calls.
But — and this is critical for you to understand:
Microsoft MCP Server for Enterprise is currently in public preview. It focuses on Microsoft Entra identity and directory read-only scenarios such as user, group, application, and device insights, plus administrative reporting.
This means it currently does NOT cover:
	•	Reading your Outlook emails
	•	Calendar events
	•	Attachments
	•	Drafting emails
	•	MOM generation
	•	Task extraction
Your project does all of that. The Microsoft MCP Server is for IT admins querying directory data — not for email productivity. So your project is not replaceable by it.

2. How it actually works
The Microsoft MCP Server for Enterprise exposes three tools:
	•	microsoft_graph_suggest_queries — uses RAG to search a curated catalog of Graph API examples matching the user’s intent
	•	microsoft_graph_get — runs read-only Graph API calls while honouring user roles and granted scopes
	•	microsoft_graph_list_properties — retrieves schema for Graph entities so the AI understands available properties
Simplified flow:

You ask: "How many users in our tenant?"
        │
        ▼
LLM calls microsoft_graph_suggest_queries
        │ returns → GET /users/$count
        ▼
LLM calls microsoft_graph_get
        │ calls Microsoft Graph API
        ▼
Returns: "There are 10,930 users"


3. Permissions Required
All operations respect existing Microsoft Graph permissions, user privileges, and tenant security policies. You must have the appropriate licenses for the data you access.
For the Microsoft Enterprise MCP Server specifically — your existing User.Read permission is enough for basic queries. For advanced admin scenarios your admin needs to grant directory read permissions.
For your project the permissions you already have are correct and sufficient.

4. How it works with LibreChat
This is where it gets interesting for you. Since it’s a standard MCP server with an HTTPS endpoint, you can connect it to LibreChat exactly the same way as your outlook server. Add it to librechat.yaml:

mcpServers:
  outlook-ai:
    type: streamable-http
    url: https://figpt-mcp-outlook.fichtnergroup.com/mcp  # your server

  microsoft-enterprise:
    type: streamable-http
    url: https://mcp.svc.cloud.microsoft/enterprise       # Microsoft's server


LibreChat will then have access to both — your email/calendar/attachment tools AND Microsoft’s directory tools simultaneously.

What this means for your Teams decision
Microsoft MCP Server for Enterprise is currently in preview and is available only in the public cloud (global service).
Since it’s preview-only and read-only, and your organisation needs production-grade Teams integration including reading channels, messages, and drafting replies, the correct approach remains:

Your outlook-ai-agent-mcp     → Email, Calendar, Attachments, MOM, Tasks (DONE)
teams-ai-agent-mcp (new)      → Teams channels, messages, meetings
Microsoft Enterprise MCP      → Directory queries, user lookup, admin reporting (optional, when GA)


Three servers, three jobs, all connected to LibreChat simultaneously. Clean, maintainable, production-ready.

One more thing — rate limits
Requests to the Microsoft Enterprise MCP Server are limited to 100 calls per minute per user. Calls made through microsoft_graph_get are also subject to standard Microsoft Graph throttling limits.
This is another reason your custom server is valuable — you control the rate limiting, retry logic, and error handling yourself rather than being subject to Microsoft’s preview limits.

Bottom line:
	•	Microsoft’s MCP Server = IT admin directory queries only, preview, read-only
	•	Your project = email productivity, full read/write, production-ready
	•	They complement each other, not compete
	•	For Teams — build a separate standalone server as discussed
Want me to now give you the Teams standalone server setup plan?​​​​​​​​​​​​​​​​