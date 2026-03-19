---
name: ms-agent-framework
description: "Generate decoupled AI agents using Microsoft Agent Framework (agent_framework + AzureOpenAIResponsesClient). USE WHEN: creating new AI agents, adding agents to Flask apps, scaffolding agent modules, setting up Azure OpenAI client infrastructure. Ensures agents are modular, authenticate via AzureCliCredential, read endpoints from .env, and follow the agents/ package pattern."
argument-hint: "Describe the agent's purpose and the app it belongs to"
---

# Microsoft Agent Framework — Agent Generator

## When to Use

- User asks to create a new AI agent for a Flask (or other) application
- User asks to scaffold an `agents/` package or add a new agent module
- User asks to set up Azure OpenAI client infrastructure
- User asks to integrate Microsoft Agent Framework into a project

## Architecture Pattern

Every project using this skill MUST follow this structure:

```
project_root/
├── .env                    # Azure endpoint + deployment config
├── app.py                  # Flask app — imports agents, calls them
├── requirements.txt
├── agents/
│   ├── __init__.py         # Agent Registry — single import point
│   ├── client.py           # Shared infrastructure (client, credentials, helpers)
│   ├── <agent_name>.py     # One file per agent (INSTRUCTIONS + create_agent)
│   └── ...
└── templates/
    └── index.html
```

## Key Principles

1. **Agents are decoupled** — each agent lives in its own file under `agents/`. Never put agent instructions or creation logic in `app.py`.
2. **Single client** — `agents/client.py` is the ONLY place where `AzureOpenAIResponsesClient` is instantiated.
3. **Endpoint in `.env`** — The Azure OpenAI endpoint and deployment name MUST be read from environment variables, never hardcoded.
4. **`__init__.py` is the registry** — `app.py` imports everything it needs from `agents` (the package), never from individual agent files.
5. **Agents are invoked via `agent.run(prompt)`** — NOT `client.run()`. The client creates agents; agents run prompts.

## Step-by-Step Procedure

### Step 1 — Ensure `.env` exists

If the project doesn't have a `.env` file, create one:

```env
AZURE_OPENAI_ENDPOINT=https://<resource-name>.openai.azure.com/openai/v1/
AZURE_OPENAI_DEPLOYMENT=gpt-4o
```

### Step 2 — Create or verify `agents/client.py`

This file provides ALL shared infrastructure. Follow this template exactly:

```python
"""
Shared infrastructure for all agents.

Provides the Azure OpenAI client, credentials,
the background async event loop, and common utility functions.
"""

import asyncio
import json
import os
import re
import threading

from azure.identity import AzureCliCredential
from agent_framework.azure import AzureOpenAIResponsesClient

# ---------------------------------------------------------------------------
# Configuration — always from environment variables
# ---------------------------------------------------------------------------
AZURE_OPENAI_ENDPOINT = os.environ.get("AZURE_OPENAI_ENDPOINT", "")
DEPLOYMENT_NAME = os.environ.get("AZURE_OPENAI_DEPLOYMENT", "gpt-5.1")

# ---------------------------------------------------------------------------
# Credentials & Client
# ---------------------------------------------------------------------------
credential = AzureCliCredential()

client = AzureOpenAIResponsesClient(
    endpoint=AZURE_OPENAI_ENDPOINT,
    deployment_name=DEPLOYMENT_NAME,
    credential=credential,
)

# ---------------------------------------------------------------------------
# Shared async event loop (runs in a background daemon thread)
# ---------------------------------------------------------------------------
_loop = asyncio.new_event_loop()


def _start_loop(loop: asyncio.AbstractEventLoop) -> None:
    asyncio.set_event_loop(loop)
    loop.run_forever()


threading.Thread(target=_start_loop, args=(_loop,), daemon=True).start()


# ---------------------------------------------------------------------------
# JSON parsing helper
# ---------------------------------------------------------------------------
def parse_agent_json(raw: str) -> dict:
    """Extract and parse JSON from agent response text."""
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        pass
    match = re.search(r"```(?:json)?\s*([\s\S]*?)```", raw)
    if match:
        try:
            return json.loads(match.group(1).strip())
        except json.JSONDecodeError:
            pass
    match = re.search(r"\{[\s\S]*\}", raw)
    if match:
        try:
            return json.loads(match.group(0))
        except json.JSONDecodeError:
            pass
    return {"error": "Could not parse agent response", "raw": raw}
```

**Critical rules for `client.py`:**
- `AZURE_OPENAI_ENDPOINT` MUST come from `os.environ.get("AZURE_OPENAI_ENDPOINT")` — never hardcode a URL
- Authentication is ALWAYS `AzureCliCredential()` — no API keys, no connection strings
- The background event loop pattern is required for Flask apps (Flask is sync, agents are async)

### Step 3 — Create the agent module

Each agent file follows this exact pattern:

```python
"""
<Agent Name> — <One-line description>.

<More detail about what this agent does.>
"""

INSTRUCTIONS = """You are a <role description>.
Your job is to <task description>.

<Detailed instructions for the agent's behavior>

Return ONLY valid JSON (no markdown, no extra text) with this exact structure:
{
    <JSON schema the agent must follow>
}
"""


def create_agent(client, tools=None):
    """Create and return the <Agent Name>."""
    return client.create_agent(
        name="<AgentName>",
        instructions=INSTRUCTIONS,
        tools=tools or [],
    )
```

**Critical rules for agent modules:**
- `INSTRUCTIONS` is a module-level constant string — the full system prompt
- `create_agent(client, tools=None)` is the ONLY public function — it receives the client, not imports it
- Agent names use PascalCase in `create_agent`, filenames use snake_case
- If the agent should return structured data, include the exact JSON schema in instructions
- Never import `client` directly in agent files — it's passed as a parameter

### Step 4 — Register in `agents/__init__.py`

```python
"""
Agent Registry — single import point for app.py.

Creates all agents at import time and exposes them, along with
shared infrastructure (clients, event loop, helpers).
"""

from .client import (
    AZURE_OPENAI_ENDPOINT,
    DEPLOYMENT_NAME,
    credential,
    client,
    _loop,
    parse_agent_json,
)

# Import agent modules
from . import <agent_module_1>
from . import <agent_module_2>

# Create agent instances
<agent_1> = <agent_module_1>.create_agent(client)
<agent_2> = <agent_module_2>.create_agent(client)
```

**Critical rules for `__init__.py`:**
- Re-export everything `app.py` needs (config, helpers, agent instances)
- Agents are created HERE at import time, not in `app.py`
- Add any project-specific directory constants from `client.py`

### Step 5 — Wire into `app.py`

In `app.py`, agents are used like this:

```python
from agents import (
    _loop,
    parse_agent_json,
    <agent_1>,
    <agent_2>,
)

def _run_agent(agent, prompt: str) -> str:
    """Run an agent synchronously via the background event loop."""
    async def _execute():
        response = await agent.run(prompt)
        return str(response)

    future = asyncio.run_coroutine_threadsafe(_execute(), _loop)
    return future.result(timeout=120)

# Then in route handlers:
raw = _run_agent(<agent_1>, "Analyze this: ...")
result = parse_agent_json(raw)
```

**Critical rules for `app.py`:**
- NEVER instantiate clients or agents in `app.py` — import them from `agents`
- Use `_run_agent(agent, prompt)` helper — it bridges sync Flask to async agents
- Use `parse_agent_json()` to safely extract JSON from agent responses
- The `_run_agent` helper calls `agent.run(prompt)`, NOT `client.run()`

### Step 6 — Update `requirements.txt`

Ensure these packages are present:

```
flask
azure-identity
agent-framework[azure]
python-dotenv
```

## Checklist

Before finishing, verify:
- [ ] `.env` has `AZURE_OPENAI_ENDPOINT` and `AZURE_OPENAI_DEPLOYMENT`
- [ ] `agents/client.py` reads endpoint from env vars (no hardcoded URLs)
- [ ] `agents/client.py` uses `AzureCliCredential()` for auth
- [ ] Each agent file has `INSTRUCTIONS` constant + `create_agent(client, tools)` function
- [ ] Agent files do NOT import `client` directly
- [ ] `agents/__init__.py` creates all agent instances and re-exports them
- [ ] `app.py` imports agents from `agents` package only
- [ ] `app.py` uses `agent.run(prompt)` pattern, not `client.run()`
- [ ] No Azure endpoints or credentials are committed to source control

## Common Mistakes to Avoid

| Mistake | Fix |
|---------|-----|
| `client.run(agent=agent, input=prompt)` | Use `agent.run(prompt)` — the agent object has the `.run()` method |
| Hardcoded endpoint in `client.py` | Use `os.environ.get("AZURE_OPENAI_ENDPOINT")` |
| Agent imports client directly | Pass `client` as parameter to `create_agent()` |
| Agent logic in `app.py` | Move to dedicated file in `agents/` |
| Nested f-strings with quotes in SSE yields | Build dict first, then `json.dumps()` outside the f-string |
