# Enterprise Agentic AI Platform — Phase 1 Prerequisites

**Organisation:** Fichtner GmbH & Co. KG
**Scope:** Everything required before implementation begins
**Status:** Pending Your Review

-----

## 1. AI Model & LLM Gateway

|Item              |Value                                                                         |Status                    |
|------------------|------------------------------------------------------------------------------|--------------------------|
|Gateway           |Requesty.AI                                                                   |Have access               |
|Primary model     |`azure/gpt-4.1-nano@swedencentral`                                            |Confirmed                 |
|Fallback model    |`mistral/mistral-small-2603`                                                  |Confirmed                 |
|API Key           |Requesty.AI API key                                                           |Need to retrieve          |
|Base URL          |`https://router.requesty.ai/v1`                                               |Standard, no action needed|
|Future flexibility|Code designed to swap to Azure OpenAI / OpenAI / Gemini via `.env` change only|Already planned           |

### How to get the Requesty.AI API Key

1. Go to **https://app.requesty.ai/**
1. Log in with your organisation’s Requesty.AI account (ask your lead if you don’t have one)
1. Navigate to **API Keys** in the left sidebar
1. Click **Create New Key** and name it `outlook-ai-phase1`
1. Copy the key immediately (shown only once) and paste into your `.env` file as `REQUESTY_API_KEY`
1. Confirm your organisation has routing enabled for `azure/gpt-4.1-nano` and `mistral/mistral-small-2603`. If not listed, ask your lead to enable those two models on the Requesty.AI dashboard under Model Access

-----

## 2. Python Libraries

|Library        |Purpose                                                              |Install Command            |
|---------------|---------------------------------------------------------------------|---------------------------|
|`fastmcp`      |MCP server framework                                                 |`pip install fastmcp`      |
|`msal`         |Microsoft authentication (token handling)                            |`pip install msal`         |
|`msgraph-sdk`  |Official Microsoft Graph API Python SDK                              |`pip install msgraph-sdk`  |
|`python-dotenv`|Loads `.env` config into Python                                      |`pip install python-dotenv`|
|`httpx`        |Async HTTP client (used internally by msgraph-sdk and Requesty calls)|`pip install httpx`        |
|`pydantic`     |Data validation for tool inputs/outputs                              |`pip install pydantic`     |

### How to get these

All are free, open-source, and installed via `pip` (Python’s package manager, included automatically with Python). No accounts or licenses needed. Once Python is installed, a single command installs everything:

```bash
pip install fastmcp msal msgraph-sdk python-dotenv httpx pydantic
```

We will also generate a `requirements.txt` file so this becomes one command (`pip install -r requirements.txt`) for any future machine, including FGSV297.

-----

## 3. Licenses

|Item                            |License Type                   |Cost                  |Status                                                    |
|--------------------------------|-------------------------------|----------------------|----------------------------------------------------------|
|FastMCP                         |Apache 2.0 (open source)       |Free                  |No action                                                 |
|msal / msgraph-sdk              |MIT (open source)              |Free                  |No action                                                 |
|Python                          |PSF License (open source)      |Free                  |No action                                                 |
|Microsoft 365 / Graph API access|Org-provided enterprise license|Already covered by org|No action                                                 |
|Requesty.AI                     |Org subscription               |Already covered by org|Confirm with lead that Phase 1 usage is within plan limits|
|LibreChat                       |MIT (open source)              |Free                  |No action, team manages it                                |

No new licenses need to be purchased for Phase 1. Everything is either open-source or already covered under existing organisational subscriptions.

-----

## 4. Endpoints & URLs

|Endpoint                                    |URL                                                              |Purpose                                       |
|--------------------------------------------|-----------------------------------------------------------------|----------------------------------------------|
|Microsoft Graph API                         |`https://graph.microsoft.com/v1.0/`                              |All Outlook/Calendar data calls               |
|Microsoft identity platform (token endpoint)|`https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token`|OAuth token exchange                          |
|Requesty.AI Gateway                         |`https://router.requesty.ai/v1`                                  |LLM inference calls                           |
|Your FastMCP server (local dev)             |`http://localhost:8000/mcp`                                      |Local testing before deployment               |
|Your FastMCP server (future, FGSV297)       |`http://<fgsv297-address>:8000/mcp`                              |Pending IP/domain from team                   |
|LibreChat OAuth callback                    |`https://<librechat-domain>/api/mcp/outlook-ai/oauth/callback`   |Needed in Azure App Registration redirect URIs|

### How to get the missing pieces

- **LibreChat domain** — ask your team for the exact LibreChat URL (the one you log into in your browser). We need this to register the OAuth callback URL.
- **FGSV297 address** — ask your team for the internal IP or hostname once it’s provisioned for your MCP server.

-----

## 5. Access & Credentials Required

|Access Item                                                 |Status                               |How to Get It                                                                                      |
|------------------------------------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------|
|Azure App Registration — Client ID                          |Already have                         |Azure Portal -> App Registrations -> your app -> Overview                                          |
|Azure App Registration — Tenant ID                          |Already have                         |Same screen as above                                                                               |
|Azure App Registration — Client Secret                      |Already have                         |If not visible, go to Certificates & Secrets -> generate new one (visible only once)               |
|Graph API Permissions — Mail.Read, Calendars.Read, User.Read|Already granted                      |Confirmed via your screenshot                                                                      |
|Graph API Permissions — Mail.ReadWrite, Mail.Send           |Configured, not yet consented        |Ask admin to click “Grant admin consent” in Azure Portal (needed for Phase 2, not blocking Phase 1)|
|LibreChat login                                             |Have, via corporate account          |No action                                                                                          |
|LibreChat MCP Settings access                               |You can self-serve add servers via UI|No action                                                                                          |
|Requesty.AI account and API key                             |Need to retrieve key                 |See Section 1 above                                                                                |
|SSH key for FGSV297                                         |Pending from team                    |Team is generating it, not a Phase 1 blocker                                                       |
|FGSV297 server address                                      |Pending from team                    |Ask team once SSH key is ready                                                                     |
|Python on FGSV297                                           |Unconfirmed                          |Ask team to confirm, or request install when SSH access is granted                                 |

-----

## 6. Local Development Environment (Your Windows VM)

|Item        |Status                  |How to Get It                                                                               |
|------------|------------------------|--------------------------------------------------------------------------------------------|
|Python 3.11+|Confirm installed       |Download from `https://www.python.org/downloads/`, check “Add Python to PATH” during install|
|VS Code     |Confirm installed       |Download from `https://code.visualstudio.com/`                                              |
|curl        |Built into Windows 11   |No action needed                                                                            |
|Git         |Not required for Phase 1|Skipped per your direction, Azure DevOps handles versioning later if needed                 |

-----

## 7. Summary — What You Need To Do Before Step 4 (Implementation)

These are the only action items blocking us:

1. Confirm Python 3.11+ is installed on your Windows VM (or install it)
1. Retrieve your Requesty.AI API key
1. Confirm with your lead that `azure/gpt-4.1-nano` and `mistral/mistral-small-2603` are enabled on your Requesty.AI account
1. Get the exact LibreChat domain/URL from your team
1. Locate your Azure App Registration’s Client ID, Tenant ID, and Client Secret (generate secret if not already visible)

Everything else (SSH key, FGSV297 address, admin consent for Mail.Send/ReadWrite) is not a blocker. Phase 1 can be fully built and tested locally on your Windows VM without them.

-----

Review this list, confirm or amend, and let me know once ready. I will wait for your Step 4 instruction before proceeding.