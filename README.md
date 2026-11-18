# Warehouse Ops MCP Toolkit

An MCP server that lets an agent answer stock and dock questions and raise and close
exceptions for Kestrel warehouse staff at three sites (LEEDS-01, READING-02, GLASGOW-03).
It has 10 tools: 4 read-only and 6 state-changing. The warehouse systems are stubbed over
the supplied `data/stock_snapshot.csv` and `data/dock_schedule.jsonl`.

Verified with Python 3.13.14 and `mcp` 2.3.0 in the repository virtual environment on
Windows. The project began with VS Code and GitHub Copilot demonstrations, then moved to
Claude Code for the final recorded run.

## Where to look

| File | What it is |
| --- | --- |
| `spec.md` | The tool interface design: names, parameters, descriptions, error shape |
| `plan.md`, `tasks.md` | How it was built, and the task breakdown |
| `CHANGES.md` | Every change from Ops' request and from the first spec, each with its evidence |
| `REFLECTION.md` | What the data forced, what went wrong, where the design is weak, effort statement |
| `AGENTS.md` (and `CLAUDE.md`, which imports it) | Optional guidance for agents using the tools |
| `evidence/` | Test output, the Claude Code transcript, screenshots, server logs and registration files, plus an archive of an earlier round of demos in Copilot |

## Set up and run the tests

```text
python -m venv .venv
.venv\Scripts\activate            # macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
pytest
```

The suite has 292 tests. Many start the real server as a subprocess and talk to it through the
MCP protocol, so a full run takes one to two minutes.

## Configuration

The server takes its identity from the environment that launches it. The model cannot set it,
and no tool accepts a `site` argument.

| Variable | Required | Meaning |
| --- | --- | --- |
| `KESTREL_ASSIGNED_SITE` | yes | One of `LEEDS-01`, `READING-02`, `GLASGOW-03`. The server refuses to start otherwise. |
| `KESTREL_CALLER_ID` | yes | An identifier for the caller, for example `ops.leeds.01` |
| `KESTREL_AUDIT_LOG` | no | A file path. If set, the server appends one line per tool call (tool, read/preview/commit, outcome, arguments, never the token). |
| `PYTHONPATH` | yes, when launched from outside the repo | The repo's `src` folder |

To run the server by hand: set `PYTHONPATH=src` and the two required variables, then run
`python -m warehouse_mcp.server`. It speaks MCP over stdio and waits for a client.

## Register it in an agent host

Register **one** server entry per session, with one site. Do not register one server per site
in the same host: the agent could then choose a site by choosing a server. To use another
site, change `KESTREL_ASSIGNED_SITE` and restart the host.

Replace the two paths with yours. On macOS or Linux, the command is
`<repo>/.venv/bin/python`.

**Claude Code** (a `.mcp.json` in the folder you start it from):

```json
{
  "mcpServers": {
    "kestrel-warehouse": {
      "command": "<repo>/.venv/Scripts/python.exe",
      "args": ["-m", "warehouse_mcp.server"],
      "env": {
        "PYTHONPATH": "<repo>/src",
        "KESTREL_CALLER_ID": "ops.leeds.01",
        "KESTREL_ASSIGNED_SITE": "LEEDS-01"
      }
    }
  }
}
```

**VS Code with GitHub Copilot** (`.vscode/mcp.json`, used in Agent mode):

```json
{
  "servers": {
    "kestrel-warehouse": {
      "type": "stdio",
      "command": "<repo>/.venv/Scripts/python.exe",
      "args": ["-m", "warehouse_mcp.server"],
      "env": {
        "PYTHONPATH": "<repo>/src",
        "KESTREL_CALLER_ID": "ops.leeds.01",
        "KESTREL_ASSIGNED_SITE": "LEEDS-01"
      }
    }
  }
}
```

The files used for the recorded runs are in `evidence/claude_code/registration/`.

**A note on the workspace.** The toolkit enforces the site on every call it receives, but it
cannot stop an agent that can read `data/stock_snapshot.csv` directly. For a fair test, start
the agent from an empty folder that contains only the registration. In a real deployment the
stock data would not sit in the agent's workspace. `REFLECTION.md` has the evidence.

## How the tools behave

- **Reads** (`get_stock`, `search_stock`, `list_dock_slots`, `list_exceptions`) change nothing.
  `search_stock` returns every plausible match and withholds quantities when more than one
  product matches.
- **Writes** (`book_dock_slot`, `cancel_dock_booking`, `correct_stock`, `move_stock`,
  `raise_exception`, `close_exception`) are two-step. The first call has no
  `confirmation_token`, changes nothing and returns `CONFIRMATION_REQUIRED` with a preview and a
  token. A second call with the same arguments and the token makes the change.
- **Errors** always have `code`, `message`, `field`, `details` and `suggested_action`, and an
  `example_call` where one can be built.
