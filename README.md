# Weather MCP

Weather MCP is a small MCP (Model-Callable Program) server that provides weather-related tools backed by the U.S. National Weather Service (NWS) API. It exposes two async tools implemented in Python that can be invoked by an agent or by a simple HTTP wrapper.

Files of interest
- Manifest: [manifest.json](C:/Users/Rathishan/Desktop/Git/Mcp_Server/weather/manifest.json)
- Entry point (tools implementation): [src/weather/__init__.py](C:/Users/Rathishan/Desktop/Git/Mcp_Server/weather/src/weather/__init__.py)

Overview
- Name: Weather MCP (display_name: Weather MCP)
- Version: 0.1.0
- Tools exposed:
  - get_alerts(state: str) -> str — Get active weather alerts for a two-letter US state code (e.g. "CA", "NY").
  - get_forecast(latitude: float, longitude: float) -> str — Get a short forecast for a lat/lon (returns next 5 periods).

Quick details from the manifest
- Entry point: src/weather/__init__.py
- Run command (from manifest mcp_config):

  uv run --directory <weather-folder> python <weather-folder>/src/weather/__init__.py

  (The manifest uses a uv wrapper; an equivalent direct run is below.)

Prerequisites
- Python 3.10+ (use the version your project uses)
- Install runtime deps used by the module: mcp (MCP framework) and httpx2

Example (use a virtualenv):

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install --upgrade pip
pip install mcp httpx2
```

Running the MCP server locally
- Direct run (simple):

```powershell
python src\weather\__init__.py
```

This runs the MCP server which registers the two tools and runs mcp.run(transport="stdio") (see __init__.py). If you prefer using the manifest command the project provides, use the mcp-configured command in the manifest.

How the tools work (brief)
- get_alerts(state): calls NWS `/alerts/active/area/{state}` endpoint and returns formatted alerts.
- get_forecast(lat, lon): calls NWS `/points/{lat},{lon}` to discover the forecast URL and returns the next 5 forecast periods in a formatted text block.

Integrating Weather MCP with Claude (guide)

This repository exposes the weather tools as Python functions accessible to an MCP server. There are multiple ways to integrate these with a Claude-based system; the two recommended approaches are:

1) HTTP wrapper approach (recommended for simplicity)
2) Direct subprocess/stdio integration (advanced — when the host agent can call MCP-style stdio tools directly)

Approach 1 — HTTP wrapper (recommended)
- Create a small HTTP adapter that imports the existing async functions and exposes them as HTTP endpoints. This is simple and makes the tools available to any code (including Claude, other LLM agents, or webhooks).

Example FastAPI wrapper (save as tools_wrapper.py):

```python
# tools_wrapper.py
from fastapi import FastAPI
import asyncio
from pydantic import BaseModel

# Import the functions from the MCP module
from src.weather.__init__ import get_alerts, get_forecast

app = FastAPI()

class AlertsRequest(BaseModel):
    state: str

class ForecastRequest(BaseModel):
    latitude: float
    longitude: float

@app.post("/alerts")
async def alerts(req: AlertsRequest):
    result = await get_alerts(req.state)
    return {"result": result}

@app.post("/forecast")
async def forecast(req: ForecastRequest):
    result = await get_forecast(req.latitude, req.longitude)
    return {"result": result}

# Run with: uvicorn tools_wrapper:app --port 8000
```

Run the wrapper:

```powershell
pip install fastapi uvicorn
uvicorn tools_wrapper:app --port 8000
```

Call from Claude or any HTTP-capable agent
- From Claude's code or a backend that the Claude assistant can call, POST JSON to the wrapper endpoints.

Curl examples:

```bash
curl -X POST "http://localhost:8000/alerts" -H "Content-Type: application/json" -d '{"state":"CA"}'

curl -X POST "http://localhost:8000/forecast" -H "Content-Type: application/json" -d '{"latitude":37.7749,"longitude":-122.4194}'
```

Python example (requests):

```python
import requests

r = requests.post('http://localhost:8000/forecast', json={"latitude":37.7749, "longitude":-122.4194})
print(r.json()['result'])
```

How to wire this into Claude code (conceptual)
- If using a Claude-hosted environment that supports external webhooks/tools, register the above HTTP endpoints as tools or webhooks in the Claude tool configuration.
- If you host the wrapper on a reachable URL (or via a tunneling service during development), provide the endpoint to Claude as a tool. When Claude needs weather info, it can call the endpoint and receive the formatted string in the response.

Notes for production
- Add authentication to the HTTP wrapper (API key, bearer token) before exposing the endpoints publicly.
- Add rate limiting and caching for repeated requests to the same coordinates/state to reduce calls to the NWS API.
- Validate inputs carefully (lat/lon ranges, 2-letter state codes).
- Respect NWS API usage policies and set a proper User-Agent header (the MCP module already sets USER_AGENT).

Approach 2 — Subprocess / stdio (advanced)
- MCP servers are often designed to run as a tool process that communicates over stdio (JSON-based protocol). If the Claude integration supports launching external tools with a stdio protocol, you can run the provided entry script and let Claude send tool invocation requests directly to it.
- The manifest’s mcp_config shows the command to run the MCP process. Use that command in your tool-launch configuration for Claude (if Claude's tool runner supports launching and communicating with stdio tools).

High-level steps:
1. Ensure the environment has the same Python packages installed as the MCP server.
2. Launch the MCP process using the manifest command (or directly run python src/weather/__init__.py).
3. Use the hosting environment/tooling to connect the agent (Claude) to the process stdio using the MCP protocol. The exact wiring depends on your Claude deployment and how it accepts external tools.

If you need help wiring up stdio-based integration, provide details about how Claude expects tools to be registered (HTTP only, JSON-RPC, or stdio) and an example of a tool config from your Claude environment; concrete instructions can be added to this README.

Troubleshooting
- "Unable to fetch..." responses come from the underlying HTTP calls to the NWS API. Check network access and ensure the NWS API is reachable.
- If you see import errors when importing src.weather from the wrapper, ensure the Python module path includes the project root or run the wrapper from the repository root so imports resolve (python executed from C:/Users/Rathishan/Desktop/Git/Mcp_Server).

Contact / Author
- Author listed in manifest: Rathishan

License
- No license specified in the project. Add license info if this repo will be published or shared.

---

If preferred, a ready-to-run wrapper file or a small example that registers the wrapper with a specific Claude tool config can be added — tell which integration mode Claude (hosted or self-hosted) is using and an example of the tool registration format, and a concrete example will be added.
