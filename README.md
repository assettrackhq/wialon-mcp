# Wialon MCP server and agent skill

Ask your Wialon fleet questions in plain language from the AI agent you already
use — Claude Code, Codex, Cursor, VS Code, Claude Desktop — and get answers
from live Wialon data: where a vehicle is, how far the fleet drove last week,
who is low on fuel, how much was filled and where fuel may have been drained,
who drives worst, who idles most, who was at the depot and for how long, which
truck is closest to an address, what changed since yesterday, which services
are due, a saved Wialon report as a PDF.

Two pieces make that work. The server is hosted; this repository holds the
skill and everything that connects a client to the server:

- **The MCP server** — `https://ai.theassettrack.com/mcp`. Nineteen read-only
  tools over your Wialon account. You sign in with your Wialon login, on
  Wialon's own page. Nothing to install: it is an address your client
  connects to.
- **The skill** — [`skills/wialon`](skills/wialon/SKILL.md). Which tool answers
  which question, and the rules that decide whether the answer is right: a
  position from April is not a moving truck, an unreadable fuel sensor is not an
  empty tank, "yesterday" is the account's yesterday.

The server also hands the skill to any client as its instructions when it
connects, so the tools work without it. Install the skill as well when your
agent loads skills on demand — it is what the agent reads before it answers,
and it keeps the rules in view across a long session.

## How signing in works

The first time your client connects, it opens a page in your browser. You
pick the Wialon server your account lives on — hosting.wialon.com, .eu or .us —
and sign in on Wialon's own login page; FleetAI never sees your password.
Wialon then issues a **view-only** access token for 30 days, which appears in
your Wialon user settings under Access tokens as "FleetAI MCP". Delete it
there and the access ends.

## Claude Code — one plugin, both pieces

```bash
claude plugin marketplace add assettrackhq/wialon-mcp
```

```bash
claude plugin install wialon@wialon-mcp
```

Then run `/mcp`, choose `fleetai` and **Authenticate**.

Only the server, without the skill:

```bash
claude mcp add --transport http fleetai https://ai.theassettrack.com/mcp
```

## Claude, ChatGPT and other agents

**1. Connect the server.**

| Client                    | Where                                                                      |
| ------------------------- | -------------------------------------------------------------------------- |
| claude.ai, Claude Desktop | Settings → Connectors → Add custom connector, URL above                    |
| ChatGPT                   | Developer mode → add an MCP server with the URL above, OAuth               |
| Codex                     | [`clients/codex.toml`](clients/codex.toml), then `codex mcp login fleetai` |
| Cursor                    | [`clients/cursor.json`](clients/cursor.json) in `.cursor/mcp.json`         |
| VS Code                   | [`clients/vscode.json`](clients/vscode.json) in `.vscode/mcp.json`         |
| Older desktop clients     | [`clients/claude-desktop.json`](clients/claude-desktop.json) (Node.js)     |

Most clients read their MCP config only at startup: restart after saving.

**A token instead of signing in.** For a script or a client that cannot open
a browser, issue an access token in your Wialon user settings and send it as
`Authorization: Bearer <token>`. An account not on hosting.wialon.com adds
`?region=eu` or `?region=us` to the URL:

```bash
claude mcp add --transport http fleetai https://ai.theassettrack.com/mcp \
  --header "Authorization: Bearer <your token>"
```

**2. Add the skill.** With the open skills installer, which knows where each
agent looks:

```bash
npx skills add assettrackhq/wialon-mcp
```

Or copy [`skills/wialon`](skills/wialon) by hand:

| Agent         | Where it goes                                                 |
| ------------- | ------------------------------------------------------------- |
| Claude Code   | `~/.claude/skills/wialon/`, or `.claude/skills/` in a project |
| Codex, Cursor | `~/.agents/skills/wialon/`, or `.agents/skills/` in a project |
| Anything else | Paste `SKILL.md` and `reference.md` into its instructions     |

## Try it

- "How many vehicles are online, and which have not reported today?"
- "How far did each truck drive last week?"
- "Show me yesterday's route for ENJ407."
- "Did anyone lose fuel this week that was not a fill?"
- "Who drove worst last month, and what for?"
- "Which vehicles idled the most yesterday, and how much fuel did it burn?"
- "Who was at the depot today, and for how long?"
- "Which truck is closest to Storgatan 1, Örebro?"
- "What changed since yesterday?"
- "Which services are overdue?"
- "Export last month's mileage report as a PDF."

In a client that draws MCP Apps — Claude and ChatGPT among them — a question
about where vehicles are, the nearest one to a place, or a route comes back
with a map in the chat as well as the answer. A position more than an hour old
is drawn grey, so it never reads as "here now". Other clients get the same
answer as text.

## What the server does with your access

- **Read-only.** Every tool reads. Nothing creates, edits or deletes anything
  in your Wialon account, and a token from signing in is view-only on Wialon's
  side too.
- **Never kept.** What your client holds is the Wialon token, encrypted; it
  travels with each request. The server uses it to sign in to Wialon, keeps
  that session in memory for a few idle minutes so a run of questions does not
  sign in again each time, and sends both only to Wialon's own hosts. No
  database, and no log line with the token or the session in it.
- **Browsers refused.** Requests carrying a browser `Origin` are rejected, so a
  web page cannot use your token through it.

## When something is off

| Symptom                                     | Check                                                                                   |
| ------------------------------------------- | --------------------------------------------------------------------------------------- |
| The agent says it has no fleet tools        | The client has not connected. Is the config in the file it reads? Restart it.           |
| The client asks you to sign in again        | The 30 days are up, or "FleetAI MCP" was deleted in Wialon. Sign in again.              |
| Every call fails at once with an auth error | With a token of your own: the region (`?region=eu` or `us`), then the `Bearer ` prefix. |
| "The connection to Wialon dropped"          | The token has expired or been deleted. Retrying the same call will not help.            |
| An export came back with no file            | The interval held no rows. Widen the dates, or check the unit reported at all.          |

Full setup notes: <https://aichat.theassettrack.com/mcp>

## License

The skill and the configs here are MIT — see [LICENSE](LICENSE). The server
itself is a hosted service, not part of this repository.
