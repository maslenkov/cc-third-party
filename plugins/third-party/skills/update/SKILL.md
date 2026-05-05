---
name: update
description: Use when the user wants to update a specific locally-installed Claude Code plugin from its source directory. Invoked as /third-party:update <plugin-name>
---

# Update Local Plugin

Plugin to update: **`$ARGUMENTS`**

---

## Step 0 — Validate input

If `$ARGUMENTS` is empty — stop and ask:
> "Укажи имя плагина, например: `/third-party:update claude-hud`"

---

## Step 1 — Find the plugin source

Read `~/.claude/plugins/known_marketplaces.json`.

Determine lookup mode from `$ARGUMENTS`:
- If it starts with `/` or `~` — match by `path` value in the entry (**path mode**)
- Otherwise — match by the entry key (**name mode**)

Find the matching entry with `"source": "directory"`.

If not found in `known_marketplaces.json` — check `~/.claude/third-party/registry.json` for a matching key. If found there, use it (fields: `path`, `build`, `install`).

If not found anywhere — go to **Step 1b**.

Extract:
- `plugin_name` — the entry key
- `plugin_path` — the `path` value
- `build_cmd` — the `build` value (if present)
- `install_cmd` — the `install` value (if present)

---

## Step 1b — Investigate unknown plugin

Launch a subagent (Bash) to gather facts:
1. `which <plugin_name>` → binary path
2. If found: `ls -la <binary_path>` — symlink? real file?
3. Look for source in common locations: `~/projects/`, `~/src/`, `~/code/`
4. In the found source dir: **read `BUILD.md` or `INSTALL.md` first** — these usually contain exact build and install commands
5. Also check for: `flake.nix`, `package.json`, `Makefile`, `Cargo.toml`
6. Check available build tools: `which bun`, `which nix`, `which npm`, `which cargo`
7. Return all findings

Based on findings, ask the user one focused question at a time to fill gaps:
- Source dir not found → ask where it is
- Build tool found in source but not in PATH → "Вижу `flake.nix`, но `nix` не нашёл в системе — чем собирали?"
- Install method unclear → "Как устанавливали бинарник после сборки?"

Once source + build + install are known, ask:
> "Сохранить эту конфигурацию в `~/.claude/third-party/registry.json` для будущих запусков? (yes/no)"

If yes — write the entry and continue. Then proceed with `plugin_name`, `plugin_path`, `build_cmd`, `install_cmd`.

---

## Step 2 — Fetch and diff

Launch a subagent (Bash access). The subagent must:

1. Run `git -C <plugin_path> fetch`
2. Run `git -C <plugin_path> rev-parse HEAD` → `current_sha`
3. Run `git -C <plugin_path> rev-parse FETCH_HEAD` → `fetch_sha`
4. If `current_sha == fetch_sha` — return "already up to date"
5. Otherwise:
   - Run `git -C <plugin_path> log HEAD..FETCH_HEAD --oneline` → commit summary
   - Run `git -C <plugin_path> diff HEAD..FETCH_HEAD` → save to `~/.claude/third-party/update-<plugin_name>.patch`
   - Run `git -C <plugin_path> diff --name-only HEAD..FETCH_HEAD` → list of changed files
   - Return: `current_sha`, `fetch_sha`, commit summary, patch path, changed file list

If already up to date — tell the user and stop.

---

## Step 3 — Security audit on the diff

Show the user the commit summary and list of changed files.

Determine which audits to run based on the changed file list:

- **Has `.md`, `.yaml`, `.json` skill/config files** → invoke `/third-party:security-audit ~/.claude/third-party/update-<plugin_name>.patch`
- **Has `.ts`, `.tsx`, `.js`, `.mjs`, `.py`, `.sh`, `.bash`, `.rb`, `.go`, `.rs` files** → invoke `/third-party:security-audit-code ~/.claude/third-party/update-<plugin_name>.patch`
- Both types present → run both

Both skills accept a `.patch` file and will check only the added lines.

**Important:** invoke these skills as foreground subagents (not background). Background subagents cannot request file read permissions from the user and will silently fail.

If any **Critical** findings from either audit — stop and do not proceed with the update.

---

## Step 4 — Confirm and apply

Ask:
> "`<plugin_name>`: `<current_sha_short>` → `<fetch_sha_short>`. Применить обновление? (yes/no)"

**Do not proceed without explicit "yes".**

Launch a subagent: `git -C <plugin_path> merge --ff-only FETCH_HEAD`

If merge fails (not fast-forward) — report the error and stop. Do not force-merge.

---

## Step 5 — Build (if needed)

Read `<plugin_path>/package.json` (if it exists).

If it has a `scripts.build` field — ask:
> "Найден package.json со скриптом build. Запустить сборку? (yes/no)"

If yes — launch a subagent: `npm run build` in `<plugin_path>`. Show output. Stop if build fails.

---

## Step 6 — Reinstall plugin

Ask the user to run in Claude Code:
```
/plugin update <plugin_name>@<plugin_name>
```

This opens the plugin TUI — the user selects the plugin and confirms the update.

Wait for confirmation that the command completed successfully.

---

## Step 7 — Done

Report:
> "`<plugin_name>` обновлён: `<current_sha_short>` → `<fetch_sha_short>`"

Clean up: delete `~/.claude/third-party/update-<plugin_name>.patch`.
