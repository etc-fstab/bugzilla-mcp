# Bugzilla MCP guide for Windows PC

# What we built

We built a small Python program that lets an AI agent read information from our on-premises Bugzilla.

The program runs on the  **Windows** laptop. It connects to Bugzilla through its REST API, using a Bugzilla API key.

The agent can request bug searches, read descriptions and comments, examine status history, and use that evidence to answer questions or prepare reports.

The program does not connect directly to MySQL. No Python installation or software change is required on the Bugzilla VM.

# What MCP means

MCP stands for  **Model Context Protocol** . It defines how an AI application discovers available tools, calls them with parameters, and receives results.

**An MCP tool is similar to a function.**  For example:

`monthly_report_data(year=2026, month=10)`

**The MCP server is the program that exposes these functions. In our case, it is a Python program running locally.**

The “Server” describes its role. It does not mean we need a separate server machine.

# What the agent does

The agent is the AI working inside Codex, with instructions and access to tools.

For example, when I ask:

“Show bugs assigned to me that were resolved in October 2026.”

The agent interprets the request, chooses the monthly reporting tool, and supplies the year and month.

The Python program collects the Bugzilla evidence and returns structured data. The agent then reads that data and writes the answer.

The responsibilities are:

Agent: understand the request, choose tools, interpret evidence, and write the response.

MCP server: expose the available functions and handle tool calls.

Bugzilla client: communicate with Bugzilla and collect the requested data.

Bugzilla: store the bugs, comments, assignments, and change history.

Our Python program contains no AI model and needs no separate OpenAI API key. The AI is provided by Codex.

# How the components communicate

The flow is:

My request → Codex agent → local MCP server → Bugzilla REST API

The response travels back through the same components.

Codex communicates with the local Python program through STDIO: standard input and standard output.

The Python program communicates with Bugzilla over HTTPS.

There is no listening MCP network port in this setup. Codex starts the Python process when connecting to the configured server.

# The two core Python files

**server.py**

This is the MCP entry point. It defines the tool names, their parameters, descriptions, and returned results.

It exposes three tools:

search_owned_bugs  
 get_bug_details  
 monthly_report_data

It also provides command-line checks and a suggested monthly-report prompt.

**bugzilla_client.py**

This handles the actual Bugzilla work:

- Reading configuration and the API key.
- Sending API requests.
- Following multiple pages of search results.
- Reading bug details, comments, and history.
- Finding resolution events within a month.
- Separating currently reopened bugs.
- Reporting missing or incomplete evidence.

These two files contain the core application logic.

# The supporting files

config.json contains non-secret settings: Bugzilla URL, default owner, reporting timezone, certificate verification, and collection limits.

requirements.txt lists the direct Python dependencies.

requirements.lock.txt records the pinned dependency versions used to reproduce the environment.

setup.ps1 creates the local Python environment and installs dependencies.

start.ps1 starts the program using that environment.

codex-mcp.toml contains an example of the Codex registration settings.

test_bugzilla.py contains automated tests using synthetic data.

smoke_stdio.py checks that a real server subprocess starts and exposes its MCP tools.

.venv contains the locally installed Python environment and packages.

# What .venv provides

A virtual environment keeps this program’s Python dependencies separate from other Python installations.

It is an ordinary directory on the laptop. It is not a container or a security boundary.

Its contents remain after reboot. However, the environment depends on the base Python installation used to create it.

Our environment was created using Codex’s bundled Python. If that runtime disappears after an application update, we may need to rebuild the environment.

A separately installed local Python can also be used.

# Authentication

Browser authentication is separate mechanisms.

Our tested Bugzilla API accepts an API key, so the program does not need to automate the browser login.

The laptop still needs network access to Bugzilla, normally you're in the office or using VPN.

The key is stored in the Windows User environment variable:

`BUGZILLA_API_KEY`

The program first checks its process environment, then the saved Windows User environment.

The saved User variable survives reboot. The program reads it again on each tool call, unless a process-level value overrides it.

The key should not be stored in Python files, config.json, Git, documentation, or chat.

Certificate verification is currently disabled, as requested during setup. That setting is separate from API authentication.

# How to set up another laptop

The following are instructions for future use, not commands executed as part of writing this guide.

**Step 1: Obtain the project files.**

Copy or check out the source into a local folder.

**Step 2: Check the prerequisites.**

Use PowerShell 7 or newer and Python 3.12 or newer.

**Step 3: Create the environment.**

From the project directory, run:

```powershell
.\setup.ps1
```

The script creates .venv and installs the pinned dependencies. It does not install packages into shared Linux Python.

**Step 4: Review config.json.**

Confirm the Bugzilla URL, default owner, and reporting timezone.

**Step 5: Set the API key.**

Open Windows “Edit environment variables for your account.”

Create BUGZILLA_API_KEY under User variables and enter the key locally.

**Step 6: Check API access.**

Connect to the VPN and run:

```powershell
.\start.ps1 --check
```

This checks connectivity and prints limited metadata without displaying the key.

**Step 7: Register the server in Codex.**

On this laptop, registration is in:

`C:\Users\your-name\.codex\config.toml`

The registration names the server bugzilla_monthly and specifies the full paths to the project’s Python executable and server.py.

codex-mcp.toml provides the example settings. Paths must be adjusted for another laptop.

**Step 8: Reconnect the MCP server.**

After registration, reconnect the server or fully restart Codex when necessary. Then verify that its tools are available.

# What happens after reboot

The source files, virtual environment, saved API key, and Codex registration remain on disk.

When Codex reconnects, it launches the local MCP server again.

We do not need a Windows service, scheduled task, or permanently open terminal.

# Read-only behavior

The client sends GET requests through an allowlist of Bugzilla read endpoints.

It exposes no operation to create bugs, edit fields, add comments, or change status.

The API key’s permissions still determine which bugs can be read. The program’s read-only design does not change the account’s permissions within Bugzilla.

# Does Git make sense?

Yes. A private internal Git repository is appropriate for this program.

It would preserve change history, support review, and make installation on another laptop easier.

The repository should normally contain:

- server.py and bugzilla_client.py.
- The test files.
- setup.ps1 and start.ps1.
- Both requirements files.
- Reviewed, non-secret configuration and registration examples.
- A README and .gitignore.

Keep API keys, private keys, environment exports, .venv, Python caches, and downloaded bug data out of Git.

Each laptop should create its own virtual environment and supply its own key. Installed environments should be rebuilt from the dependency files rather than copied between machines.

# How to maintain it

Change server.py when adding a tool or changing tool parameters.

Change bugzilla_client.py when changing API queries, collection rules, pagination, or date handling.

Change config.json when changing non-secret defaults.

After Python changes, run the unit tests and STDIO smoke check, then reconnect the MCP server so Codex starts the updated program.

The existing verification commands are:

```powershell
.\.venv\Scripts\python.exe -m unittest -v test_bugzilla
```

```powershell
.\.venv\Scripts\python.exe smoke_stdio.py
```

The first tests application behavior without network access. The second checks the actual local MCP connection.
