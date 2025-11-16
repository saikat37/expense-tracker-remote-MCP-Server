# Expense Tracker — Remote MCP Server

**Project overview**
- **What:** An Expense Tracker implemented as a FastMCP server. Provides MCP tools to add and list expenses and summarize by category.
- **Why:** Demonstrates building a remote MCP server (HTTP transport) that can be deployed to FastMCP Cloud and consumed remotely (for example by Claude Desktop).
- **Files:** `main.py` (server), `proxy.py` (local proxy example), `categories.json` (optional static data), `requirements.txt`, `pyproject.toml`.

**Quick summary**
- Local run (direct): `uv run python main.py` (runs server directly)
- Development with inspector: `uv run fastmcp dev main.py` (starts MCP Inspector + proxy)
- Deploy: push repo to GitHub and deploy from https://fastmcp.cloud

**Table of contents**
- Project structure
- Local development
- FastMCP dev (Inspector)
- Deploy to FastMCP Cloud
- Connect with Claude Desktop (Pro)
- Connect without Pro using local proxy
- Troubleshooting (common errors & fixes)
- Notes & production recommendations

**Project structure**
- `main.py` — MCP server implementation (defines `mcp = FastMCP("ExpenseTracker")` and tools/resources)
- `proxy.py` — Example proxy client that connects to a remote FastMCP server (useful for local clients or non-Pro connector usage)
- `categories.json` — Optional file containing default categories
- `requirements.txt`, `pyproject.toml` — dependency manifests

**Local development**
1. Create and activate a Python virtualenv (Windows PowerShell):
	 - `python -m venv .venv`
	 - `.\\.venv\\Scripts\\Activate.ps1`  (PowerShell) or `.\\.venv\\Scripts\\activate.bat` (cmd)
2. Install dependencies (using `uv` or pip):
	 - `uv pip sync requirements.txt`  (recommended if using `uv` environment)
	 - or `pip install -r requirements.txt`

3. Run directly (good for debugging):
	 - `uv run python main.py`
	 - This runs the server directly (the `if __name__ == "__main__"` block executes). Useful when you want a direct process and to see Uvicorn output.

**FastMCP Inspector (recommended for dev)**
- Start the Inspector and proxy that helps debug and inspect MCP traffic:
	- `uv run fastmcp dev main.py`
- Inspector runs a browser UI (typically opens automatically) and a local proxy. The Inspector expects your server to be reachable at the configured HTTP endpoint. `fastmcp dev` will import your module and start your `mcp` instance.

Notes about `__main__` and `fastmcp dev` (important):
- Do NOT put the MCP server startup (i.e. `mcp.run(...)`) in a module-level `if __name__ == "__main__":` block if you expect the FastMCP dev runner or FastMCP Cloud to start your server. `fastmcp dev` and the cloud IMPORT your module, they do not execute it as a script.
- Instead: define the `mcp` instance at module level (for example `mcp = FastMCP("ExpenseTracker")`) and let the runner start it. If you want to keep a `if __name__ == "__main__"` block, only use it for manual direct runs (i.e. `python main.py`) — but it is not required for dev or cloud deployments.
- What we changed in this project: removed writing the DB into the code directory (we now use the OS temp directory or `DB_PATH` env var) and ensured `mcp` is defined at module level so FastMCP can detect it. Keep `if __name__ == "__main__"` ONLY for direct local runs if you want to.

**Deploying to FastMCP Cloud**
1. Push your repository to GitHub (or Git provider):
	 - `git init` (if new), `git add .`, `git commit -m "initial commit"`
	 - `git remote add origin <repo-url>`
	 - `git push -u origin main`
2. Go to https://fastmcp.cloud and connect your GitHub account.
3. Choose your repository and deploy.
	 - Set the entry point to `main.py`.
	 - The platform will install `requirements.txt` (or `pyproject.toml`) and run a pre-flight check.
4. After a successful deployment you get a server URL like `https://<your-app>.fastmcp.app/mcp`.

**Connecting with Claude Desktop (Pro)**
- Claude Desktop Pro supports adding a custom connector by URL.
- In Claude Desktop: Settings → Connectors → Add Custom Connector
	- Name: `ExpenseTracker` (or whatever you want)
	- URL: `https://<your-app>.fastmcp.app/mcp`
- With the Pro plan, you can add that URL directly and the connector will use Streamable HTTP.

**Connecting without Pro (workaround using local proxy)**
If you don't have Claude Pro or another client that supports streamable HTTP out-of-the-box, you can run a local proxy that bridges between a local STDIO or HTTP client and the remote FastMCP server.

Proxy: copy-and-run example (for non‑Pro clients)

If you don't have Claude Desktop Pro you can run a local proxy that forwards requests to your FastMCP Cloud server. Copy the `proxy.py` file below (or use the one in this repo), then run it locally.

`proxy.py` (copy into a file named `proxy.py`):
```python
from fastmcp import FastMCP

# Replace the URL below with your FastMCP Cloud server URL if different
mcp = FastMCP.as_proxy(
	"https://expanse-tracker.fastmcp.app/mcp",
	name="Saikat server proxy",
)

if __name__ == "__main__":
	# Starts a local proxy that bridges local clients to the remote server
	# The inspector/proxy normally listens on a localhost port (e.g. 6277)
	mcp.run()
```

Run the proxy locally:
- Activate your venv, then run:
```powershell
uv run python proxy.py
# OR
python proxy.py
```

Optional: register/install the proxy for Claude Desktop automatically
- Some `fastmcp` toolchains provide a helper to register or install a local connector for Claude Desktop. If your `uv`/`fastmcp` tool supports it, you can run:
```powershell
uv run fastmcp install claude-desktop proxy.py
```
This attempts to install/register the local `proxy.py` as a Claude Desktop connector so the client can connect to `http://localhost:<proxy-port>/mcp` without manual configuration. If the command isn't available in your `fastmcp` version, just run `python proxy.py` or `uv run python proxy.py` and point Claude Desktop at the proxy URL shown in the proxy output.

After the proxy is running, point your local client (e.g. Claude Desktop) to the proxy URL. Example:
- `http://localhost:6277/mcp` (the Inspector often uses port `6277` — check the proxy output for the exact URL).

This setup lets non‑Pro clients use your remote FastMCP server via a local bridge. The proxy simply forwards Streamable‑HTTP requests from your local machine to `https://expanse-tracker.fastmcp.app/mcp`.
```python
from fastmcp import FastMCP

# Create a proxy to your remote FastMCP Cloud server
mcp = FastMCP.as_proxy(
		"https://expanse-tracker.fastmcp.app/mcp",
		name="Saikat server proxy",
)

if __name__ == "__main__":
		# Runs a local proxy server that clients (e.g., Claude Desktop running locally)
		# can connect to. For example, the MCP Inspector uses a local proxy.
		mcp.run()
```
How to use the proxy:
- Start the proxy locally:
	- `uv run python proxy.py`  (or `python proxy.py`)
- The proxy will expose a local transport (often STDIO or HTTP depending on how you run it).
- In Claude Desktop, point the custom connector URL to the proxy (for example `http://localhost:6277/mcp` if your proxy listens there). The Inspector typically uses port `6277`.

Security note: Running an open proxy may expose access to your remote server. Use local-only bindings and tokens if available.

**Troubleshooting (common errors & fixes)**
- `ModuleNotFoundError: No module named 'aiosqlite'`
	- Cause: missing dependency in environment. Fix:
		- `uv pip sync requirements.txt` OR `pip install aiosqlite`
		- Ensure `requirements.txt` or `pyproject.toml` includes `aiosqlite`.

- `ECONNREFUSED` when Inspector tries to reach `http://localhost:8000/mcp`
	- Cause: the actual server isn't listening on port `8000` or startup failed.
	- Fixes:
		- Run server directly to confirm: `uv run python main.py` and verify Uvicorn started on `http://0.0.0.0:8000`.
		- Check for `if __name__ == "__main__"` block: `fastmcp dev` IMPORTS your module. If you previously only started the server inside that block, the dev runner may not find it. Prefer defining `mcp = FastMCP(...)` at module level so the runner can detect it.
		- Check for port conflicts: `netstat -ano | findstr ":8000"` then `taskkill /F /PID <PID>` to free the port.

- `Read-only filesystem` or failure creating `expenses.db` in cloud
	- Cause: Cloud environment often has read-only application directories.
	- Fix: Use a writable path (temp directory) or external DB. Example in `main.py`:
		```py
		import tempfile, os
		TEMP_DIR = tempfile.gettempdir()
		DB_PATH = os.environ.get("DB_PATH", os.path.join(TEMP_DIR, "expenses.db"))
		```
	- For production, use a managed DB (Postgres, hosted SQLite replacement, etc.) to persist data across deployments.

- `Proxy server port in use` when Inspector starts
	- Cause: previous proxy instance still running. Fix: identify and kill process or choose another port.
		- `netstat -ano | findstr "6277"` then `taskkill /F /PID <PID>` on Windows.

- `CancelledError` or dev server shows `Dev server failed` intermittently
	- Cause: fastmcp dev may spawn an inspector + server and the order/timings can sometimes cause transient failures. Retry: stop and re-run `uv run fastmcp dev main.py`.

**CLI reference (Windows PowerShell)**
- Create venv: ``python -m venv .venv``
- Activate: ``.\\.venv\\Scripts\\Activate.ps1``
- Install: ``uv pip sync requirements.txt``
- Run locally (direct): ``uv run python main.py``
- Run dev inspector: ``uv run fastmcp dev main.py``
- Deploy: push to GitHub; in fastmcp.cloud select repo & deploy

**Testing your server (simple HTTP)**
- After starting server (local):
	- Check `/mcp` health via curl (example): ``curl -X GET http://localhost:8000/mcp``
	- Use the Inspector UI for interactive testing and tool invocation.

**Persistence & production notes**
- Using the temp directory is fine for development or demo, but cloud deployments are ephemeral. For production, use an external persistent database (Postgres, managed SQLite store) or object storage.
- Add configuration via environment variables (e.g., `DB_PATH`, `PORT`) and avoid writing into your code directory.

**Example quick start (copy-paste)**
- PowerShell:
	- `python -m venv .venv`
	- `.\\.venv\\Scripts\\Activate.ps1`
	- `uv pip sync requirements.txt`
	- `uv run fastmcp dev main.py`  # inspector + proxy

**Acknowledgements & Notes**
- This README collects the steps from the recorded agenda: creating a remote MCP server, replacing the simple demo with an Expense Tracker, and deploying to FastMCP Cloud.
- If you want, I can also add a small `deploy.sh` script, a `fastmcp.yaml` sample, or CI workflow to automate deployment from commits.

--
Generated README for the Expense Tracker Remote MCP Server project.
