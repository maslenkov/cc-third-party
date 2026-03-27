# third-party — Claude Code Plugin

Two skills for safely bringing external plugins and skill sets into Claude Code from local sources — no remote marketplaces, no automatic downloads.

## Usage

```
/third-party:security-audit /path/to/downloaded/repo   # audit before installing
/third-party:install /path/to/downloaded/repo          # then install
```

## Setup from local repo

**Prerequisites:** Claude Code CLI installed and working.

### 1. Clone the repo

```sh
git clone <repo-url>
```

### 2. Register as a marketplace

In a Claude Code session, run:

```
/plugin marketplace add /path/to/third-party-plugin
```

### 3. Install the plugin

```
/plugin install third-party@third-party
```

### 4. Verify

Type `/third-party:` — you should see `install` and `security-audit` in the autocomplete.

> **Note:** If you skip `/plugin install` and manually edit `settings.json` instead, the skills load but tab-completion will not work. Always use `/plugin install`.

---

## Security: use a sandbox working directory

When auditing or installing third-party content, **open Claude Code from a neutral, empty folder** — not from inside the cloned repo itself. Claude Code inherits tool permissions relative to its working directory, so launching it from a downloaded repo means you may end up granting broad access to untrusted files.

A clean setup looks like this:

```
~/projects/
├── claude-sandbox/        ← open Claude Code here for auditing and installs
└── opensource/
    └── some-plugin/       ← cloned repo lives here, passed as an argument
```

Open a Claude Code session in `~/projects/claude-sandbox/`, then run:

```
/third-party:security-audit ~/projects/opensource/some-plugin
/third-party:install ~/projects/opensource/some-plugin
```

The untrusted repo is only ever referenced by path — Claude never runs from within it.

---

## Skills

### `/third-party:install`

Installs a plugin, skill set, or command set from a local directory.

Supports two installation flows depending on what it finds in the source folder:

**Flow A — Full Plugin** (folder contains `.claude-plugin/plugin.json`)

1. Reads the plugin manifest and `marketplace.json`
2. Guides you through registering the local folder as a Claude Code marketplace
3. Installs the plugin via `/plugin install`
4. Verifies tab-completion works

**Flow B — Skills / Commands / Agents only** (folder contains `skills/`, `commands/`, or `agents/` directories, no plugin manifest)

1. Lists what will be copied and to where
2. Copies each item to `~/.claude/skills/`, `~/.claude/commands/`, or `~/.claude/agents/`
3. Detects conflicts and offers backup before overwriting
4. Verifies the installed skill can be invoked

Key safety rules:
- Never writes a file without showing the plan and asking for confirmation first
- `yes-all` shortcut available to approve remaining items in bulk
- Handles missing `marketplace.json` by offering to create it from `plugin.json`

---

### `/third-party:security-audit`

Audits a folder of AI agent skills, plugins, or modules for security threats before you install anything.

Checks every file for six threat categories:

| Category | What it looks for |
|---|---|
| **Prompt Injection** | Instructions that try to override the agent's behavior or persona |
| **Excessive Permissions** | Overly broad `Bash(*)`, `Write(*)`, or access to sensitive paths in `allowed-tools` |
| **Data Exfiltration** | `curl`, `fetch`, webhooks, URLs in instruction blocks pointing outside the local machine |
| **Jailbreak** | Attempts to disable safety restrictions ("DAN", "no restrictions", fake authority claims) |
| **Hardcoded Secrets** | API keys (`sk-...`), `Bearer` tokens, `password=`, `api_key=` patterns |
| **System Hijacking** | Instructions to modify `CLAUDE.md`, `settings.json`, other skills, or read `~/.ssh` |

Findings are rated **Critical / Suspicious / Note** and explained in plain language. The audit runs in an isolated subagent that uses only read-only tools (Glob, Grep, Read) — it cannot execute code, make network calls, or write files.
