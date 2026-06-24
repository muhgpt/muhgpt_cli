# MuhGPT

A modular, human-in-the-loop CLI assistant for **authorized** penetration testing
and OSINT. It talks to your OpenAI-compatible `muh-chat` endpoint, lets the model
drive predefined local tools via **native function calling**, requires explicit
operator approval before any command runs, and exports a clean Markdown report.

## Project structure

```text
muhgpt_cli/
├── main.py                  # CLI loop, banner, authorization gate, report export
├── pyproject.toml           # packaging + `muhgpt` console entry point + pytest config
├── requirements.txt
├── .env.example
├── .gitignore
├── README.md
├── muhgpt/
│   ├── __init__.py
│   ├── config.py            # .env loading -> immutable Settings
│   ├── api_client.py        # resilient HTTP client (retries, backoff, error types)
│   ├── tools.py             # tool schemas + dispatcher + human-in-the-loop
│   ├── guard.py             # autonomous-mode safety classifier + run budget
│   ├── mcp.py               # Model Context Protocol client (stdio + HTTP), pure stdlib
│   ├── arsenal.py           # recon tool catalog + attack-chain playbooks
│   ├── packages.py          # package-manager detection + install commands
│   ├── session.py           # JSONL audit log + Markdown report
│   ├── render.py            # terminal Markdown renderer
│   ├── bidi.py              # display-only RTL (Arabic) fix
│   ├── ui.py                # ANSI color theme
│   └── agent.py             # multi-step tool-use feedback loop
└── tests/                   # pytest suite (no network — model + HTTP are faked)
```

## Setup

```bash
cd muhgpt_cli
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# edit .env and set MUHGPT_API_KEY
```

`.env` format:

```ini
MUHGPT_API_KEY=mghp_your_key_here
MUHGPT_BASE_URL=https://api.muhgpt.com/v1
MUHGPT_MODEL=muh-chat
```

The API key is read from the environment only; nothing is hardcoded. See
`.env.example` for the full set of optional tuning variables (timeouts, retries,
`MUHGPT_MAX_HISTORY_MESSAGES`, and `MUHGPT_TEMPERATURE=none` to omit the field
for reasoning models that reject it).

Optionally install it as the `muhgpt` command:

```bash
pip install -e .
muhgpt --version
```

## Step-by-step quick start

From zero to a finished recon report. Use only against systems you are
**authorized** to test (the examples use `scanme.nmap.org`, which Nmap provides
for exactly this).

### 1. Install and configure

```bash
git clone <repo> && cd muhgpt_cli
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # then edit .env and set MUHGPT_API_KEY
```

### 2. Start a session (manual mode — the safe default)

```bash
python main.py
```

You get a `[you@muhgpt] ❯` prompt. Type `/help` any time. **Every** command the
model wants to run shows up in a box and waits for your `[y/N]` — nothing executes
without your approval.

### 3. Run your first recon

Type a **skill** followed by an authorized target:

```text
/recon scanme.nmap.org
```

The agent plans and proposes commands (whois → dig → nmap → httpx …); press `y`
to run each. Findings are saved as it goes. Single-purpose skills: `/subdomains`,
`/dns`, `/tls`, `/ports`, `/web`. Bigger attack-chain playbooks: `/pentest`,
`/osint`, `/cloud`, `/api`, `/vulns`.

> Declare the target as the scope so the agent knows it's authorized:
> `python main.py --scope scanme.nmap.org`. The agent will refuse hosts that are
> clearly outside the confirmed scope.

### 4. Use the vulnerability playbooks (skills)

MuhGPT ships a knowledge base of per-class playbooks (find → **validate** →
report). List and preview them:

```text
/skills              # xss, sqli, ssrf, idor, ssti, xxe, rce, auth-jwt, …
/skills sqli         # preview the SQL-injection playbook
```

During a run the agent loads the right one itself (the `load_skill` tool) before
hunting a bug class — that's where its techniques, payloads, and the
"no PoC, no finding" validation method come from. Add your own by dropping a
Markdown file into `muhgpt/skills/`.

### 5. Choose how deep to go (scan modes)

```bash
python main.py --scan-mode quick      # fast, breadth, high-impact only
python main.py --scan-mode standard   # balanced (default)
python main.py --scan-mode deep       # exhaustive + vulnerability chaining
```

When the agent confirms a real, **proven** vulnerability it files it with
`report_vulnerability` — which requires a proof of concept and auto-computes a
**CVSS 3.1** score. It also keeps a scratchpad (`note` / `recall_notes`). All of
this lands in the exported report (a severity-sorted *Vulnerabilities* section,
plus *Notes & Methodology*).

### 6. Ask it anything (no target needed)

Assistant skills run a one-off in a dedicated role:

```text
/explain how TLS session resumption works
/code a python script that parses nmap -oX output
/security  <paste a code snippet to review>
```

### 7. Add web search + OSINT (free MCP servers, no API key)

Requires Node/`npx`. Just add `--mcp` — a curated free set loads automatically
(DuckDuckGo search, fetch, Wikipedia, sequential-thinking):

```bash
python main.py --mcp
```

Check what's connected with `/mcp`, then let the model search the web:

```text
search the web for recent nginx CVEs and summarize them
```

Make it always-on by adding `MUHGPT_MCP_ENABLED=1` to `.env` (then plain
`python main.py` already has MCP).

### 8. Plug in your own MCP servers (optional)

Create `mcp.json` (the standard `mcpServers` shape) and keep API keys in `.env`,
not in the file:

```json
{ "mcpServers": { "shodan": { "command": "npx", "args": ["-y", "shodan-mcp"] } } }
```

```bash
python main.py --mcp --mcp-config mcp.json
```

Your servers are **merged on top** of the bundled free ones. Use
`--no-mcp-defaults` to load only yours.

### 9. Go hands-off (autonomous mode)

Give one objective and let it run read-only recon end-to-end without approving
each step (you acknowledge the scope once at launch; destructive commands stay
blocked, installs still ask):

```bash
python main.py --auto --scope scanme.nmap.org
```

### 10. Maximum hands-off (YOLO)

Auto-approves everything **except** the destructive denylist and secret-file
reads. Only against targets you fully trust:

```bash
python main.py --yolo --scope your-lab.example.com
```

### 11. One-shot (for cron / CI)

Run a single objective, export the report, and exit — no prompts:

```bash
python main.py --auto --objective "/recon scanme.nmap.org" --scope scanme.nmap.org
```

### 12. Get your report

Inside a session type `/report` (and you're offered an export on exit). Reports
are written to `reports/report-*.md`, with a full JSONL audit log next to them.

## Run

```bash
python main.py
```

The session starts immediately. In-session commands: `/help`, `/install <pkg>`,
`/mcp`, `/skills`, `/scope`, `/report`, `/exit`. The operator handle defaults to your login name and
the report scope label defaults to `unrestricted`; override either with
`--operator NAME` / `--scope "target(s)"`. Run `python main.py --version` to
print the version.

**Recon skills.** Built-in `/<skill> <target>` commands expand into a ready-made
recon playbook and run it through the agent (hands-off under `--auto`,
step-approved otherwise): `/recon`, `/subdomains`, `/dns`, `/tls`, `/ports`,
`/web`. For example `/recon example.com` runs WHOIS → DNS → subdomains → TLS →
nmap → HTTP fingerprinting and saves findings to the report.

**Attack-chain playbooks.** Larger HexStrike-style multi-tool objectives that
chain the recon arsenal end-to-end (also `/<skill> <target>`): `/pentest` (full
attack-surface chain), `/osint` (passive profile, no active scan), `/cloud`
(internet-facing cloud footprint), `/api` (map + probe an API), `/vulns`
(non-destructive nuclei/nikto vulnerability scan). They run through the same
guard as everything else — nothing in a playbook can bypass approval or the
denylist.

**Assistant skills.** General-purpose `/<skill> <your text>` commands run your
request under a dedicated role — its own system prompt, isolated from the pentest
context — so the model answers in-character: `/code`, `/analyze`, `/debug`,
`/explain`, `/write`, `/optimize`, `/security`. For example `/security review this
login handler …` runs a focused security review. `/help` lists everything.

Output is colorized (gradient banner, highlighted command-approval box, dimmed
model reasoning). Colors auto-disable when output isn't a terminal, and can be
turned off with `--no-color` or the `NO_COLOR` / `MUHGPT_NO_COLOR` env vars —
audit logs and Markdown reports are always written plain.

Replies **stream token-by-token** as the model produces them. On a terminal,
once a reply finishes it is re-rendered in place as Markdown: pipe tables drawn
with box borders and aligned columns, plus headings, bullet/numbered lists,
blockquotes, fenced code blocks, and inline bold/italic/code. (If the reply is
taller than the screen, or output is piped, the raw stream is kept as-is.) Use
`--no-stream` (or `MUHGPT_STREAM=false`) to buffer the whole reply and render it
in one shot. Rendering is display-only — raw Markdown is what gets logged and
saved to reports.

**Right-to-left (Arabic) replies.** Most terminals lay text out left-to-right and
don't implement the Unicode BiDi algorithm or Arabic shaping, so an Arabic reply
shows up with its letters mirrored and disconnected. MuhGPT fixes this at the
display layer: lines containing RTL text are reshaped (contextual Arabic forms +
lam-alef ligatures) and reordered so they read correctly even on a non-BiDi
terminal. It's pure-stdlib and **display-only** — audit logs and saved reports
stay in logical order. Controlled by `MUHGPT_BIDI` (`auto` default / `on` / `off`);
set it to `off` if your terminal already does BiDi, to avoid double-reversal.

When the API reports token usage, a dim `↑prompt ↓completion · session total`
line is printed after each turn, and a **Token Usage** section is appended to
the exported report. Set `MUHGPT_PRICE_PROMPT_PER_1M` / `MUHGPT_PRICE_COMPLETION_PER_1M`
(USD per 1M tokens) to also show an estimated `~$cost` per turn and per session.

## How it works

- **Native tool calling.** Instead of parsing ```` ```bash ```` blocks out of prose,
  the model is given four tools — `execute_terminal_command`, `install_package`,
  `read_file`, `save_report` — and invokes them through the standard `tool_calls`
  interface. The agent feeds each tool's output back as a `tool` message, so the
  model can interpret results and pick the next step. That round-trip is the
  feedback loop; it is capped per turn by `MUHGPT_MAX_TOOL_ROUNDS`. Any reasoning
  the model narrates alongside its tool calls is printed before each approval
  prompt, so you see *why* a command is proposed.
- **Installs missing tools.** When a command fails because its tool isn't
  installed (exit 127 / "command not found"), the runtime identifies the missing
  binary, offers to install it via the detected package manager — `brew`,
  `apt-get`, `pkg` (Termux), `dnf`, `yum`, `pacman`, `apk`, or `zypper` (choosing
  the right command, with `sudo` only when needed) — and, on approval, installs
  it and **re-runs the original command automatically**. This recovery is
  deterministic, so it works even with models that are weak at tool-calling. The
  model can also call `install_package` directly. Either way the install is shown
  and approved like any other command before it runs.
- **Operator-driven installs.** You can also install tools yourself without the
  model: `/install nmap` (or `/install nmap masscan`), or simply typing a bare
  request like `install nmap` / `instala o nmap`. These route straight to the
  package manager through the same `[y/N]` approval — bypassing the model so a
  weak tool-caller can't refuse. Package names are validated against a strict
  allowlist before they reach the shell.
- **Bounded context.** The running conversation is trimmed to the most recent
  `MUHGPT_MAX_HISTORY_MESSAGES` (default 40, `0` to disable) so long engagements
  don't silently blow the model's context window or cost. Trimming only cuts on
  turn boundaries, never splitting a tool call from its result.
- **Human-in-the-loop (default).** Both command execution and file reads route
  through a confirmation prompt and only proceed on an explicit `y`. The model
  cannot cause a side effect on its own.
- **Autonomous mode (opt-in, `--auto`).** Give one objective and the agent plans,
  runs read-only recon, installs missing tools, maps the target, and writes the
  report end-to-end without approving each step. A safety guard classifies every
  command at the execution boundary — **independently of the model**, so it holds
  even if scanned output prompt-injects it:
  - **BLOCK** — destructive/irreversible/weaponized commands (`rm -rf`, `dd`,
    `mkfs`, fork bombs, `curl … | sh`, disk/`/etc` writes, `~/.ssh` writes, exfil,
    reverse shells, brute-force/exploit tooling, `sudo`, …) never run and aren't
    even offered.
  - **auto-run** — only a curated allowlist of single-purpose read-only recon
    tools (nmap, whois, dig, httpx, whatweb, subfinder, amass, sslscan, nikto,
    plus the expanded arsenal: dnsx, naabu, tlsx, asnmap, cdncheck, katana,
    nuclei, …) with no shell metacharacters.
  - **CONFIRM** — everything else still stops for your `[y/N]`: unknown binaries,
    any pipe/chaining, installs, local file reads, the general-purpose tools
    `curl`/`wget`/`openssl` (whose flags can read/write/exfil files, so they're
    never auto-run), and any command whose target host looks **outside the
    declared scope** (a soft guard against injected scope pivots).

  It's bounded by a budget (rounds / commands / installs / wall-clock; see
  `.env.example`) plus a **no-progress guard** that halts the run if the model
  goes several rounds without a command actually executing (stuck talking, or all
  blocked/declined). Abortable with Ctrl-C, fully audit-logged, and requires a
  one-time scope acknowledgement at launch. The denylist has no disable flag — to
  run a blocked command, drop back to manual mode. Enable with `--auto` or
  `MUHGPT_AUTO=1`. Default stays human-in-the-loop.
- **YOLO mode (opt-in, `--yolo`).** Maximum hands-off: in autonomous mode it
  auto-approves the **CONFIRM tier too** — curl/wget/openssl, pipes/chaining,
  installs, and reads of non-secret files all run unattended, with **no per-step
  prompts**. Two lines stay absolute even here: the **BLOCK denylist** (rm -rf,
  dd, exfil, reverse shells, `sudo`, …) never runs, and **secret/credential file
  reads** (`~/.ssh`, `.env`, `*.pem`, …) still require a prompt. Still bounded by
  the same budget. This trades away the CONFIRM safety layer, so scanned output
  could prompt-inject the model into running a CONFIRM-tier command — use it only
  against targets you fully trust (your own lab/infra), never untrusted hosts.
  Enable with `--yolo` (implies `--auto`) or `MUHGPT_AUTO_YOLO=1`.
- **MCP client (opt-in, `--mcp`).** Connect to external [Model Context
  Protocol](https://modelcontextprotocol.io) servers and let the model call their
  tools alongside the built-ins. Both **stdio** (local subprocess) and **HTTP**
  servers are supported — implemented in pure stdlib + `requests`, no SDK.
  Discovered tools are namespaced `mcp__<server>__<tool>` and routed through the
  **same guard** as shell commands: in `--auto` every MCP call defaults to a human
  `[y/N]` (never auto-runs), tool names that look weaponized
  (exploit/shell/payload/brute-force) are **BLOCKED**, out-of-scope target
  arguments downgrade to a confirm, and only tools you list in
  `MUHGPT_MCP_AUTO_TOOLS` may auto-run. Tool descriptions and outputs are treated
  as untrusted input. `/mcp` lists connected servers and tools. Off by default.
  - **Batteries included.** When you enable MCP, a curated set of **free,
    no-API-key** servers loads automatically (needs Node/`npx`): **ddg**
    (DuckDuckGo web search — deep search / OSINT), **fetch** (URL → html/markdown/
    txt/json — recon), **wikipedia** (search + read), and **think** (sequential
    reasoning). So `python main.py --mcp` works out of the box with no config.
    Versions are pinned; disable the bundle with `--no-mcp-defaults` /
    `MUHGPT_MCP_DEFAULTS=0`.
  - **Add your own.** Point `--mcp-config mcp.json` (or `MUHGPT_MCP_CONFIG`) at a
    standard `{"mcpServers": {…}}` file; your servers are **merged on top** of the
    bundled ones (a same-named entry overrides the default). Great for keyed
    OSINT/recon servers (Shodan, VirusTotal, Brave, a pinned nmap/nuclei wrapper,
    …) — put the API keys in `.env`, not in the JSON. MuhGPT never auto-installs a
    server; review and pin them yourself.
- **One-shot / scripting (`--objective`).** Run a single objective and exit,
  exporting the report automatically: `python main.py --auto --objective "/recon
  example.com"`. Non-interactive (no prompts; `--auto` is the consent) — for cron
  or CI.
- **Auditing + reporting.** Every message, proposed command (approved or not), and
  finding is appended to `reports/session-*.jsonl` as it happens. `save_report`
  builds the human-facing report, exported to `reports/report-*.md`.
- **Vulnerability playbooks (skills).** A bundled knowledge base of per-class
  playbooks (XSS, SQLi, SSRF, IDOR, SSTI, XXE, RCE, JWT/auth, path-traversal,
  CSRF, open-redirect, NoSQLi) — each with where-to-look, detection, **validation
  ("no PoC, no finding")**, payloads, and remediation. The model loads one on
  demand via the `load_skill` tool; browse them with `/skills` (or `/skills xss`
  to preview). The available names are injected into the system prompt so even a
  weak model knows what it can pull in. Pure prompt data — grants no new execution
  power. (Inspired by [Strix](https://github.com/usestrix/strix)'s skills.)
- **Scratchpad memory.** `note` / `recall_notes` give the agent durable
  engagement memory (leads, plan, in-scope creds) that survives history trimming —
  rendered in the report under *Notes & Methodology*.
- **Validated reporting + CVSS.** `report_vulnerability` files a structured
  finding that **requires a proof of concept** and computes a real **CVSS 3.1**
  base score/severity/vector (pure stdlib, no dependency), de-duplicating by
  title. Vulnerabilities render in their own severity-sorted report section.
- **Scan modes.** `--scan-mode quick|standard|deep` (or `MUHGPT_SCAN_MODE`) shapes
  the agent's depth — `quick` (breadth, fast), `standard` (balanced), `deep`
  (exhaustive + vulnerability chaining).

### A note on `shell=True`

`execute_terminal_command` runs through the shell so real recon one-liners (pipes,
redirects, globs) work. The safety boundary is the mandatory approval prompt: you
see the exact command string before it can run. Review each command before
approving — only run this against systems you are authorized to test.

## Tests

The suite fakes the model and the HTTP layer, so it runs offline and fast:

```bash
pip install -e ".[dev]"   # or: pip install pytest ruff
python -m pytest
ruff check .              # lint (config in pyproject.toml)
```

## Termux / Android

Works under Termux. If `python` is missing: `pkg install python`. The optional
`MUHGPT_COMMAND_TIMEOUT` is handy on mobile to stop long scans. Everything else is
pure-Python with two small dependencies.

## License

Released under the [MIT License](LICENSE). For **authorized** security testing
only — you are responsible for having permission to test any target.
